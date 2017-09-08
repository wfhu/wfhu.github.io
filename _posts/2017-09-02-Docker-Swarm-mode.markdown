---
layout: post
title:  "使用全套开源工具搭建生成环境可用的Docker Swarm mode集群"
date:   2017-09-02 14:32:34 +0800
categories: docker
---

## 使用开源工具，从零开始搭建完整生态的swarm mode生产环境集群
相关代码、模版在github: <https://github.com/wfhu/docker-swarm-full-stack>




## 本文档不包含说明的一些配套工具，这些需要各位去查看另外的文档
* 一套可用的ELK工具，用于接收和展示搜集的日志
* 一个可用的docker镜像仓库
* 一台或多台基于CentOS7.3的服务器，root权限
* 可用的NFS服务，用于存储的持久化
* Prometheus服务，用于接收和存储监控数据
* Grafana展示前端，用于展示监控数据
* CI/CD：SVN + Jenkins，用于自动打包镜像和测试

## 整套生产环境主要使用以下工具：
* 集群管理和编排：Docker Swarm Mode   
* 容器监控：cAdvisor + prometheus    
* 节点监控：node_exporter + prometheus  
* 监控展示：Grafana    
* 前端UI界面：portainer  
* 日志搜集展示和搜索：ELK+logspout  
* 持久化存储：NFS+NAS  
* 外部负载均衡器：阿里云SLB



## 首先，我们先搭建集群  

### 1. install the docker-ce tools

```
yum install -y yum-utils device-mapper-persistent-data lvm2

yum-config-manager --enable extras

yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum makecache fast

yum install -y docker-ce

systemctl start docker

systemctl enable docker

```



#### 确认docker是否工作正常
```
docker run hello-world

```



### 2. setup the swarm manager leader node
```
docker  swarm init --advertise-addr 192.168.33.5
```



### 3. add two manager node, join this swarm as manager node(also as worker node)
```
docker swarm join --token SWMTKN-1-YOUR-MANAGER-TOKEN 192.168.33.5:2377
```



### 4. add two worker node
```
docker swarm join --token SWMTKN-1-YOUR-WORKER-TOKEN 192.168.33.5:2377

```

### 备注：如忘了上面的TOKEN，可以通过在Manager节点上运行以下命令来获取：
#### 获取worker token
```
docker swarm join-token worker
```

#### 获取manager token
```
docker swarm join-token manager
```









## 第二步，我们需要把监控的agent进程起来 

### 1. use cAdvisor to monitor container's CPU/Memory/Network    
```
docker service create --name cadvisor \
    --mount type=bind,source=/var/lib/docker/,destination=/var/lib/docker,readonly \
    --mount type=bind,source=/var/run,destination=/var/run \
    --mount type=bind,source=/sys,destination=/sys,readonly \
    --mount type=bind,source=/,destination=/rootfs,readonly \
    --mode global \
    --detach=true \
    --publish mode=host,published=18080,target=8080 \
    google/cadvisor:latest
```


### 2. use prometheus's node_exporter to monitor Swarm node's basic infomation     
```
docker service create --name node_exporter \
    --mount type=bind,source=/proc,destination=/host/proc,readonly \
    --mount type=bind,source=/sys,destination=/host/sys,readonly \
    --mount type=bind,source=/,destination=/rootfs,readonly \
    --mode global \
    --detach=true \
    --publish mode=host,published=9100,target=9100 \
    quay.io/prometheus/node-exporter  \
    -collector.procfs /host/proc \
    -collector.sysfs /host/sys \
    -collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```

### 3. configure the prometheus server, add the above newly added targets    
```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.

  - job_name: 'MySwarmCluster'
    static_configs:
      - targets: ['192.168.24.160:18080','192.168.24.160:9100'] 
```


到这里，你就可以用Grafana来查看Swarm集群和运行的服务的信息了

![grafana Docker Swarm Dashboard](/assets/grafana_template_screenshot.jpg)



## 第三步，安装管理UI工具portainer

### 默认docker-engine只监听UNIX Socket的，先把docker-engine配置修改，新增TCP监听，重启docker-engine
```
sed -i "/^ExecStart/c ExecStart=/usr/bin/dockerd -H unix:///var/run/docker.sock -H tcp://$(ip a |grep global |grep eth0 |awk '{print $2}' |cut -d'/' -f1):2375" /usr/lib/systemd/system/docker.service
grep '^ExecStart' /usr/lib/systemd/system/docker.service
systemctl daemon-reload
systemctl restart docker
```

### 在每个host主机上创建相应的目录，然后启动portainer服务
```
mkdir -p /data/portainer_prod

docker service create \
    --name portainer \
    --publish 9000:9000 \
    --constraint 'node.role == manager' \
    --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
    --mount type=bind,src=/data/portainer_prod,dst=/data \
    portainer/portainer \
    -H unix:///var/run/docker.sock
```

Then, you can visit http://your-ip-address:9000 to visit the portainer UI



## 第四步，搜集stdout/stderr日志，传送到ElasticSearch中

日志流向如下 : *container stdout/stderr* -> *host主机上的logspout服务* -> *集群内部的logstash服务* -> *外部的ElasticSearch*

**由于logspout需要访问内部的logstash服务，我们这里利用自定义overlay网络的DNS功能，这样logspout只要指定访问logstash服务的服务名就可以顺利地访问到该服务了，logstash可以随意在集群内漂移**

### 1. 创建user-defined overlay网络*lognet*，专门给logspout和logstash互相沟通使用
```
docker network create --driver overlay lognet
```

### 2. 创建我的logstash服务：mylogstash，监听TCP/19300端口
```
docker service create \
    --name mylogstash \
    --network lognet \
    --publish 19300:19300 \
    logstash -e 'input { tcp { port => 19300 mode => "server" ssl_enable => false } } output { elasticsearch { hosts => ["YOUR-ElasticSearch-ADDRESS:PORT"] index => "my-docker-cluster"} }'
```

### 3. 创建logspout服务

#### 由于logspout默认不支持logstash，需要自己打包进去logstash plugin，代码可以参考[mylogsptou](https://github.com/wfhu/docker-swarm-full-stack)
```
cd mylogspout && docker build -t mylogspout:v1 .
```
如果打包镜像时有报错，注意mylogspout/build.sh要有执行权限, 参考[here](https://github.com/gliderlabs/logspout/issues/238)

#### tag/push你的镜像到你的镜像仓库中
```
# docker login -u YOUR-USER-NAME -p YOUR-PASSWORD  YOUR-REGISTRY-ADDRESS
# docker tag mylogspout:v1 YOUR-REGISTRY-ADDRESS/mylogspout:v1
# docker push YOUR-REGISTRY-ADDRESS/mylogspout:v1
```

#### now you can create your *logsplout* service based on the new image
```
docker service create \
    --name mylogspout \
    --network lognet \
    --with-registry-auth \
    --mode global \
    --detach=true \
    --mount type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock,readonly \
    -e ROUTE_URIS=logstash+tcp://mylogstash:19300 \
    YOUR-REGISTRY-ADDRESS/mylogspout:v1 
```
**注意：ROURTE_URIS里面直接写_mylogstash_就可以访问到logstash服务了**


*如果你的其他container的日志都是直接输出到stdout/stderr, 你最好限制一下日志文件的大小 *[max-size](https://docs.docker.com/engine/admin/logging/json-file/)* 默认的日志驱动是JSON File log driver*



## 存在的问题：
* Swarm Mode本身缺少autoscaling的能力，暂时需要手动操作来伸缩
* portainer UI缺少资源限制(CPU/memory)的能力，如果有这方面的需求，只能使用命令行



