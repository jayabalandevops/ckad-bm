# CKAD Concepts from Benjamin

## Kubernetes Object Structure

- API Version `v1`, `apps/v1`, ``
- Kind `Pod`, `Deployment`, `Quota`, `ConfigMap`, `ResourceQuota`, `Secret`, `ServiceAccount`
- Metadata `Name`, `Namespace`, `Labels`, ``
- Spec `Desired state`
- Status `Actual state`

## ConfigMap

Creating ConfigMaps (Imperative)

```bash
# Literal values
$ kubectl create configmap db-config --from-literal=db=staging

# Single file with environment variables
$ kubectl create configmap db-config --from-env-file=config.env

# File or Directory
$ kubectl create configmap db-config --from-file=config.txt
```
Definition of a ConfigMap (declarative)

```yaml
apiVersion: v1
data:
  db: staging
  username: jdoe
kind: ConfigMap
metadata:
  name: db-config
```

Mounting a ConfigMap

- Two options

1. ConfigMap environment variables in Pod

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: backend
spec:
  containers:
  - image: nginx
    name: backend
    envFrom:
      - configMapRef:
          name: db-config
```

```bash
$ kubectl exec -it nginx -- envFrom
DB=staging
USERNAME=jdoe
...
```

2. ConfigMap in Pod as Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: nginx
    name: backend
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: db-config
```

```bash
$ kubectl exec -it backend -- /bin/sh
# ls /etc/config
db
username
# cat /etc/config/db
staging
```
