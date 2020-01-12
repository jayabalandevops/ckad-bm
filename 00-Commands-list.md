## Useful commands

- These are few important tips about commands.

```shell
kubectl create --help
kubectl explain pods.spec
alias k=kubectl
k version
kubectl config set-context abcd-project --namespace=abcd-facebook
kubectl get ns
kubectl get namespace
kubectl describe pvc claim
kubectl describe persistentvolumeclaim claim
kubectl delete pod nginx --grace-period=0 --force
if [ ! -d ~/tmp ]; then mkdir -p ~/tmp; fi; while true; do echo $(date) >> ~/tmp/date.txt; sleep 5; done;
cat ~/tmp/date.txt 
kubectl describe pods | grep -C 10 "author=John Doe"
kubectl get pods -o yaml | grep -C 5 labels;
```
