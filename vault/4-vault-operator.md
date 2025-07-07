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
    helm upgrade --install vault-secrets-operator hashicorp/vault-secrets-operator --namespace $vso_ns 
    ```
Go to Vault
create Vault secret for application and mySQL db
create Vault policy that can see secrets. 'data' path needs to be inserted when dealing with API level. Also you can check API path in vault ui where you have created a secret to get correct path for this policy. 
`kubectl exec -n $VAULT_K8S_NAMESPACE vault-0 -it -- sh`
`echo 'path "secrets/data/application/*" { capabilities=["read"] }' | vault policy write vso-policy -`