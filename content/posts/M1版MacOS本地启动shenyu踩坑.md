---
title: "M1版MacOS本地启动shenyu踩坑"
date: Thu Nov 25 22:22:25 CST 2021
categories: ["中间件"]
tags: ["中间件"]
draft: false
---

`mvn clean install`:

1. `RedisRateLimiterScriptsTest.java`测试不通过
原因：内嵌的RedisServer启动失败，不支持m1
embedded-redis社区issue：https://github.com/kstyrc/embedded-redis/issues/127
2. protobuf-maven-plugin执行失败，原因grpc不支持m1
解决方法如下：
```
// 指定系统识别号，或者在settings.xml里面改（推荐）
// 见：https://github.com/grpc/grpc-java/issues/7690
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:osx-x86_64</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:osx-x86_64</pluginArtifact>
                    <protoSourceRoot>src/main/resources/proto</protoSourceRoot>
                </configuration>
```

本地运行集成测试

1. TODO: docker打包后是amd64架构的镜像，无法启动。
