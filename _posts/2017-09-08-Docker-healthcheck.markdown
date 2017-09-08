---
layout: post
title:  "Docker HEALTHCHECK健康检查及注意事项"
date:   2017-09-08 18:00:25 +0800
categories: docker
---

# Docker HEALTHCHECK健康检查及注意事项


HEALTHCHECK 指令告诉 Docker 应该如何进行判断容器的状态是否正常，这是 Docker 1.12 引入的新指令。

在没有 HEALTHCHECK 指令前，Docker 引擎只可以通过容器内主进程是否退出来判断容器是否状态异常。如果程序进入死锁状态，或者死循环状态，应用进程并不退出，但是该容器已经无法提供服务了。在 1.12 以前，Docker 不会检测到容器的这种状态，从而不会重新调度，导致可能会有部分容器已经无法提供服务了却还在接受用户请求。


### 1. 编写一个简单的web服务并开启健康检查：
```
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html

CMD ["nginx", "-g", "daemon off;"]

HEALTHCHECK CMD wget -O /dev/null http://localhost:80/

```


### 2. 打包镜像并启动后，可以通过docker ps查看容器最初状态，此时正在开始健康检查(health: starting)：
```
[root@VM_33_14_centos ~]# docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED             STATUS                            PORTS                                          NAMES
1cafd6fae2cf        registry.cn-shanghai.aliyuncs.com/boyaa-test/opsdemo_opsweb:20170908_1753_209   "nginx -g 'daemon ..."   3 seconds ago       Up 3 seconds (health: starting)   80/tcp
```


### 3. 过几秒中之后，再通过docker ps查看容器状态，已经变成healthy了：
```
[root@VM_33_14_centos ~]# docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED              STATUS                        PORTS                                          NAMES
1cafd6fae2cf        registry.cn-shanghai.aliyuncs.com/boyaa-test/opsdemo_opsweb:20170908_1753_209   "nginx -g 'daemon ..."   About a minute ago   Up About a minute (healthy)   80/tcp 
```







## 需要特别注意，监控检查的命令，必须确保每次执行结果都保持一致
如果将上面的检查命令修改一下：
```
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html

CMD ["nginx", "-g", "daemon off;"]

HEALTHCHECK CMD wget http://localhost:80/
```
我们去掉了*-O /dev/nul*，会发生什么呢？


### 1. 启动容器后，状态是开始健康检查(health: starting)：
```
[root@VM_33_14_centos ~]# docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED             STATUS                                     PORTS                                          NAMES
5aab61378e58        registry.cn-shanghai.aliyuncs.com/boyaa-test/opsdemo_opsweb:20170908_1804_210   "nginx -g 'daemon ..."   3 seconds ago       Up Less than a second (health: starting)   80/tcp
```


### 2. 过几秒钟，容器状态健康(healthy)：
```
[root@VM_33_14_centos ~]# docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED              STATUS                        PORTS                                          NAMES
5aab61378e58        registry.cn-shanghai.aliyuncs.com/boyaa-test/opsdemo_opsweb:20170908_1804_210   "nginx -g 'daemon ..."   About a minute ago   Up About a minute (healthy)   80/tcp 
```


### 3. 过两分钟后，容器状态变成不健康(unhealthy)：
```
[root@VM_33_14_centos ~]# docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED             STATUS                     PORTS                                          NAMES
5aab61378e58        registry.cn-shanghai.aliyuncs.com/boyaa-test/opsdemo_opsweb:20170908_1804_210   "nginx -g 'daemon ..."   2 minutes ago       Up 2 minutes (unhealthy)   80/tcp 
```

#### 随后容器被重新调度，**注意CONTAINER ID的变化**：
```
[root@VM_33_5_centos ~]# docker ps
CONTAINER ID        IMAGE                                                                           COMMAND                  CREATED              STATUS                        PORTS                     NAMES
cb1da285ac16        registry.cn-shanghai.aliyuncs.com/boyaa-test/opsdemo_opsweb:20170908_1804_210   "nginx -g 'daemon ..."   About a minute ago   Up About a minute (healthy)   80/tcp                    opsweb.1.i4u54q7cdm7pz6f2x2fmufrs6
```


#### 原因是wget第一次执行结果和接下来继续运行的**结果不一致**：
健康检查命令第一次运行后，本地保存了index.html文件，导致后续所有的检查命令全部失败
```
/ # wget http://localhost:80
Connecting to localhost:80 (127.0.0.1:80)
wget: can't open 'index.html': File exists
```





