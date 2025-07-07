### Prepare variables
Note: "vault-ha-tls" and "vault-internal"  are really recomended if you use helm
export VAULT_SERVICE_NAME="vault-internal" \
export VAULT_K8S_NAMESPACE="vault" \
export VAULT_HELM_RELEASE_NAME="vault" \
export SECRET_NAME="vault-ha-tls" \
export K8S_CLUSTER_NAME="cluster.local" \
export CSR_NAME="vault-csr"

generate a keypair
  openssl genrsa -out vault.key 2048
### csr config
cat > vault-csr.conf <<EOF
[req]
default_bits = 2048
prompt = no
encrypt_key = yes
default_md = sha256
req_extensions = v3_req
distinguished_name = kubelet_serving
[ kubelet_serving ]
O = system:nodes
CN = system:node:*.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = *.${VAULT_SERVICE_NAME}
DNS.2 = *.${VAULT_SERVICE_NAME}.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
DNS.3 = *.${VAULT_K8S_NAMESPACE}
DNS.4 = *.cluster.com
DNS.5 = *.vault.svc
IP.1 = 127.0.0.1
IP.2 = 192.168.1.100
EOF
###generate the csr
openssl req -new -key vault.key -out vault.csr -config vault-csr.conf


###We wrap the vault.csr to a yaml file and send the csr to k8s-api.
cat > csr.yaml <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
   name: vault.svc
spec:
   signerName: kubernetes.io/kubelet-serving
   expirationSeconds: 31536000
   request: $(cat vault.csr|base64|tr -d '\n')
   usages:
   - digital signature
   - key encipherment
   - server auth
EOF

kubectl create -f csr.yaml
### approve the csr
kubectl certificate approve vault.svc
  Note: this approval does not work if in the usege we also have "client auth"
kubectl get csr vault.svc
    ### CONDITION: Approved,Issued
### get the crt
serverCert=$(kubectl get csr vault.svc -o jsonpath='{.status.certificate}')
echo "${serverCert}" | openssl base64 -d -A -out vault.crt

### Retrieve Kubernetes CA certificate.
kubectl get secret \
  -o jsonpath="{.items[?(@.type==\"kubernetes.io/service-account-token\")].data['ca\.crt']}" \
  | base64 --decode > vault.ca

###Create the TLS secret.
kubectl create secret generic ${SECRET_NAME} \
   -n $VAULT_K8S_NAMESPACE \
   --from-file=vault.key=vault.key \
   --from-file=vault.crt=vault.crt \
   --from-file=vault.ca=vault.ca

###install Vault CLI (1.19.5-1)
Vault TLS key and self-signed certificate have been generated in '/opt/vault/tls'.

helm install -n $VAULT_K8S_NAMESPACE $VAULT_HELM_RELEASE_NAME hashicorp/vault -f overrides.yaml

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator init \
    -key-shares=5 \
    -key-threshold=2 \
    -format=json > cluster-keys.json


export VAULT_UNSEAL_KEY_1=<your key>
export VAULT_UNSEAL_KEY_2=<your key>
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_1
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY_2

###Join vault-1 pod to the Raft cluster
kubectl exec -n $VAULT_K8S_NAMESPACE -it vault-1 -- vault operator raft join -address=https://vault-1.vault-internal:8200 -leader-ca-cert="$(cat /vault/userconfig/vault-ha-tls/vault.ca)" -leader-client-cert="$(cat /vault/userconfig/vault-ha-tls/vault.crt)" -leader-client-key="$(cat /vault/userconfig/vault-ha-tls/vault.key)" https://vault-0.vault-internal:8200

Unseal vault-1.
kubectl exec -n $VAULT_K8S_NAMESPACE vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_1
kubectl exec -n $VAULT_K8S_NAMESPACE vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY_2

export CLUSTER_ROOT_TOKEN=<your root token>

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault login $CLUSTER_ROOT_TOKEN
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault operator raft list-peers
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault status
    Desired response:
    Sealed (should be false), Storage Type  (should be raft), HA Enabled (should be true), HA Cluster (should be your leader vault pod) 

###Ingress
create ingress secret using public and private key signed by the authority
  k create secret tls ingress-local-cert --cert=ingress-cluster-public.crt --key=ingress-cluster-com.key -n vault

For ServersTransport to use K8s CA:
  kubectl -n vault create secret generic vault-ca-secret   --from-file=ca.crt=vault.ca

k create -f ingressroute.yaml

Check the http response: 
curl -v https://vault.cluster.com:443/v1/sys/health

## Troubleshoot
###Create new root token
    vault operator generate-root -init
    Nonce         <your nonce>
    Started       true
    Progress      0/2
    Complete      false
    OTP           <your otp>
    OTP Length    28

echo ${VAULT_UNSEAL_KEY_1} | vault operator generate-root -nonce=<your nonce> -
echo ${VAULT_UNSEAL_KEY_2} | vault operator generate-root -nonce=<your nonce> -
result:
  Encoded Token    <your token>

vault operator generate-root -decode=<your token> -otp=<your otp>
    Voala!

Upgrade vault with helm:
`helm upgrade vault hashicorp/vault -f overrides.yaml -n vault`  
Make stateful set redeploy pods safely:
`kubectl patch statefulset vault -n $VAULT_K8S_NAMESPACE -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"restartedAt\":\"$(date +%s)\"}}}}}"`     

###Future notes: 
I think i should remake certificated for Vault, bacause i desire these alt names to be used:
    [alt_names]
    DNS.1 = *.${VAULT_SERVICE_NAME}
    DNS.2 = *.${VAULT_SERVICE_NAME}.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
    DNS.3 = *.${VAULT_K8S_NAMESPACE}
    DNS.4 = *.cluster.com
    DNS.5 = *.vault.svc
    DNS.6 = vault-active.${VAULT_K8S_NAMESPACE}.svc.${K8S_CLUSTER_NAME}
Because I enabled active service feature in values.yaml for vault helm chart. Thus i could use "spec.routes.services.name: vault-active" in IngressRoute and in ServersTransport use "spec.serverName: vault-active.vault.svc.cluster.local"
