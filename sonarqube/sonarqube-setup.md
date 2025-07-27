# sonarqube-kubernetes

Encode USERNAME and PASSWORD of Postgres using following commands:
--------
Create the Secret using kubectl apply:
-------
    kubectl apply -f postgres-secrets.yml
-------
Deploy local NFS provisioner
-------
    kubectl apply -f 2-nfs-sonar-provisioner.yaml 
-------
Create PVC for Postgres using yaml file; Check if it is BOUND;
-----
    kubectl apply -f 3-postgres-pvc.yaml
-----------
Deploying Postgres with kubectl apply:
-----------
    kubectl apply -f 4-postgres-deployment.yaml
    kubectl apply -f 5-postgres-service.yaml
----------
Let's check the mount points in the pod
---------
    kubectl exec postgres-99f7fc766-9sv56 -n sonar -- df /var/lib/postgresql/data
-------
Create PVC for Sonarqube:
-------------
    kubectl apply -f 1-sonar-pvcs.yml
-------
Create configmaps for URL which we use in Sonarqube:
-------
    kubectl apply -f sonar-configmap.yaml
Deploy Sonarqube and service:
-------------
    kubectl apply -f 8-sonar-deployment.yml
Create a secret tls-ca (containing Vault CA TLS cert) for cert manager to trust Vault. 
-------
    kubectl create secret generic tls-ca \
        -n sonar \
        --from-file=ca.crt=vault.ca
Create cert issuer roles, role bindings, certificate and issuer resources:
-------
    kubectl create -f 9-cert_issuer_rbac.yaml
Create IngressRoute. Remember to update /etc/hosts with appropriate domain:
-------
    kubectl create -f 10-sonar-ingress.yml
Access https://sonar.cluster.com in your browser. Default Credentials for Sonarqube:
-------

  