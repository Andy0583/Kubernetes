# K8S教育訓練
### Service-NodePort
```
root@k8s1:~# vi pod

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

root@k8s1:~# kubectl apply -f pod
pod/nginx created

root@k8s1:~# kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          17s   10.244.2.4   k8s3.andy.com   <none>           <none>
```
```
root@k8s1:~# vi sv1

apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 8080
      nodePort: 30008
  selector:
    app: andy

root@k8s1:~# kubectl apply -f sv1
service/nodeport-service created

root@k8s1:~# kubectl get service
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP          47h
nodeport-service   NodePort    10.104.215.55   <none>        8080:30008/TCP   22s
```
* 在外部開啟瀏覽器輸入任一Node IP：30008，即可連入Pod內App。

### Service-ClusterIP
```
root@k8s1:~# vi sv2

apiVersion: v1
kind: Service
metadata: 
  name: clusterip-service
spec:
  type: ClusterIP
  ports:
    - targetPort: 80
      port: 80
  selector: 
    app: andy

root@k8s1:~# kubectl apply -f sv2
service/nodeport-service created

root@k8s1:~# kubectl get service
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
clusterip-service   ClusterIP   10.98.120.204   <none>        80/TCP    7m28s
kubernetes          ClusterIP   10.96.0.1       <none>        443/TCP   2d

root@k8s1:~# telnet 10.98.120.204 80
Trying 10.98.120.204...
Connected to 10.98.120.204.
Escape character is '^]'.

```
