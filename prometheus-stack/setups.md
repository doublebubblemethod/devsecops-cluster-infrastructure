### Persistant Storage
Prometheus - NFS provision is required ~ 8GB
Grafana - NFS provision is required ~ 3GB
Alertmanager - persistance is not required
Kube-State-Metrics - persistance is not required
Node Exporter, etc - persistance is not required

## Prometheus Scraping
* Install a jenkins plugin "Prometheus metrics plugin"
✔️ Use proper PVC-backed storage

Ensure each Prometheus instance uses a dedicated PersistentVolume (block-based or xfs/ext4)—not shared storage:

    Avoid shared network storage (NFS, CephFS) for Prometheus TSDB.

    Use a unique PVC per pod, ensuring isolation and lock capability.

✔️ Remove stale lock file (carefully)

If the storage is dedicated and still fails:

    Scale down or stop the Prometheus pod/container.

    Exec into the volume mount and remove the lock file:

kubectl exec pod/prometheus-<id> -- rm /prometheus/lock


## Alertmanager
Create a Kubernetes Secret containing your password file

Put your SMTP password (app password from gamil) in a file (e.g. auth_password) and create a Secret:
```kubectl -n monitoring create secret generic alertmanager-smtp-secret \
  --from-file=auth_password```   

Mount the Secret into your Alertmanager pod, the file name is confgured in field 'path' (for me it is smtp_auth_password)

    ```k apply -f prometheus-stack/10-alert-cm.yaml
    k apply -f prometheus-stack/11-alert-deploy-svc.yaml```

## Grafana
Recover Locked Database File [https://opsverse.io/2022/12/15/grafana-sqlite-and-database-is-locked/?utm_source=chatgpt.com]
```   
initContainers:
- name: grafanadb-clone-and-replace
  image: keinos/sqlite3
  command:
  - "/bin/sh"
  - "-c"
  - "/usr/bin/sqlite3 /var/lib/grafana/grafana.db '.clone /var/lib/grafana/grafana.db.clone'; mv /var/lib/grafana/grafana.db.clone /var/lib/grafana/grafana.db; chmod a+w /var/lib/grafana/grafana.db"
  imagePullPolicy: IfNotPresent
  securityContext:
    runAsUser: 0
  volumeMounts:
  - name: storage
    mountPath: "/var/lib/grafana"
```   
We need the sqlite3 client (to interact with the DB), so that image is pulled. When the container starts up (before the grafana-server itself, as this is an init container), it clones the locked grafana.db. It then replaces /var/lib/grafana/grafana.db with the newly cloned DB file.


The first thing you can try is to enable Write-Ahead Logging (WAL). In grafana.ini:  
``` 
  [database]:
  ...
  wal: true
  ...
  cache_mode: shared
  ...
```   
This should only be changed if enabling WAL is not possible for you. From their official documentation, SQLite claims it’s obsolete.

{
  "id": 4,
  "type": "stat",
  "title": "Used",
  "gridPos": {
    "x": 0,
    "y": 5,
    "h": 3,
    "w": 4
  },
  "fieldConfig": {
    "defaults": {
      "mappings": [
        {
          "type": "special",
          "options": {
            "match": "null",
            "result": {
              "text": "N/A"
            }
          }
        }
      ],
      "thresholds": {
        "mode": "absolute",
        "steps": [
          {
            "value": null,
            "color": "green"
          },
          {
            "value": 80,
            "color": "red"
          }
        ]
      },
      "unit": "bytes"
    },
    "overrides": []
  },
  "pluginVersion": "12.2.0-16557133545",
  "targets": [
    {
      "expr": "sum(
      container_memory_working_set_bytes{
      pod=~"^$Deployment.*$",
      kubernetes_io_hostname=~"^tools-.*$",
      container!="POD",
      container!=""
    }
    ) by (pod, namespace)",
      "format": "time_series",
      "intervalFactor": 2,
      "refId": "A",
      "step": 1800,
      "datasource": {
        "uid": "P1809F7CD0C75ACF3",
        "type": "prometheus"
      }
    }
  ],
  "maxDataPoints": 100,
  "datasource": {
    "uid": "P1809F7CD0C75ACF3",
    "type": "prometheus"
  },
  "options": {
    "reduceOptions": {
      "values": false,
      "calcs": [
        "lastNotNull"
      ],
      "fields": ""
    },
    "orientation": "horizontal",
    "textMode": "auto",
    "wideLayout": true,
    "colorMode": "none",
    "graphMode": "none",
    "justifyMode": "auto",
    "showPercentChange": false,
    "percentChangeColorMode": "standard"
  }
}
