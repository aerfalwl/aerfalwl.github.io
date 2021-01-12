---

title: Zookeeper集群模式部署
date: 2021-01-12 16:18:12
tags:
	- 安装部署
	- 配置
categories: 配置
---

# Zookeeper集群模式部署

```shell
# 解压文件件，例如
tar -xzvf apache-zookeeper-3.6.1-bin.tar.gz -C /home/app/

# 更改配置文件
cd /home/app/apache-zookeeper-3.6.1-bin
cp conf/zoo_sample.cfg conf/zoo.cfg



# 编辑配置文件如下,
# 首先添加如下内容至文件末尾
server.2 = node2:9010:9011
server.3 = node3:9010:9011
server.4 = node4:9010:9011

# 更改文件dataDir的位置。
dataDir=/app/zookeeper

# 分别在node2、node3、node4的/app/zookeeper的目录下放置文件myid，内容分别为2/3/4

# 在三台节点上分别启动如下命令来启动zookeeper
bin/zkServer.sh start

# 查看状态，可分别看到为leader, follower, follower
bin/zkServer.sh status









```

