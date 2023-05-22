## 前言

因为我们在管理 kubernets 集群的时候，可能需要给予不同的用户不同的权限； 这个时候 kubernetes 提供的 serviceaccount 可以提供一个很不错的我们管理的切入点， 虽然 serviceaccount 本意是给运行在集群中服务使用的；

```plain
Service account是为了方便Pod里面的进程调用Kubernetes API或其他外部服务而设计的。它与User account不同
　　1.User account是为人设计的，而service account则是为Pod中的进程调用Kubernetes API而设计；
　　2.User account是跨namespace的，而service account则是仅局限它所在的namespace；
　　3.每个namespace都会自动创建一个default service account
　　4.Token controller检测service account的创建，并为它们创建secret
　　5.开启ServiceAccount Admission Controller后
       1.每个Pod在创建后都会自动设置spec.serviceAccount为default（除非指定了其他ServiceAccout）
　　　　2.验证Pod引用的service account已经存在，否则拒绝创建
　　　　3.如果Pod没有指定ImagePullSecrets，则把service account的ImagePullSecrets加到Pod中
　　　　4.每个container启动后都会挂载该service account的token和ca.crt到/var/run/secrets/kubernetes.io/serviceaccount/
```



## 案例1：

```shell
[root@k8s-master sa]# kubectl create namespace qiangungun #创建一个名称空间
[root@k8s-master sa]# kubectl get sa -n qiangungun  #名称空间下，会自动生成serivceaccount
NAME      SECRETS   AGE
default   1         74s
[root@k8s-master sa]# kubectl get secrets -n qiangungun #同时会成成一个secrets
NAME                  TYPE                                  DATA   AGE
default-token-9bwxj   kubernetes.io/service-account-token   3      2m21s
```



### **在创建的名称空间中新建一个pod**

```shell
[root@k8s-master sa]# cat pod_demo.yml 
kind: Pod
apiVersion: v1
metadata:
  name: task-pv-pod
  namespace: qiangungun
spec:
  containers:
  - name: nginx
    image: ikubernetes/myapp:v1
    ports:
     - containerPort: 80
       name: www
[root@k8s-master sa]# kubectl apply -f pod_demo.yml  
[root@k8s-master sa]# kubectl get pod -n qiangungun
NAME          READY   STATUS    RESTARTS   AGE
task-pv-pod   1/1     Running   0          59s
[root@k8s-master sa]# kubectl get pod -n qiangungun -o yaml
```



![img](assets/k8s-ServiceAccount/image-20210918114253389.png)



可以看到，pod将serviceaccount中的secrets挂载到了pod内部的/var/run/secrets/kubernetes.io/serviceaccount



![img](assets/k8s-ServiceAccount/image-20210918114424239.png)



**名称空间新建的pod如果不指定sa，会自动挂载当前名称空间中默认的sa(default)**



```shell
进入到pod中，查看验证一下，果然是有的
[root@k8s-master sa]# kubectl exec -it task-pv-pod /bin/sh -n qiangungun
/ # cd /var/run/secrets/kubernetes.io/serviceaccount/
/var/run/secrets/kubernetes.io/serviceaccount # ls
ca.crt     namespace  token
```



## 案例2：

创建ServiceAccount

```shell
[root@k8s-master sa]# cat test-sa.yml 
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system
```



将这个ServiceAccount跟ClusterRole进行绑定：

```shell
[root@k8s-master sa]# cat test-clusterrolebinding.yml 
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
```



这样ServiceAccount就拥有了ClusterRole的权限



```shell
[root@k8s-master sa]# cat test-pod.yml 
apiVersion: v1
kind: Pod
metadata:
  name: pod
  namespace: kube-system
spec:
  containers:
  - name: podtest
    image: nginx
  serviceAccountName: kubernetes-dashboard-admin
```

总结：serviceaccount是pod访问apiserver的权限认证，每个pod中都默认会绑定当前名称空间默认的sa中的secrets，来保证pod能正常的访问apiserver；



serviceaccount能和clusterrole进行绑定。绑定成功之后，serviceaccount的权限就和clusterrole一致了。后续pod再使用这个serviceaccount,pod的权限也就相同了；



不过这个，是pod进程的权限；如果真正关乎与用户的权限的话，还是使用RBAC