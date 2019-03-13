---
layout:     post
title:      "Docker Kubenetets"
subtitle:   "Docker Kubenetets"
date:       2019-03-06 13:30:00
author:     "Psl"
catalog:    true
tags:
  - Kubenetets
---

## 集群快速搭建

### minikube安装

```bash
wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
rm -f minikube-linux-amd64
cd /usr/local/bin/
sudo chmod a+x minikube
```

### kubectl安装

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.4/bin/linux/amd64/kubectl
sudo mv kubectl /usr/local/bin/kubectl
rm -f kubectl
cd /usr/local/bin/
sudo chmod a+x kubectl
```

### 使用

```bash
minikube start      //创建k8s集群
minikube ssh        //进入虚拟机内
```

start过程中从从k8s.gcr.io拉取镜像失败
报错信息:
```bash
Using Kubernetes version: v1.13.4
[preflight] Running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
	[WARNING Swap]: running with swap on is not supported. Please disable swap
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-apiserver:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-controller-manager:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-scheduler:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/kube-proxy:v1.13.4: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/pause:3.1: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/etcd:3.2.24: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
	[ERROR ImagePull]: failed to pull image k8s.gcr.io/coredns:1.2.6: output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
, error: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`

```

docker.io仓库对google的容器做了镜像，可以通过下列命令下拉取相关镜像：

```bash
docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.11.4
docker pull mirrorgooglecontainers/kube-controller-manager-amd64:v1.11.4
docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.11.4
docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.11.4
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd-amd64:3.2.24
docker pull coredns/coredns:1.2.6
```

版本信息需要根据实际情况进行相应的修改。通过docker tag命令来修改镜像的标签

```bash

docker tag docker.io/mirrorgooglecontainers/kube-proxy-amd64:v1.11.3 k8s.gcr.io/kube-proxy-amd64:v1.11.4
docker tag docker.io/mirrorgooglecontainers/kube-scheduler-amd64:v1.11.3 k8s.gcr.io/kube-scheduler-amd64:v1.11.4
docker tag docker.io/mirrorgooglecontainers/kube-apiserver-amd64:v1.11.3 k8s.gcr.io/kube-apiserver-amd64:v1.11.4
docker tag docker.io/mirrorgooglecontainers/kube-controller-manager-amd64:v1.11.3 k8s.gcr.io/kube-controller-manager-amd64:v1.11.4
docker tag docker.io/mirrorgooglecontainers/etcd-amd64:3.2.18  k8s.gcr.io/etcd-amd64:3.2.24
docker tag docker.io/mirrorgooglecontainers/pause:3.1  k8s.gcr.io/pause:3.1
docker tag docker.io/coredns/coredns:1.2.6  k8s.gcr.io/coredns:1.2.6
```


## pod

### 创建nginx pod

基本命令:
```bash
kubectl create -f <podfile.yml>     //创建
kubectl delete -f <podfile.yml>     //删除
kubectl get pods      //查看pod信息
```
k8s 最小单位,创建pod

创建pod_nginx.yml文件,内容如下

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

执行
```bash
kubectl create -f pod_nginx.yml
```

![avatar](/img/in-post/2019-03-08/Snip20190309_2.png)


查看nginx pod详细信息
```bash
kubectl describe pods nginx
Name:               nginx
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               minikube/10.0.2.15
Start Time:         Sat, 09 Mar 2019 20:46:25 +0800
Labels:             app=nginx
Annotations:        <none>
Status:             Running
IP:                 172.17.0.4
Containers:
  nginx:
    Container ID:   docker://b3c9a2e399a35712373de92b619b0bb79bc5075718ec04f614e70e92ac998b16
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:98efe605f61725fd817ea69521b0eeb32bef007af0e3d0aeb6258c6e6fe7fc1a
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sat, 09 Mar 2019 20:47:20 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-q8wk4 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-q8wk4:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-q8wk4
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  8m16s  default-scheduler  Successfully assigned default/nginx to minikube
  Normal  Pulling    8m15s  kubelet, minikube  pulling image "nginx"
  Normal  Pulled     7m21s  kubelet, minikube  Successfully pulled image "nginx"
  Normal  Created    7m21s  kubelet, minikube  Created container
  Normal  Started    7m21s  kubelet, minikube  Started container
```

### 端口映射

```bash
kubectl port-forward nginx 8080:80
```

打开浏览器输入[127.0.0.1:8080](http://127.0.0.1:8080)即可看到nginx 欢迎页

## Replication Controller（RC）

应用托管在K8S后，K8S需要保证应用能够持续运行，这是RC的工作内容

### 主要功能

确保pod数量：RC用来管理正常运行Pod数量，一个RC可以由一个或多个Pod组成，在RC被创建后，系统会根据定义好的副本数来创建Pod数量。在运行过程中，如果Pod数量小于定义的，就会重启停止的或重新分配Pod，反之则杀死多余的。

确保pod健康：当pod不健康，运行出错或者无法提供服务时，RC也会杀死不健康的pod，重新创建新的。

弹性伸缩 ：在业务高峰或者低峰期的时候，可以通过RC动态的调整pod的数量来提高资源的利用率。同时，配置相应的监控功能（Hroizontal Pod Autoscaler），会定时自动从监控平台获取RC关联pod的整体资源使用情况，做到自动伸缩。

滚动升级：滚动升级为一种平滑的升级方式，通过逐步替换的策略，保证整体系统的稳定，在初始化升级的时候就可以及时发现和解决问题，避免问题不断扩大。


### 横向扩展

yml文件实例(rc_nginx.yml)

```bash
apiVersion: v1
kind: ReplicationController 
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

执行:
```bash
kubectl create -f rc_nginx.yml
```

![avatar](/img/in-post/2019-03-08/Snip20190309_3.png)

通过ReplicationController方式可以自动化维持pods数目，以方便治理某个pod宕机不可用情况

eg:删除一个pod

![avatar](/img/in-post/2019-03-08/Snip20190309_5.png)

删掉一个后status立马变为ContainerCreating去重新创建一个pod

扩展升级

![avatar](/img/in-post/2019-03-08/Snip20190309_6.png)

## ReplicaSet

“升级版”的RC。RS也是用于保证与label selector匹配的pod数量维持在期望状态。

区别：
```bash
1、RC只支持基于等式的selector（env=dev或environment!=qa），但RS还支持新的，基于集合的selector（version in (v1.0, v2.0)或env notin (dev, qa)），这对复杂的运维管理很方便。
 
2、升级方式
    RS不能使用kubectlrolling-update进行升级
    kubectl rolling-update专用于rc
    RS升级使用deployment或者kubectl replace命令
```

yml文件实例(rs_nginx.yml)

```bash
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      name: nginx
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

执行:
```bash
kubectl create -f rs_nginx.yml
```

![avatar](/img/in-post/2019-03-08/Snip20190309_9.png)


扩展

kubectl scale rs nginx --replicas=2
kubectl scale rs nginx --replicas=4

## Deployments

deployment 是用来管理无状态应用的，面向的集群的管理，而不是面向的是一个不可变的个体

Deployment 为Pod 和 ReplicaSet 之上，提供了一个声明式定义（declarative）方法，用来替代以前的ReplicationController 来方便的管理应用。

你只需要在 Deployment 中描述您想要的目标状态是什么，Deployment controller 就会帮您将 Pod 和ReplicaSet 的实际状态改变到您的目标状态。您可以定义一个全新的 Deployment 来

创建 ReplicaSet 或者删除已有的 Deployment 并创建一个新的来替换。也就是说Deployment是可以管理多个ReplicaSet的

典型的应用场景包括：

1. 定义Deployment来创建Pod和ReplicaSet,使用Deployment来创建ReplicaSet。ReplicaSet在后台创建pod。检查启动状态，看它是成功还是失败。
2. 然后，通过更新Deployment的PodTemplateSpec字段来声明Pod的新状态。这会创建一个新的ReplicaSet，Deployment会按照控制的速率将pod从旧的ReplicaSet移动到新d的ReplicaSet中。
3. 如果当前状态不稳定，回滚到之前的Deployment revision。每次回滚都会更新Deployment的revision。
4. 扩容Deployment以满足更高的负载。
5. 暂停Deployment来应用PodTemplateSpec的多个修复，然后恢复上线。
6. 根据Deployment 的状态判断上线是否hang住了。
7. 清除旧的不必要的 ReplicaSet。

### 创建
yml文件实例(deployment_nginx.yml)

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.12.2
        ports:
        - containerPort: 80
```

执行:
```bash
kubectl create -f deployment_nginx.yml
```

![avatar](/img/in-post/2019-03-08/Snip20190309_10.png)

查看deployment信息

```bash
kubectl get deployment
kubectl get deployment -o wide
```

![avatar](/img/in-post/2019-03-08/Snip20190309_11.png)

### update nginx to 1.13

```bash
kubectl set image
   '<resource> <name>'
   '<resource>'
```

执行
```bash
kubectl set image deployment nginx-deployment nginx=nginx:1.13
```

![avatar](/img/in-post/2019-03-08/Snip20190309_12.png)

### rollout

查看history
```bash
kubectl rollout history deployment nginx-deployment
```
回退版本
```bash
kubectl rollout undo deployment nginx-deployment
```

![avatar](/img/in-post/2019-03-08/Snip20190309_13.png)

### expose port


```bash 
kubectl expose deployment nginx-deployment --type=NodePort
kubectl get svc
```

![avatar](/img/in-post/2019-03-08/Snip20190309_14.png)

