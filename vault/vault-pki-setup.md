# TLS/HTTPS with Traefik, cert-manager & Vault

Generate the root CA

vault write pki/root/generate/internal \
    common_name=cluster.com \
    ttl=8760h

Set the PKI URLs; The cert-manager will call these URLs to get the certificates and validate certificates that are revoked.

vault write pki/config/urls \
    issuing_certificates="https://vault-internal.vault.svc.cluster.local:8200/v1/pki/ca" \
    crl_distribution_points="https://vault-internal.vault.svc.cluster.local:8200/v1/pki/crl"

Define the PKI role (see it in vault UI /pki/roles/cluster-dot-com  ) 
Set the allowed domain to cluster.com, and also allow subdomains

vault write pki/roles/cluster-dot-com \
    allowed_domains=cluster.com \
    allow_subdomains=true \
    require_cn=false \
    max_ttl=72h

    Reponse: {
        Key                                   Value
        ---                                   -----
        allow_any_name                        false
        allow_bare_domains                    false
        allow_glob_domains                    false
        allow_ip_sans                         true
        allow_localhost                       true
        allow_subdomains                      true
        allow_token_displayname               false
        allow_wildcard_certificates           true
        allowed_domains                       [cluster.com]
        allowed_domains_template              false
        allowed_other_sans                    []
        allowed_serial_numbers                []
        allowed_uri_sans                      []
        allowed_uri_sans_template             false
        allowed_user_ids                      []
        basic_constraints_valid_for_non_ca    false
        client_flag                           true
        cn_validations                        [email hostname]
        code_signing_flag                     false
        country                               []
        email_protection_flag                 false
        enforce_hostnames                     true
        ext_key_usage                         []
        ext_key_usage_oids                    []
        generate_lease                        false
        issuer_ref                            default
        key_bits                              2048
        key_type                              rsa
        key_usage                             [DigitalSignature KeyAgreement KeyEncipherment]
        locality                              []
        max_ttl                               72h
        no_store                              false
        not_after                             n/a
        not_before_duration                   30s
        organization                          []
        ou                                    []
        policy_identifiers                    []
        postal_code                           []
        province                              []
        require_cn                            false
        serial_number_source                  json-csr
        server_flag                           true
        signature_bits                        256
        street_address                        []
        ttl                                   0s
        use_csr_common_name                   true
        use_csr_sans                          true
        use_pss                               false
    }
Set the service account for the issuer; include your namespace; In vault UI go to Access panel -> kubernetes/issuer
    Note: if it fails, it might help : vault auth enable kubernetes

vault write auth/kubernetes/role/jenkins-role \
    bound_service_account_names=jenkins-sa \
    bound_service_account_namespaces=jenkins

vault write auth/kubernetes/role/sonar-role \
    bound_service_account_names=sonar-sa \
    bound_service_account_namespaces=sonar

vault write auth/kubernetes/role/argo-role \
    bound_service_account_names=argo-sa \
    bound_service_account_namespaces=argo

Create ACL policies
The following policy grants permissions to use the PKI engine:
    pki* → Read/list PKI configuration (e.g., fetch CA cert)
    pki/sign/cluster-dot-com → Sign CSR manually (optional)
    pki/issue/cluster-dot-com → Issue certs using the PKI role cluster-dot-com
    🟢 This policy is needed by any identity that should request certificates from Vault.

vault policy write pki - <<EOF
path "pki*" { capabilities = ["read", "list"] }
path "pki/sign/cluster-dot-com" { capabilities = ["create", "update"] }
path "pki/issue/cluster-dot-com" { capabilities = ["create"] }
EOF

✅ Then the k8s secret-token will be used by cert-manager in the Vault Issuer or ClusterIssuer to authenticate.

k create -f issuers-service-accounts.yaml

Create Issuer resource, This is called by the cert-manager to request the certificate when the certificate is expired/required.
We point it to the vault server, provide the path to the PKI that we configured in the previous section, and provide the service account to use (issuer) 

