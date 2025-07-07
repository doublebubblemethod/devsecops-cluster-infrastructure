#  Vault Agent Injector set up 
 I have configured:
 Vault Agent Injector validates the Vault server TLS certificate. This ensures secure communication between your Kubernetes pods and the Vault server, validating that you are connecting to the correct Vault server and preventing man-in-the-middle (MITM) attacks.

Prepare variables:
 
Go to Vault
    create Vault secret for application and mySQL db
    create Vault policy that can see secrets. 'data' path needs to be inserted when dealing with API level. Also you can check API path in vault ui where you have created a secret to get correct path for this policy. 
`kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -it -- sh`  
`vault write auth/kubernetes/role/agent-injector-role \
   bound_service_account_names=vaultagent-injector \
   bound_service_account_namespaces=application \
   policies=application-injector-policy \
   ttl=1h`  
`echo 'path "secrets/data/application/*" { capabilities=["read"] }' | vault policy write application-injector-policy -`
To do:
1. Check if V. Agent trusts Vault's CA since they are deployed by 1 helm chart
2. Inject the CA Certificate into the Vault Agent Injector Deployment
3. Continue "How to deploy Vault for Kubernetes in 2022 and inject secrets"
4. Inject secrets to mySQL DB and check if it works
5. Deploy PetClinic
