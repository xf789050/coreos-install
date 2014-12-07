首先解释下什么叫服务发现。一个高可用的服务理论上都会有多个不同的实例分布在不同的idc，机器上。因此，启动或者停止一个新的实例的时候，我们希望我们的命名空间或者后端可以自动在负载均衡服务中注册或者解除这个实例。这就是服务发现，负载均衡服务自动发现提供服务的实例。


目前有很多工具可以实现这个需求(例如zk，etcd)，这些工具的普遍原则就是一个服务启动的时候，必须手动注册进入配置管理服务器，停止的时候，也必须从其中去除。注册之后，其他服务可以在配置管理服务器可以搜索搭配这些新增服务的实例列表。

随着docker的火爆，一些docker配套的服务发现软件也开始出现了。例如[synapse](https://github.com/airbnb/synapse),并在此推荐两篇使用介绍的文章：
* [《服务发现与 Docker》](http://www.tuicool.com/articles/J3MRjm) 
* [《Docker 与 服务发现 - 2》](http://www.tuicool.com/articles/6v2iMnA)
* [《使用 Etcd 和 Haproxy 做 Docker 服务发现》](http://segmentfault.com/blog/yexiaobai/1190000000730186)。

那coreos如何来做服务发现呢？

在coreos中使用了一种sidekick的模式来做服务发现。

sidekick模式很像 Synapse，使用一个隔离的的服务（进程）代理来监控提供服务的实例。
首先假设我们在2个机器上跑一个apache服务，apache@1.service和apache@2.service
```
[Unit]
Description=My Apache Frontend
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill apache1
ExecStartPre=-/usr/bin/docker rm apache1
ExecStartPre=/usr/bin/docker pull coreos/apache
ExecStart=/usr/bin/docker run -rm --name apache1 -p 80:80 coreos/apache /usr/sbin/apache2ctl -D FOREGROUND
ExecStop=/usr/bin/docker stop apache1

[X-Fleet]
Conflicts=apache@*.service
```
其中最后的`Conflicts=apache@*.service`表示不同的apache@*.service为了提供服务的高可用性会分布在不同的机器上。可以看到启动的apache服务会对外暴露80端口。
同时，我们做一个监控service来监控2个节点的健康。
```
[Unit]
Description=Announce Apache1
BindsTo=apache@%i.service
After=apache@%i.service

[Service]
ExecStart=/bin/sh -c "while true; do etcdctl set /services/website/apache@%i '{ \"host\": \"%H\", \"port\": 80, \"version\": \"52c7248a14\" }' --ttl 60;sleep 45;done"
ExecStop=/usr/bin/etcdctl rm /services/website/apache@%i

[X-Fleet]
MachineOf=apache@%i.service
```
里面使用到的MachineOf来保证apache@%i.service可以和apache-discovery@%i.service`跑在同一个机器上，因此，apache-discovery@.service会将所在机器上的80端口和版本信息，并且每隔45s更新一下etcd中的`/services/website/apache@%i`，设置ttl超时为60s,如果发现机器挂掉了， 就会在最晚60s后发现对应的key失效不存在，那么将对应的服务信息从etcd中删除。
```
$ etcdctl ls /services/ --recursive
/services/website
/services/website/apache@1
/services/website/apache@2
$ etcdctl get /services/website/apache@1
{ "host": "coreos2", "port": 80, "version": "52c7248a14" }
```
其实，针对单个机器，我们看出， coreos给出的实例非常的简陋，但是非常容易实现。





