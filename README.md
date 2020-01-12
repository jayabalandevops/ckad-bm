# CKAD Concepts from Benjamin

## Kubernetes Object Structure

- API Version `v1`, `apps/v1`, ``
- Kind `Pod`, `Deployment`, `Quota`, `ConfigMap`, ``
- Metadata `Name`, `Namespace`, `Labels`, ``
- Spec `Desired state`
- Status `Actual state`

## ConfigMap

Creating ConfigMaps (Imperative)

```shell
# Literal values
$ kubectl create configmap db-config --from-literal=db=staging

# Single file with environment variables
$ kubectl create configmap db-config --from-env-file=config.env

# File or Directory
$ kubectl create configmap db-config --from-file=config.txt
```
Definition of a ConfigMap (declarative)

```shell
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

```shell
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

```shell
$kubectl exec -it nginx -- envFrom
DB=staging
USERNAME=jdoe
...
```

2. ConfigMap in Pod as Volumes

```shell
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

```shell
$ kubectl exec -it backend -- /bin/sh
# ls /etc/config
db
username
# cat /etc/config/db
staging
```
