---
title: "Apache ShenYu构建多平台Docker镜像解决方案"
date: 2022-04-02T00:23:50+08:00
author: JooKS
authorLink: https://github.com/JooKS-me
tags: ["云原生","ShenYu"]
categories: ["云原生"]
draft: false
---

确保docker版本大于等于19.03，并且实验性功能已经开启（即docker配置文件中 `experimental` 参数为 `true`。

输入 `docker buildx ls` 可查看 builder 列表

1. 新建一个builder实例（不能使用default）

   ```shell
   docker buildx create --name shenyu
   docker buildx use shenyu
   ```

2. 登陆docker账号

   ```shell
   docker login
   ```

3. 依次运行以下命令，构建镜像并推送到docker hub（注意将 `${PUBLISH.VERSION}` 替换为发布的版本号）

   ```shell
   git checkout v${PUBLISH.VERSION}
   cd ~/shenyu/shenyu-dist/
   mvn clean package -Prelease
   
   docker buildx build \ 
     -t apache/shenyu-admin:latest \ 
     -t apache/shenyu-admin:${PUBLISH.VERSION} \ 
     --build-arg APP_NAME=apache-shenyu-incubating-${PUBLISH.VERSION}-admin-bin \ 
     --platform=linux/arm64,linux/amd64,linux/arm/v6,linux/arm/v7,linux/386,linux/ppc64le,linux/s390x \ 
     -f ./shenyu-admin-dist/Dockerfile --push
   
   docker buildx build \ 
     -t apache/shenyu-bootstrap:latest \ 
     -t apache/shenyu-bootstrap:${PUBLISH.VERSION} \ 
     --build-arg APP_NAME=apache-shenyu-incubating-${PUBLISH.VERSION}-bootstrap-bin \ 
     --platform=linux/arm64,linux/amd64,linux/arm/v6,linux/arm/v7,linux/386,linux/ppc64le,linux/s390x \ 
     -f ./shenyu-bootstrap-dist/Dockerfile --push
   ```

