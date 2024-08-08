# K8S教育訓練
### Service
```
root@k8s1:~# vi pod
```
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: andy
spec:
  containers:
  - name: nginx
    image: nginx
```
```
root@k8s1:~# kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          17s   10.244.2.4   k8s3.andy.com   <none>           <none>
```
