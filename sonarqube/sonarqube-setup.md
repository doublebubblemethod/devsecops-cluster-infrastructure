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
    kubectl apply -f postgres-service.yaml
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
Deploy Sonarqube:
-------------
    kubectl apply -f sonar-deployment.yml
    kubectl apply -f sonar-service.yml
Create a secret for IngressRoute (cd traefik folder):
-------
    k create secret tls ingress-local-cert --cert=ingress-cluster-public.crt --key=ingress-cluster-com.key -n sonar


Default Credentials for Sonarqube:
-------
    UserName: admin
    PassWord: admin
sonarMaster token: 8d824f346e2ba02111ce7c2f34ef41a29a7184cd    
Now we can cleanup by using below commands:
--------
