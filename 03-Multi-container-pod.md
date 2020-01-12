# Multi-Container Pods

## Creating an Init Container

- Initialize a web application by standing up environment-specific configuration through an init container.

- Kubernetes runs an init container before the main container. In this scenario, the init container retrieves configuration files from a remote location and makes it available to the application running in the main container. The configuration files are shared through a volume mounted by both containers. The running application consumes the configuration files and can render its values.

1. Create a new Pod in a YAML file named `business-app.yaml`. The Pod should define two containers, one init container and one main application container. Name the init container `configurer` and the main container `web`. The init container uses the image `busybox`, the main container uses the image `jayabalandevops/nodejs-read-config:1.0.0`. Expose the main container on port 8080.
2. Edit the YAML file by adding a new volume of type `emptyDir` that is mounted at `/usr/shared/app` for both containers.
3. Edit the YAML file by providing the command for the init container. The init container should run a `wget` command for downloading the file `https://raw.githubusercontent.com/jayabalandevops/ckad-bm/master/app/config/config.json` into the directory `/usr/shared/app`.
4. Start the Pod and ensure that it is up and running.
5. Run the command `curl localhost:8080` from the main application container. The response should render a database URL derived off the information in the configuration file.
6. (Optional) Discuss: How would you approach a debugging a failing command inside of the init container?


<details><summary>Show Solution</summary>
<p>
- Solution

Start by generating the basic skeleton of the Pod.

```shell
$ kubectl run business-app --image=jayabalandevops/nodejs-read-config:1.0.0 --restart=Never --port=8080 -o yaml --dry-run > business-app.yaml
```

You should end up with the following configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: business-app
  name: business-app
spec:
  containers:
  - image: jayabalandevops/nodejs-read-config:1.0.0
    name: business-app
    ports:
    - containerPort: 8080
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Edit the file to change the main application container. Moreover, add the init container section.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: business-app
spec:
  initContainers:
  - name: configurer
    image: busybox
  containers:
  - image: jayabalandevops/nodejs-read-config:1.0.0
    name: web
    ports:
    - containerPort: 8080
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Add the volume and mount it to the path `/usr/shared/app` for each container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: business-app
spec:
  initContainers:
  - name: configurer
    image: busybox
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
  containers:
  - image: jayabalandevops/nodejs-read-config:1.0.0
    name: web
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: configdir
    emptyDir: {}
status: {}
```

Define the command for init container for downloading the `config.json` file. The final YAML configuration should look similar to the one below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: business-app
spec:
  initContainers:
  - name: configurer
    image: busybox
    command:
    - wget
    - "-O"
    - "/usr/shared/app/config.json"
    - https://raw.githubusercontent.com/jayabalandevops/ckad-bm/master/app/config/config.json
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
  containers:
  - image: jayabalandevops/nodejs-read-config:1.0.0
    name: web
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: configdir
    emptyDir: {}
status: {}
```

Create the Pod from the YAML file. During the creation of the Pod you can follow the creation of individual containers.

```shell
$ kubectl apply -f business-app.yaml
pod/business-app created

$ kubectl get pods
NAME           READY   STATUS     RESTARTS   AGE
business-app   0/1     Init:0/1   0          4s

$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
business-app   1/1     Running   0          37m
```

Once the application is running, shell into the container. The mounted volume path contains the downloaded configuration file. The `curl` command renders the values from the configuration file.

```shell
$ kubectl exec business-app -it -- /bin/sh
# ls /usr/shared/app
config.json
# curl localhost:8080
Database URL: localhost:5432/customers
```

- Optional

> How would you approach a debugging a failing command inside of the init container?

Adding a temporary `sleep` command to the init container help with reserving time for debugging the data available on the mounted volume. You simply `kubectl exec` into the container and inspect the contents.

</p>
</details>


## Implementing the Adapter Pattern

- Adapter pattern for multi-container-pod

- The adapter pattern helps with providing a simplified, homogenized view of an application running within a container. For example, we could stand up another container that unifies the log output of the application container. As a result, other monitoring tools can rely on a standardized view of the log output without having to transform it into an expected format.

1. Create a new Pod in a YAML file named `adapter.yaml`. The Pod declares two containers. The container `app` uses the image `busybox` and runs the command `while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; sleep 5; done;`. The adapter container `transformer` uses the image `busybox` and runs the command `sleep 20; while true; do while read LINE; do echo "$LINE" | cut -f2 -d"|" >> $(date +%Y-%m-%d-%H-%M-%S)-transformed.txt; done < /var/logs/diskspace.txt; sleep 20; done;` to strip the log output off the date for later consumption by a monitoring tool. Be aware that the logic does not handle corner cases (e.g. automatically deleting old entries) and would look different in production systems.
2. Before creating the Pod, define an `emptyDir` volume. Mount the volume in both containers with the path `/var/logs`.
3. Create the Pod, log into the container `transformer`. The current directory should continuously write a new file every 20 seconds.


<details><summary>Show Solution</summary>
<p>

- Solution

You can create the initial Pod setup with the following command.

```shell
$ kubectl run adapter --image=busybox --restart=Never -o yaml --dry-run -- /bin/sh -c 'while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; sleep 5; done;' > adapter.yaml
```
The final Pod YAML file should look something like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: adapter
spec:
  volumes:
    - name: config-volume
      emptyDir: {}
  containers:
  - args:
    - /bin/sh
    - -c
    - 'while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; sleep 5; done;'
    image: busybox
    name: app
    volumeMounts:
      - name: config-volume
        mountPath: /var/logs
    resources: {}
  - image: busybox
    name: transformer
    args:
    - /bin/sh
    - -c
    - 'sleep 20; while true; do while read LINE; do echo "$LINE" | cut -f2 -d"|" >> $(date +%Y-%m-%d-%H-%M-%S)-transformed.txt; done < /var/logs/diskspace.txt; sleep 20; done;'
    volumeMounts:
      - name: config-volume
        mountPath: /var/logs
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
You should find that a new text file in the current directory every 20 seconds. Each of the files contain the disk space without the date prefix.

```shell
$ kubectl create -f adapter.yaml
$ kubectl exec adapter --container=transformer -it -- /bin/sh
# cat /var/logs/diskspace.txt
Tue Nov 12 15:13:48 UTC 2019 | 4.0K	/root
Tue Nov 12 15:13:53 UTC 2019 | 4.0K	/root
Tue Nov 12 15:13:58 UTC 2019 | 4.0K	/root
Tue Nov 12 15:14:03 UTC 2019 | 4.0K	/root
# ls -l
-rw-r--r--    1 root     root            60 Nov 12 15:14 2019-11-12-15-14-10-transformed.txt
-rw-r--r--    1 root     root           108 Nov 12 15:14 2019-11-12-15-14-30-transformed.txt
...
# cat 2019-11-12-15-14-10-transformed.txt
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
 4.0K	/root
# exit
```


</p>
</details>