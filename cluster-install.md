#### 目录
* 集群安装
* 集群测试

> 
1， systemd测试
2， fleet测试

#### 集群安装

按照[前一篇](https://github.com/duanbing/coreos-install/blob/master/install.md)的安装教程，但是需要注意修改cloud-config.yaml.
由于etcd至少需要三个peer才能搭建 一个集群，因此，我这里有三个配置：

coreos1： 
```
#cloud-config

hostname: coreos1

coreos:
  etcd:
    addr: 192.168.1.120:4001
    peer-addr: 192.168.1.120:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: static.network
      content: |
        [Match]
        Name=enp0s8

        [Network]
        Address=192.168.1.120/24
        Gateway=192.168.1.254
        DNS=8.8.8.8
        DNS=8.8.4.4
users:
      ... (见单机安装)
```
coreos2：
```
#cloud-config

hostname: coreos2

coreos:
  etcd:
    peers: 192.168.1.120:7001
    addr: 192.168.1.121:4001
    peer-addr: 192.168.1.121:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: static.network
      content: |
        [Match]
        Name=enp0s8

        [Network]
        Address=192.168.1.121/24
        Gateway=192.168.1.254
        DNS=8.8.8.8
        DNS=8.8.4.4
users:
      ... (见单机安装)
```
coreos3:
```
#cloud-config

hostname: coreos3

coreos:
  etcd:
    peers: 192.168.1.120:7001
    addr: 192.168.1.122:4001
    peer-addr: 192.168.1.122:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
    - name: static.network
      content: |
        [Match]
        Name=enp0s8

        [Network]
        Address=192.168.1.122/24
        Gateway=192.168.1.254
        DNS=8.8.8.8
        DNS=8.8.4.4
users:
      ... (见单机安装)
```
安装完成后，在集群中的任何一个机器上,执行，得到如下结果
```
core@coreos1 ~ $ fleetctl list-machines 
MACHINE	 IP	 METADATA 
5c96b86d...	192.168.1.122	- 
8de0a52e...	192.168.1.121	- 
d4937498...	192.168.1.120	-
```
#### 集群测试

##### systemd测试

测试需要使用到fleet，暂时没有做专门的介绍，大家可以先看看[这里](),暂时熟悉下，后面会专门介绍。
这是一个简单的unit文件（可以理解为服务描叙文件），保存为myapp.service
```
[Unit]
Description=MyApp
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill busybox1
ExecStartPre=-/usr/bin/docker rm busybox1
ExecStartPre=/usr/bin/docker pull busybox
ExecStart=/usr/bin/docker run --name busybox1 busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"

[Install]
WantedBy=multi-user.target
```
然后执行保存在 `/etc/systemd/system/myapp.service`，并执行
```
core@coreos2 ~ $ sudo systemctl enable /etc/systemd/system/myapp.service 
Created symlink from /etc/systemd/system/multi-user.target.wants/myapp.service to /etc/systemd/system/myapp.service. 
core@coreos2 ~ $ sudo systemctl start myapp.service

```
这个步骤会执行的很慢， 因为docker回去`https://index.docker.io/v1/repositories/busybox/images` 下载镜像，busybox虽然只有5M多，但是上米国网站，就是很慢。执行完， 应该什么什么没有。如下验证：
```
core@coreos2 ~ $ journalctl -f -u myapp.service 
-- Logs begin at Mon 2014-12-01 14:43:51 UTC. -- 
Dec 01 16:25:05 coreos2 docker[725]: Hello World 
Dec 01 16:25:06 coreos2 docker[725]: Hello World 
Dec 01 16:25:07 coreos2 docker[725]: Hello World 
Dec 01 16:25:08 coreos2 docker[725]: Hello World 
Dec 01 16:25:09 coreos2 docker[725]: Hello World 
Dec 01 16:25:10 coreos2 docker[725]: Hello World 
Dec 01 16:25:11 coreos2 docker[725]: Hello World 
Dec 01 16:25:12 coreos2 docker[725]: Hello World 
Dec 01 16:25:13 coreos2 docker[725]: Hello World 
...
```
如果不幸，显示
```
core@coreos1 ~ $ sudo systemctl start myapp.service 
Job for myapp.service failed. See 'systemctl status myapp.service' and 'journalctl -xn' for details. 
core@coreos1 ~ $ sudo journalctl -xn 
-- Logs begin at Sun 2014-11-30 17:14:31 UTC, end at Mon 2014-12-01 16:18:03 UTC. -- 
Dec 01 16:17:15 coreos1 docker[895]: Pulling repository busybox 
Dec 01 16:17:16 coreos1 ntpd[451]: Listen normally on 6 docker0 172.17.42.1 UDP 123 
Dec 01 16:17:16 coreos1 ntpd[451]: peers refreshed 
Dec 01 16:17:56 coreos1 docker[823]: Get https://index.docker.io/v1/repositories/busybox/images: dial tcp: lookup index.docker.io: con 
Dec 01 16:17:56 coreos1 docker[823]: [66f49073] -job pull(busybox, ) = ERR (1) 
Dec 01 16:17:56 coreos1 systemd[1]: myapp.service: control process exited, code=exited status=1 
Dec 01 16:17:56 coreos1 systemd[1]: Failed to start MyApp. 
-- Subject: Unit myapp.service has failed 
-- Defined-By: systemd 
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel 
-- 
-- Unit myapp.service has failed. 
-- 
-- The result is failed. 
Dec 01 16:17:56 coreos1 systemd[1]: Unit myapp.service entered failed state. 
Dec 01 16:17:56 coreos1 docker[895]: 2014/12/01 16:17:56 Get https://index.docker.io/v1/repositories/busybox/images: dial tcp: lookup 
Dec 01 16:18:03 coreos1 sudo[908]: core : TTY=pts/0 ; PWD=/home/core ; USER=root ; COMMAND=/bin/journalctl -xn
```
也别慌，首先验证下，能否`ping index.docker.io`, 如果不能，那就换一个能ping通的coreos host测试。

##### fleet测试

上面是测试了systemdctl来执行unit，下面来个fleet执行的例子,有什么区别？systemctl是单机上的[systemd manager](https://wiki.archlinux.org/index.php/Systemd)，fleet是集群上的systemd manager。
先保存myapp.fleet.service
```
[Unit]
Description=MyApp
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill busybox1
ExecStartPre=-/usr/bin/docker rm busybox1
ExecStartPre=/usr/bin/docker pull busybox
ExecStart=/usr/bin/docker run --name busybox1 busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
ExecStop=/usr/bin/docker stop busybox1
```
然后执行
```
core@coreos2 ~ $ fleetctl start myapp.fleet.service 
Unit myapp.fleet.service launched on 5c96b86d.../192.168.1.122  ## 注意我是在192.168.1.121这个coreos host上执行的
core@coreos2 ~ $ fleetctl list-units 
UNIT	 MACHINE	 ACTIVE	SUB 
myapp.fleet.service	5c96b86d.../192.168.1.122	active	running

core@coreos1 ~ $ fleetctl load myapp.fleet.service 
Unit myapp.fleet.service loaded on 5c96b86d.../192.168.1.122 
core@coreos1 ~ $ fleetctl list-unit-files 
UNIT	 HASH	DSTATE	STATE	TARGET 
myapp.fleet.service	391d247	loaded	loaded	5c96b86d.../192.168.1.122
```
上面看到，我们可以在任意的coreos host上面提交unit，但是如果要执行 fleet status和 fleet ssh的话， 就需要到跳板机上面来执行。下面我们在跳板机上安装好git,golang，并且clone下来fleet（`git clone https://github.com/coreos/fleet.git`）, 编译fleet，并且测试。
```
ccore@ubuntu:~/fleet$ ./build 
Building fleetd... 
Building fleetctl...
core@ubuntu:~/fleet$ ./bin/fleetctl --endpoint http://192.168.1.120:4001 list-units 
UNIT	 MACHINE	 ACTIVE	 SUB 
myapp.fleet.service	5c96b86d.../192.168.1.122	active	running

```
然后添加ssh授权
```
 eval `ssh-agent`
 ssh-add ~/.ssh/id_rsa
 cp ~/.ssh/known_hosts ~/.fleetctl/
```
不出意外，你应该就可以看到一大串字符串dump出来
```
core@ubuntu:~$ ./fleet/bin/fleetctl --endpoint http://192.168.1.120:4001 status myapp.fleet.service 
2014/12/02 01:15:50 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:1 - hashed hosts not implemented 
2014/12/02 01:15:50 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:2 - hashed hosts not implemented 
2014/12/02 01:15:50 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:3 - hashed hosts not implemented 
2014/12/02 01:15:50 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:4 - hashed hosts not implemented 
2014/12/02 01:15:50 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:5 - hashed hosts not implemented 
2014/12/02 01:15:50 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:6 - hashed hosts not implemented 
2014/12/02 01:15:50 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:7 - hashed hosts not implemented 
The authenticity of host '192.168.1.122' can't be established. 
RSA key fingerprint is d9:1a:7d:61:a9:12:1c:37:ca:aa:24:6b:d6:62:3c:08. 
Are you sure you want to continue connecting (yes/no)? yes 
Warning: Permanently added '192.168.1.122' (RSA) to the list of known hosts. 
2014/12/02 01:15:55 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:1 - hashed hosts not implemented 
2014/12/02 01:15:55 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:2 - hashed hosts not implemented 
2014/12/02 01:15:55 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:3 - hashed hosts not implemented 
2014/12/02 01:15:55 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:4 - hashed hosts not implemented 
2014/12/02 01:15:55 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:5 - hashed hosts not implemented 
2014/12/02 01:15:55 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:6 - hashed hosts not implemented 
2014/12/02 01:15:55 WARN known_hosts.go:225: /home/core/.fleetctl/known_hosts:7 - hashed hosts not implemented 
● myapp.fleet.service - MyApp 
Loaded: loaded (/run/fleet/units/myapp.fleet.service; linked-runtime) 
Active: active (running) since Mon 2014-12-01 17:15:15 UTC; 40s ago 
Process: 1604 ExecStartPre=/usr/bin/docker pull busybox (code=exited, status=0/SUCCESS) 
Process: 1594 ExecStartPre=/usr/bin/docker rm busybox1 (code=exited, status=0/SUCCESS) 
Process: 1580 ExecStartPre=/usr/bin/docker kill busybox1 (code=exited, status=0/SUCCESS) 
Main PID: 1620 (docker) 
CGroup: /system.slice/myapp.fleet.service 
└─1620 /usr/bin/docker run --name busybox1 busybox /bin/sh -c while true; do echo Hello World; sleep 1; done 

Dec 01 17:15:46 coreos3 docker[1620]: Hello World 
Dec 01 17:15:47 coreos3 docker[1620]: Hello World 
Dec 01 17:15:48 coreos3 docker[1620]: Hello World 
Dec 01 17:15:49 coreos3 docker[1620]: Hello World 

```

恭喜你， 测试成功了！！！！

当然，要到这一步非常不容易。 中间碰到很多问题暂时不一一记录了，主要是ssh授权的问题。大家可以加Q群：412065906，一起交流讨论。 

接下来， 我会将coreos的各个组件一一给大家介绍。 




