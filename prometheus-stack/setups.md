### Persistant Storage
Prometheus - NFS provision is required ~ 8GB
Grafana - NFS provision is required ~ 3GB
Alertmanager - persistance is not required
Kube-State-Metrics - persistance is not required
Node Exporter, etc - persistance is not required

## Alertmanager
Create a Kubernetes Secret containing your password file

Put your SMTP password (app password from gamil) in a file (e.g. auth_password) and create a Secret:
```kubectl -n monitoring create secret generic alertmanager-smtp-secret \
  --from-file=auth_password```   

Mount the Secret into your Alertmanager pod, the file name is confgured in field 'path' (for me it is smtp_auth_password)

    ```k apply -f prometheus-stack/10-alert-cm.yaml
    k apply -f prometheus-stack/11-alert-deploy-svc.yaml```

