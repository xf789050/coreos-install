#### 单机fleet： systemd

传统的sysvinit使用[inittab](http://www.2cto.com/os/201108/98426.html)来控制init完成之后，执行哪些shell脚本。大量的shell脚本执行，效率低下， 并且无法并行执行。因此产出了systemd。systemd 是linux操作系统下面系统和服务的init管理软件,当boot过程启动第一个进程（PID=1）的时候，systemd启动并且维护用户空间的服务。旨在尽可能启动更少进程；尽可能将更多进程并行启动，“其开发目标是提供更优秀的框架以表示系统服务间的依赖关系，并依此实现系统初始化时服务的并行启动，同时达到降低Shell的系统开销的效果，最终代替现在常用的System V与BSD风格init程序”(摘自http://zh.wikipedia.org/wiki/Systemd)。

systemd的特性有：
* 支持并行化任务
* 同时采用socket式与D-Bus总线式激活服务；
* 按需启动守护进程（daemon）；
* 利用 Linux 的 cgroups 监视进程；
* 支持快照和系统恢复；
* 维护挂载点和自动挂载点；
* 各服务间基于依赖关系进行精密控制。

有关使用systemd的文档： [http://www.freedesktop.org/software/systemd/man/](http://www.freedesktop.org/software/systemd/man/)

举个例子来说明他的功能和使用方法。测试环境当然选择安装好的coreos host。
```
core@coreos1 ~ $ systemctl cat myapp.service 
# /etc/systemd/system/myapp.service 
[Unit]   #基本配置信息和依赖关系
Description=MyApp 
After=docker.service   ##当前单元必须在docker.service启动之后才启动
Requires=docker.service  ## 表示强依赖，如果是Wants=docker.service，则表示依赖关系可选。如果After=，则2个单元同时启动

[Service]  # 服务特定的配置信息
TimeoutStartSec=0   # 服务启动限时，0表示不限制， 如果超时，服务启动失败，并且被关闭。
ExecStartPre=-/usr/bin/docker kill busybox1    #ExecStart之前执行的命令，如果加=后面带-，表示执行失败，其他的命令继续执行，多个同样的命令按照先后顺序执行
ExecStartPre=-/usr/bin/docker rm busybox1 
ExecStartPre=/usr/bin/docker pull busybox 
ExecStart=/usr/bin/docker run --name busybox1 busybox /bin/sh -c "while true; do  #服务启动之后，执行的命令

[Install] 
WantedBy=multi-user.target  #
```
myapp.service依赖于docker.service。再看看docker.service
```
core@coreos1 ~ $ systemctl cat docker.service 
# /usr/lib64/systemd/system/docker.service 
[Unit] 
Description=Docker Application Container Engine 
Documentation=http://docs.docker.io 
Requires=docker.socket 

[Service] 
Environment="TMPDIR=/var/tmp/"    #添加环境变量
ExecStartPre=/bin/mount --make-rprivate / 
LimitNOFILE=1048576 
LimitNPROC=1048576 
# Run docker but don't have docker automatically restart 
# containers. This is a job for systemd and unit files. 
ExecStart=/usr/bin/docker --daemon --storage-driver=btrfs --host=fd:// 

[Install] 
WantedBy=multi-user.target  #在multi-user.target下面启动

core@coreos1 ~ $ systemctl cat docker.socket 
# /usr/lib64/systemd/system/docker.socket 
[Unit] 
Description=Docker Socket for the API 

[Socket] 
SocketMode=0660  #socket文件的file mode
SocketUser=docker  # 所属用户和组
SocketGroup=docker 
ListenStream=/var/run/docker.sock  #监听网络流， sequential packet等的地址

[Install] 
WantedBy=sockets.target 

```
其中`.targets`启动级别 ，参考[这里](http://zh.wikipedia.org/wiki/%E8%BF%90%E8%A1%8C%E7%BA%A7%E5%88%AB)。通过下面命令可以查看目标单元下面启动的服务。
```
core@coreos1 /etc/systemd/system $ systemctl show -p "Wants" multi-user.target 
Wants=system-config.target user-config.target myapp.service systemd-networkd.service motdgen.service getty.target dbus.service sshd-ke 
core@coreos1 /etc/systemd/system $ systemctl show -p "Wants" sockets.target 
Wants=systemd-journald.socket systemd-journald-dev-log.socket systemd-udevd-control.socket dbus.socket systemd-shutdownd.socket system
```
systemctl start myapp.service的流程就是： myapp.service启动之前需要启动docker.service,并且使用socket激活式来让docker监听和处理/var/run/docker.sock上的网络流和数据包,然后在myapp.service调用docker client启动一个busybox container，并且在container里面不停地执行一个echo hello world的shell命令。

至于更加详细的有关systemd的介绍，可以参见systemd官方文档[http://www.freedesktop.org/software/systemd/man](http://www.freedesktop.org/software/systemd/man)

#### coreos的[go-systemd](https://github.com/coreos/go-systemd)
systemd的go封装。有如下几个服务包：
* activation 支持写入和使用socket式激活；
* dbus提供运行服务和单元的启停，查询；
* journal支持写入systemd的logging service;
* unit 支持units文件的序列化，反序列化以及对比

#### fleet

fleet基于etcd（分布式kv）和systemd构建，在coreos集群上提供分布式，高容错的应用部署服务。
你可以认为coreos集群共享同一个分布式的systemd，coreos建议用户将应用写的小而且容易在集群内迁移。

主要有以下组件:

##### fleetd 

coreos集群中的每个系统都执行一个单独的fleetd守护进程。fleetd即是engine，也可以是client。engine担当units调度决策者，agent担当unit执行者。engine和client使
用协调模式，周期性的产生实例当前的状态的一个快照，并且跟etcd维护的期望状态对比，并且产生一定的动作来满足期望。

1，  Engine

* Engine负责调度。在跟agent的协调过程中，周期性的或者根据etcd发出的事件触发对应的调度策略。
* 在协调进程运行之前，engine收集集群全体的一个快照信息，包括集群里面所有units的已知状态和期望状态，以及正在运行的agents的集合，并且已知努力去将units的实际状态向期望状态靠近。
* Engine使用租赁模式来保证任何时候只有一个Engine。当一个协调请求到来的时候，Engine试图去从etcd获取一个租约，如果租赁成功，这个协调请求就会得到处理，否则要等下一个协调周期开始。
* Engine使用简单的最小负载调度算法： 有限选择运行units最少的agent来执行调度的unit。

2， Agent

* Agent负责实际执行units。Agent通过D-Bus跟本地的systemd实例通信。
* Agent周期性的从etcd上获取当前实例的快照期望，并且执行对应的动作来满足期望。
* 定期的将units的快照汇报给etcd

#### fleet对象模型

##### Units

一个units就是一个单独的systemd units文件。一个unit被提交之后，其名字和unit 文件的内容就无法更改了，但是unit对应的状态可以改变。因此如果发生修改，需要销毁当前的unit file，并且上传更新之后的unit file。

一系列相关联的unit可以看成一个服务，而不仅仅是一组进程。如果一个unit在挂掉，fleet会在重新找到一个机器进行迁移（reschedule）。

##### State

##### Unit states
由于来自用户和集群的操作，Units和机器的statue都是动态变化的。 
fleet定义了3个集群层面的状态：
* inactive : fleet感知到这个unit，但是还没有真正调度机器上执行；
* loaded： 已经被分配到某个机器，并且已经加载到systemd，但是还没有被启动；
* launched： 加载到systemd，并且执行了systemctl start来启动服务。

状态机的转移图

| Command	| Desired State	| Valid Previous States | 
| :-----: | :-----------: | :-------------------: |
| fleetctl submit |	inactive |	(unknown) | 
| fleetctl load	| loaded | (unknown) or inactive | 
| fleetctl start | launched |	(unknown) or inactive or loaded | 
| fleetctl stop |	loaded | launched |
| fleetctl unload |	inactive | launched or loaded |
| fleetctl destroy |	(unknown)	| launched or loaded or inactive | 

##### systemd states

systemd 状态只能出现在 units state为loaded或者launched的情况下。

* LOAD ： 反应一个unit 定义块是否正确加载
* ACTIVE  ： 高等级的unit 激活状态
* SUB ： 低级别的unit激活状态，根据unit的type来确认值

#### 安全性

目前fleet并不对提交units进行任何的权限验证。任何client都能访问你的etcd集群，在多数机器上都能任意的跑你的代码。
但是公网如果要访问你的etcd集群，就要使用ssh tunnel方式，带上--tunnel 参数来访问。




