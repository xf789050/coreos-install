安装过程比较麻烦，如果仅仅按照[官方文档 Installing CoreOS to Disk](https://coreos.com/docs/running-coreos/bare-metal/installing-to-disk/) ，下载iso并且启动，只能得到一个完全无法登录的系统，每次重启之后都会初始化，因为ISO仅仅加载在RAM上。需要将其安装到硬盘中。本文我带领代价在virtualbox虚拟机上安装coreos。

步骤1 ： 解决翻墙问题
       安装过程需要下载coreos镜像。我在百度网盘提供了安装过程中需要的所有文件： [http://pan.baidu.com/s/1sjNSJFJ](http://pan.baidu.com/s/1sjNSJFJ)，因此你需要做的就是下载这四个文件，然后放在一个你可以简单的wget到的位置。例如，我就是在virtualbox上面，虚拟了一个ubuntu，设置网络为网桥，假设ip为192.168.1.130，然后安装一个apache2,并且将其放在/var/www/html下面，
```
root@ubuntu:/var/www/html# ls 
444.5.0 coreos-install coreos_production_image.bin.bz2 id_rsa.pub static.network 
cloud-config.yaml coreos-install2 coreos_production_image.bin.bz2.sig index.html
```
, 然后你再在其他的机器上，执行 `wget http://192.168.1.130/cloud-config.yaml`， 如果下载没有问题，那么步骤一算是完成了。

步骤2 :  配置cloud-config.yaml
    安装过程中最重要的就是修改cloud-config.yaml。我在步骤一已经为大家提供了一个配置模板，大家需要修改的是`ssh-authorized-keys`对应的值。这个是你登录coreos的跳板机ssh的公钥，可以通过如下步骤简单的生成：
```
root@ubuntu:~# ssh-keygen
enter
enter
...
```
中间需要选择的步骤，全部默认回车完成。然后在当前用户的~/.ssh/下面生成了`id_rsa id_rsa.pub`2个文件，将id_rsa.pub里面的加密串贴在`ssh-authorized-keys`对应的值上。
其次是要保证 `static.network` 对应的name和network跟cloud-config.yaml中一致。例如我的配置：
```
#cloud-config

hostname: coreos1

coreos:
  etcd:
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
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
    - name: core 
        ssh-authorized-keys: 
    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDrBCWmYcPhOtsTYtgM7pF4cv5apCP/LHW7ZMZUZO5AZHaDG61fXGmgFc5Sy8t9yRV40/QLGE1BcwRiGVxx1ChPRzFB9/qSzxyzfErt0WGys44ly/d1KvKFNEhZif0hMtKcfGwntI8pILeaRX8pDK6Vct2u3oabPgvFZJZUUCcjv4Sf8ROjV9E8BVjtQNv7iNwgsDEP+Sgdhq/bsR+Nhcp6VX49rbT 
S+jEuAu+6EwvfU0ICv6S0txZn3x1W9b4XG9YfusRXNocNmtPjFsOCpL2hFwJk8mhorQFBLymOttzNcsW6WxuPyLScAbbrQBmgf/ej8GWw61fwKWba77acBNFt core@localho 
st 

    - groups: 
        - sudo 
        - docker
```

步骤3 ： 安装
在virtualbox新建一个虚拟机，网络》连接方式 ： 桥接网卡，然后存储》存储树》Controller： IDE》添加盘片》选择磁盘》选择你刚才下载的coreos_production_iso_image.iso， 最后点击保存
启动刚才添加的虚拟机，是一个core账号所在的用户环境，可以如下直接切换到root
```
sudo -i 
``` 
1， wget http://192.168.1.130/cloud-config.yaml,  因为已经在前面配置好了， 因此你什么都不需要修改(集群环境下，需要修改为不同的网络配置，修改ip，下同)
2,  wget http://192.168.1.130/static.network,  并且
```
mv static.network  /etc/systemd/network/
systemctl restart systemd-networkd 
``` 
3,   wget http://192.168.1.130/coreos-install ,修改里面207行的BASE_URL为你Apache2所在的机器ip，这种方式也有另外的好处。  你也可以直接修改coreos自带的coreos-install的BASE_URL：
```
cp /usr/bin/coreos-install coreos-install 
chmod +x coreos-install
vim 修改BASE_URL （简单点，没有sed了）
```  

4， 安装， 
```
./coreos-install -d /dev/sda -C stable -c ./cloud-config.yaml
```
如果无意外，最后会显示 成功安装的英文巴拉巴拉。

5， 登录
安装完成之后， 需要验证下是否正确安装到磁盘上。 首先停止coreos虚拟机，然后从 设置》存储》controll IDC 》 删除前面步骤选择的iso，这个时候，你就是直接从硬盘启动了。
然后启动虚拟机。
启动之后，如果没有意外，就会显示：
```
...
coreos1 login:
this is coreos1
```
然后，切换到你的跳板机，ssh core@192.168.1.120
```
core@ubuntu:~$ ssh core@192.168.1.120 
Last login: Mon Dec 1 00:37:45 2014 from 192.168.1.130 
CoreOS (stable) 
core@coreos1 ~ $
```
安装成功

步骤4 ： 再次修改cloud-config.yaml

如果你对当前coreos的配置不满意，那么你可以继续修改它（当然也有土豪会选择直接从新安装）
1， 停止coreos，插入coreos iso盘片(步骤3)，并且重新启动
2， 修改
```
[core@localhost ~]$ sudo -i 
[root@localhost~]# mount -o subvol=root /dev/sda9 /mnt 
[root@localhost ~]# vim /mnt/var/lib/coreos-install/user_data
```
3, 停止虚拟机，拔掉光盘，启动就能看到你的配置是否生效。

下一章节将介绍coreos集群安装。

