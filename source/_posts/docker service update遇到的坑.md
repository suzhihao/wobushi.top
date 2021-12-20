---
title: docker service update遇到的坑
date: 2020-12-21 22:59:55
categories:
  - [后端, docker]
tags:
  - docker
---

### 场景复盘

某天在发布项目的时候，发现服务启动后业务无法访问。检查日志也没有发现错误，检查服务状态发现服务也正常启动没有退出。
初步怀疑 redis 配置有问题，但经过大量时间排查后发现 redis 配置没有问题，后来又怀疑是某个 npm 包更新导致，或者是因为 nodejs 升级导致。
然后又去各种服务器上进行测试，发现运行都没有问题。而且服务在测试环境能够正常运行，但在预发布就无法运行。

经过大量时间的排查，最终将问题定位到了 docker 中，在排查 CI 文件的时候发现，由于之前做服务迁移需要，服务启动的端口由原本的 9000 改到了 9004 端口，但是 docker 宿主机端口和容器端口的映射关系并没有更改，于是就产生服务能够正常启动但访问不到的情况。

### 原因梳理

目前使用 docker 容器的时候，有两种启动容器的方法，一种是集群模式，而另一种是非集群模式。

测试采用的非集群模式，在 CI 中配置的启动脚本如下：

```bash
- EXIST_CONTAINER_ID=`docker ps -af name=xxx --format "{{.ID}}"`
- if [ -n "$EXIST_CONTAINER_ID" ] ; then docker rm -f $EXIST_CONTAINER_ID; fi
- docker run -d --name xxx -p 60001:9000 xxx
```

新容器启动前会先将旧的容器删除，然后再起一个全新的服务。

而预发布采用集群模式：

```bash
- EXIST_SERVVICE_ID=`docker service ls -f name=xxx -q`
- if [ -n "$EXIST_SERVVICE_ID" ] ; then docker service update --image xxx --replicas 8 xxx --detach=false; else docker service create --name xxx --detach=false -p 60001:9000 --replicas 8 xxx; fi
```

脚本先检查服务是否存在，如果存在直接更新镜像，如果不存在才会新起一个服务。然而只有在容器创建的时候才会指定端口映射关系，更新的时候并不会重置，所以导致服务端口变化的时候，宿主机端口还映射老的端口，导致服务无法正常访问。

另外经查阅似乎没有很好的办法可以动态修改已经运行中的容器端口映射，目前只能删掉原有的服务然后用新镜像重新创建一个容器。[参考资料](https://stackoverflow.com/questions/19335444/how-do-i-assign-a-port-mapping-to-an-existing-docker-container)
