# sonarqube-kubernetes

Encode USERNAME and PASSWORD of Postgres and create the Secret using kubectl apply:   
    kubectl apply -f postgres-secrets.yml  

## Deploy local NFS provisioner:
    kubectl apply -f 2-nfs-sonar-provisioner.yaml   

Create PVC for Postgres using yaml file; Check if it is BOUND;
    kubectl apply -f 3-postgres-pvc.yaml  

## Deploying Postgres....
    helm chart
log into db to check id database and user account is set up correctly:
`psql -U sonaruser0910 -d sonarsql`  
If you access the container using bash, make sure that you execute `/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash`  in order to avoid the error "psql: local user with ID 1001} does not exist"

check if client connection works:
`kubectl run postgres-qube-client --rm --tty -i --restart='Never' --namespace sonar --image docker.io/bi
tnami/postgresql:17.5.0-debian-12-r20       --command -- psql --host postgres-qube -d sonarsql -U sonaruser0910 -p 5432`   

My problem: sonarqibe could not connect to th epostgres because i hav elayerd up 3 network policies; although they were permissive, still deleteng them helped me!   
`kubectl run netshoot1 --rm --tty -i --restart='Never' --namespace sonar --image nicolaka/netshoot` 
    > `nslookup postgres-qube`
    `nc -vz postgres-qube 5432`
    
# Sonarqube deploy with helm chart
1. prepare sonarqube/helm-overrides.yaml
2. prepare VSO secret provision:
       ``` kubectl create secret generic tls-ca \
            -n sonar \
            --from-file=ca.crt=vault.ca```   
    check if secret has correct values
3. Install chart
`helm upgrade -n sonar sonar sonarqube/sonarqube --values helm-sonar-overrides.yaml`   
4. create network policy to allow egress trafik for init containers:
    `k create -f sonarqube/netpolicy.yaml`  

## set up ingress route with tls provisioning
Create cert issuer roles, role bindings, certificate and issuer resources:
    kubectl create -f 9-cert_issuer_rbac.yaml
Create IngressRoute. Remember to update /etc/hosts with appropriate domain:
-------
    kubectl create -f 10-sonar-ingress.yml
Access https://sonar.cluster.com in your browser. Default Credentials for Sonarqube:
-------

  