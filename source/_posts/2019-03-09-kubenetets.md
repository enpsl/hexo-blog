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
minikube start      创建k8s集群

```
