#  Vault Secrets Operator set up <span style="color: red;">(Not implemented yet)</span>

The Vault Secrets Operator syncs the secrets between Vault and the Kubernetes secrets in a specified namespace. Within that namespace, applications have access to the secrets. The secrets are still managed by Vault, but accessed through the standard way on Kubernetes. 

In this article, I will show how to:

    Install the Vault Secrets Operator (VSO)
    Configure the VSO to authenticate to an external Vault server using JWT auth mode
    Create Vault-level secret for key/value pairs, synchronize to Kubernetes-level secret
    Deploy a running container that does a rolling restart whenever Vault-level secrets are made
    Create a Vault-level secret for a TLS cert+key, synchronize to Kubernetes-level TLS secret

Prepare variables:
    ```vso_ns=<your namespace for vso>
    vaultIP=<yourVaultIPAddress>
    export VAULT_ADDR=http://$vaultIP:8200```

Install VSO using helm:
    ```
    helm repo add hashicorp https://helm.releases.hashicorp.com
    helm repo update hashicorp
    helm search repo hashicorp/vault-secrets-operator
    helm upgrade --install vault-secrets-operator hashicorp/vault-secrets-operator -f vso-values.yaml --namespace $vso_ns 
    ```
Go to Vault
create Vault secret for application and mySQL db
create Vault policy that can see secrets. 'data' path needs to be inserted when dealing with API level. Also you can check API path in vault ui where you have created a secret to get correct path for this policy. 


NEW 
The Database Secrets Engine is a Vault plugin that generates dynamic DB credentials instead of using static ones. These credentials have a lease and can be revoked or rotated automatically
In Vault, a role under the engine maps to a dynamic user template using SQL commands executed to create a temporary DB user and Grants ALL PRIVILEGES on the postgres database to it in role
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault secrets enable -path=db database

helm upgrade --install postgres bitna
mi/postgresql --namespace application --set audit.logConnections=true  --set auth.d
atabase=petclinic --set auth.username=petclinic --set auth.postgresPassword=<your password>

(it did not work for me and i had to create a user, grand roles and create DB:
CREATE DATABASE petclinic;
)


kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault write db/config/db \
   plugin_name=postgresql-database-plugin \
   allowed_roles="petuser" \
   connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.application.svc.cluster.local:5432/petclinic?sslmode=disable" \
   username="petclinic" \
   password="<your password>" 
Success! Data written to: db/config/db

Create a role for the PostgreSQL pod.

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault write db/roles/petuser \
   db_name=petclinic \
   creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
      GRANT ALL PRIVILEGES ON DATABASE petclinic TO \"{{name}}\";" \
   revocation_statements="REVOKE ALL ON DATABASE petclinic FROM  \"{{name}}\";" \
   backend=db \
   name=petuser \
   default_ttl="1h" \
   max_ttl="12h"

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault read db/creds/petuser

Create the auth-policy-db policy.

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- cat <<EOF > tmp/auth-policy-db.hcl
path "db/creds/petuser" {
   capabilities = ["read"]
}
EOF

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault policy write auth-policy-db /dev/stdin < tmp/auth-policy-db.hcl

### Helm upgrade with your values file
helm upgrade \
  -f vso-values.yml \
  vault-secrets-operator hashicorp/vault-secrets-operator \
  --namespace vault

kubectl logs deploy/vault-secrets-operator-controller-manager \
  -n vault | grep "clientCache"

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -it -- sh
vault secrets enable -path=demo-transit transit

Create a encryption key:
vault write -force demo-transit/keys/vso-client-cache

Create a policy for the operator role to access the encryption key:
cat <<EOF > tmp/demo-auth-policy-operator.hcl
path "demo-transit/encrypt/vso-client-cache" {
   capabilities = ["create", "update"]
}
path "demo-transit/decrypt/vso-client-cache" {
   capabilities = ["create", "update"]
}
EOF

vault policy write demo-auth-policy-operator dev/stdin < tmp/demo-auth-policy-operator.hcl

    Success! Uploaded policy: demo-auth-policy-operator

Create Kubernetes auth role for the operator: 
vault write auth/vso/role/auth-role-operator \
   bound_service_account_names=vault-secrets-operator-controller-manager \
   bound_service_account_namespaces=vault\
   token_ttl=0 \
   token_period=120 \
   token_policies=demo-auth-policy-operator \
   audience=vault

   Success! Data written to: auth/vso/role/auth-role-operator

Create a new role for the dynamic secret.
kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- vault write auth/vso/role/auth-role \
   bound_service_account_names=dynamic-application \
   bound_service_account_namespaces=application \
   token_ttl=0 \
   token_period=120 \
   token_policies=auth-policy-db \
   audience=vault

------------------
# Static secrets:
vault auth enable -path auth-mount kubernetes

vault write auth/auth-mount/config \
   kubernetes_host="https://10.96.0.1:443"

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- \
   vault write auth/auth-mount/role/pet-role \
   bound_service_account_names=static-secrets \
   bound_service_account_namespaces=application \
   policies=injector-policy \
   audience=vault \
   ttl=24h

k create -f application/vso-auth-static.yaml

kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -- \
   vault write auth/auth-mount/role/sonar-vso-role \
   bound_service_account_names=vso-secrets \
   bound_service_account_namespaces=sonar \
   policies=vso-sonar \
   audience=vault \
   ttl=24h

check if the secret was created; If so, refer to it in a deployment at spec.template.spec block
   ```env:
      - name: POSTGRES_USER
         valueFrom:
            secretKeyRef:
            name: secretkv
            key: POSTGRES_USER   ```
check application pod's logs and postres logs;

### Troubleshooting: 
My pod needs to be authorised to my docker repository; I created this K8S resource and added to spec.template.spec block of my petclinic deployment: 
   ```kubectl create secret docker-registry regcred \
      --docker-username=askdragon \
      --docker-password=<your pass> \
      --docker-email="<your email>" \
      -n application```

Postres logs error: 
`2025-07-14 21:57:23.817 GMT [7674] ERROR:  permission denied for schema public at character 28
2025-07-14 21:57:23.817 GMT [7674] STATEMENT:  CREATE TABLE IF NOT EXISTS vets ( id INT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY, first_name TEXT, last_name TEXT )`
exec your postres pod, log into as a user  (postres in my case is db admin user) and check your petclinic user permissions:
`k exec postgres-postgresql-0 -it -n application -- sh`  
`psql -U postgres` to login
`\l` list your tables in the db
`\c petclinic` to get into a specific db 
`\du` List of roles
`GRANT USAGE ON SCHEMA public TO petclinic;
GRANT ALL ON SCHEMA public TO petclinic;`

Another postres log that helped me to pinpoint out the issue:
`LOG:  connection authorized: user=petclinic database=petclinic`
I realized that the app's deployment POSTGRES_URL is not correct:
`incorrect value: jdbc:postgresql://postgres-postgresql.application.svc.cluster.local:5432/petclinic` 
`correct value: jdbc:postgresql://postgres-postgresql.application.svc.cluster.local:5432/postgres `