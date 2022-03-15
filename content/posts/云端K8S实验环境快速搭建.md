---
title: "云端K8S实验环境快速搭建"
date: 2022-03-16T00:23:50+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","Kubernetes"]
categories: ["云原生"]
draft: false
---

>这是利用腾讯云按量付费主机和自制镜像，快速创建k8s和kubevela实验环境，留给自己看的操作记录
>
>镜像中已经包含go、kubectl、docker
>
>已经配置好sshd_config
>
>一般来说创建实例的时候拉到4h8g + 100M带宽 + 香港节点，可以用的非常爽 : )
>
>实验做完就销毁，美滋滋

## 创建root用户

```shell
sudo passwd root //会让你输入当前用户密码。输入按下回车输入两次root密码

su root //提示输入root密码。输入即可
```

## 设置环境变量

```shell
export GOROOT=/usr/local/share/go
export GOPATH=/opt/go-project

export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

## 安装kind

```shell
go install sigs.k8s.io/kind@v0.11.1
```

## 创建kind集群

```shell
cat <<EOF | kind create cluster --image=kindest/node:v1.21.1 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
```

## 安装ingress-nginx

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```

## 安装helm

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## 安装kubevela

https://kubevela.io/zh/docs/install

## 安装可观测性插件

https://kubevela.io/zh/docs/platform-engineers/system-operation/observability

## 安装可观测性插件后

```
vela port-forward addon-observability -n vela-system 3000:80 --address 0.0.0.0
```

然后浏览器就可以在3000端口访问grafana（公网）

账号：admin

密码：

```shell
kubectl get secret grafana -o jsonpath="{.data.admin-password}" -n vela-system | base64 --decode ; echo
```



