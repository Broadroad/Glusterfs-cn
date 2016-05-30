# 安装GlusterFS-一个快速开始指南
这篇文档的目的是在你第一次设置GlusterFS的时候一步一步的指导你。在这篇教程里，我们假设你使用Fedora22（及之后），其他的发型版本和方法能在我们的新用户知道里发现。这里我们也不会详细的解释细节，这篇指南之为了榜之你尽快的使用和运行。在你部署GlusterFS后，我们建议你读GlusterFS Admin Guide来学习怎么管理GlusterFS并且怎么选择一卷（volume）来满足你的需求。阅读新用户指南来获取更多的详细解释。我们希望你能在一个短时间内获得成功。

如果你想要了解使用不同的方法和不同的发行版本的话，看一下安装指南。

## 使用Puppet-Gluster+Vagrant部署GLuster
如果你想使用Pupper-Gluster+Vagrant来自动部署GlusterFS，看一下这篇文章[Automatically deploying GlusterFS with Puppet-Gluster + Vagrant!](https://ttboj.wordpress.com/2014/01/08/automatically-deploying-glusterfs-with-puppet-gluster-vagrant/)

##步骤一 - 需要至少两个节点
* Fedora22(或者之后的发行版本)在两个节点上，各自命名为“Server1”和“Server2”
* 一个有效的网络连接
* 至少两块虚拟磁盘，一块用来安装操作系统，另一款用来服务GlusterFS storage（sdb）。这会模拟真实的部署环境，用于将sdb和操作系统安装区分开来。
* 注意GlusterFS动态存储它的配置文件在/var/lib/glusterd
* 注意：如果在任何时候GlusterFS不能写这些文件（例如背后的文件系统满了），这会给你的系统带来不可预知的错误，甚至，完全让你的系统宕机。建议在其他目录创建别的分区来保证这不会发生。

## 步骤二 - 格式化并且挂载bricks
(在两个节点上)：这些例子都假设brick将属于/dev/sbd1。

```c
	mkfs.xfs -i size=512 /dev/sdb1
    mkdir -p /data/brick1
    echo '/dev/sdb1 /data/brick1 xfs defaults 1 2' >> /etc/fstab
    mount -a && mount
```
你现在应该能看到sdb1挂载在/data/brick1

## 步骤三 - 安装GlusterFS
（在两个节点）：安装软件

```c
yum install glusterfs-server
```

启动GlusterFS管理守护进程：

```c
service glusterd start
    service glusterd status
    glusterd.service - LSB: glusterfs server
           Loaded: loaded (/etc/rc.d/init.d/glusterd)
       Active: active (running) since Mon, 13 Aug 2012 13:02:11 -0700; 2s ago
      Process: 19254 ExecStart=/etc/rc.d/init.d/glusterd start (code=exited, status=0/SUCCESS)
       CGroup: name=systemd:/system/glusterd.service
           ├ 19260 /usr/sbin/glusterd -p /run/glusterd.pid
           ├ 19304 /usr/sbin/glusterfsd --xlator-option georep-server.listen-port=24009 -s localhost...
           └ 19309 /usr/sbin/glusterfs -f /var/lib/glusterd/nfs/nfs-server.vol -p /var/lib/glusterd/...

```

##步骤四 - 配置信任池
在“Server1”

```c
	gluster peer probe server2
```

在“Server2”

```c
	 gluster peer probe server1
```

注意：一旦这个池子建立了，只有信任的成员能注意到新的server。一个新的Server无法感知pool，它必须被池子感知。

##步骤五 - 配置一个GlusterFS卷
在Server1和server2上

```c
	mkdir /data/brick1/gv0
```

从任意一个server：

```c
	gluster volume create gv0 replica 2 server1:/data/brick1/gv0 server2:/data/brick1/gv0
    gluster volume start gv0
```

确认那个卷显示“Started”

```c
	gluster volume info
```
如果这个卷未启动的话，在服务器中的一台或者两台的/var/log/glusterfs目录下确认下log里哪里出错了，通常在etc-glusterfs-glusterd.vol.log。

##步骤六 - 测试GlusterFS卷
在这一步中，我们使用一台服务器来挂载卷。典型的，你应该使用一台外部的机器被称为client来做这些事。因为使用这种方法需要在client上安装额外的包，我们将使用其中一台服务器来简单地测试下，将其当做client。

```c
	mount -t glusterfs server1:/gv0 /mnt
      for i in `seq -w 1 100`; do cp -rp /var/log/messages /mnt/copy-test-$i; done
```

首先，检查下挂载点：

```c
	ls -lA /mnt | wc -l
```

你应该能看到100个文件。然后检查在每一台服务器上的GlusterFS挂载点：

```c
	ls -lA /data/brick1/gv0b
```
你应该能看到100个文件在每个服务器上。如果没有重复的选项的话，在每一台都会看到50个文件。

这里是一些你要了解的术语。

