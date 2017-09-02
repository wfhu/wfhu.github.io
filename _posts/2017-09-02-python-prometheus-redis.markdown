---
layout: post
title:  "python编写Prometheus exporter获取redis自定义数据"
date:   2017-09-02 18:37:20 +0800
categories: python Prometheus redis
---

近期有使用Prometheus监控我们的redis，使用了第三方的[redis_exportor](https://github.com/oliver006/redis_exporter)，但是它无法获取*redis实例的maxmemory*，我就通过自己编写python脚本自己来获取这个数据了


```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
from prometheus_client import start_http_server
from prometheus_client import Gauge
import redis
import random
import time

fp=open("/export/scripts/sa/redisList.txt", "r")

g = Gauge('redis_maxmemory', 'Maxmemory of redis instance', ['addr', 'alias'])

if __name__ == '__main__':
    # Start up the server to expose the metrics.
    start_http_server(8000)
    # Generate some requests.
    while True:
        for line in fp:
            RdHost,RdPort,RdWarn,RdName,RdContact = line.split()
            #print RdHost
            r=redis.Redis(host=RdHost, port=RdPort, db=0, password='YOUR REDIS PASSWORD')
            maxmemory=r.config_get("maxmemory")
            #print maxmemory
            g.labels(addr="%s:%s" % (RdHost,RdPort),alias=u"%s" % RdName.decode('utf-8')).set(float(maxmemory['maxmemory']))

```


```bash
python readRedis.py
```

然后修改Prometheus.yml，添加上述IP:port到target里面，通过Grafana模板，可以查看到相关信息了


![new](/assets/grafana_redis.png)


监控redis的grafana模版可供 [下载](/assets/grafana_redis_template.json)








