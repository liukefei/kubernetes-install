# Kubernetes Dashboard

```

## 创建Dashboard
```
环境：Kubernetes1.10，Dashboard1.8.3

在k8s中 dashboard可以有两种访问方式：kubeconfig（HTTPS）和token（http）本篇先来介绍下Token方式的访问。（Token访问是无登录密码的，简单方便）

```
下载官方的dashboard YAML文件或者下载我的YAML

[root@linux-node1 ~]# git clone https://github.com/gh-Devin/kubernetes-dashboard.git

下载dashboard镜像文件 https://github.com/datagrand/k8s_deploy/tree/master/images/dashboard
kubernetes-dashboard-amd64.tgz
kubernetes-dashboard-init-amd64.tgz

在集群的节点上部署镜像文件或上传到本地的镜像仓库
##this is kubernetes-dashboard-amd64.tgz
##Instructions
[root@linux-node2 ~]# docker load < kubernetes-dashboard-amd64.tgz
[root@linux-node2 ~]# docker tag [IMAGE ID] k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3

##this is kubernetes-dashboard-init-amd64.tgz
##Instructions
[root@linux-node2 ~]# docker load < kubernetes-dashboard-init-amd64.tgz
[root@linux-node2 ~]# docker tag [IMAGE ID] k8s.gcr.io/kubernetes-dashboard-init-amd64:v1.0.1

创建dashboard
[root@linux-node1 ~]# kubectl create -f kubernetes-dashboard.yaml

[root@linux-node1 ~]# kubectl get svc,pod --all-namespaces -o wide| grep dashboard 
kube-system   service/kubernetes-dashboard-external   NodePort    10.1.39.91    <none>        9090:30090/TCP   23m       k8s-app=kubernetes-dashboard
kube-system   pod/kubernetes-dashboard-5cc6564db9-7c28l   1/1       Running            0          23m       10.2.56.22   192.168.56.13

访问所在节点的 30090端口 http://192.168.56.13:30090

注意：pod无法正常启动
如果pod出现问题需要删除dashboard创建时所需的资源，然后查看节点镜像拉取情况
[root@linux-node1 ~]# kubectl delete -f kubectl create -f kubernetes-dashboard.yaml

注意：界面访问权限报错
RBAC访问控制策略，可以使用kubectl或Kubernetes API进行配置。使用RBAC可以直接授权给用户，让用户拥有授权管理的权限，这样就不再需要直接触碰Master Node。
修改kubernetes-dashboard.yaml文件中的ServiceAccount名称

[root@linux-node1 ~]# vi kubernetes-dashboard.yaml
147 serviceAccountName: kubernetes-dashboard-admin
[root@linux-node1 ~]# kubectl create -f kubernetes-dashboard.yaml

```


## 创建CoreDNS
```
[root@linux-node1 ~]# kubectl create -f coredns.yaml 

[root@linux-node1 ~]# kubectl get pod -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
coredns-77c989547b-9pj8b                1/1       Running   0          6m
coredns-77c989547b-kncd5                1/1       Running   0          6m
```

