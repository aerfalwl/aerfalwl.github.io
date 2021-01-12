---

title: clickhouse 安装和使用
date: 2021-01-12 16:19:12
tags:
	- 安装部署
	- 配置
categories: 配置
---
# clickhouse 安装和使用

## 离线安装

[参考官网](https://clickhouse.tech/docs/en/getting-started/install/)

本人的服务器系统版本是centos类型，因此在官网上下载了rpm包在本地安装。[rpm地址](https://repo.yandex.ru/clickhouse/rpm/stable/x86_64/)

下载内容包含三个部分：

1. clickhouse-client-20.9.7.11-2.noarch.rpm
2. clickhouse-server-20.9.7.11-2.noarch.rpm
3. clickhouse-common-static-20.9.7.11-2.x86_64.rpm

以上内容上传至服务器的clickhoust文件夹下，在该文件夹下执行如下命令即安装完毕。

```bash
rpm -ivh * 
```

## 服务启动和停止

有两种方式，一种使用service启动，一种使用systemctl启动。

### service

```bash
service start clickhouse-server
service stop clickhouse-server
service status clickhouse-server
clickhouse-client
```

若service启动过程报错为：

```bash
Init script is already running
```

则使用systemctl方式启动.

### systemctl

```bash
systemctl start clickhouse-server
systemctl stop clickhouse-server
systemctl status clickhouse-server
clickhouse-client
```

两种方式启动的日志文件目录均为```/var/log/clickhouse-server/```。

另外两种启动方式：

```shell
$ sudo /etc/init.d/clickhouse-server start
$ sudo -u clickhouse clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

## 配置

配置文件路径：```/etc/clickhouse-server/```，默认的配置文件为config.xml，但是建议将更新的配置放入config.d文件夹下，以防版本更新导致修改文件被覆盖。

默认数据保存路径：```/var/lib/clickhouse/```

如果改路径下的存储空间比较小，建议将其该为大存储空间下的目录，例如：/app/clickhouse

注意：请注意/app/clickhouse的访问权限，需要赋予clickhouse用户的访问权限，也可以直接使用```chmod 775 /app/clickhouse```为其设置写访问权限。



## 使用

配置完之后，便可使用如下命令访问该数据库。

```shell
$ clickhouse-client --host 127.0.0.1 --port 9000
$ clickhouse-client
$ clickhouse-client --host hostname --port 9000	
```

注意：以上命令中的某个可能会出现如下错误。

```shell
ClickHouse client version 20.4.1.1.
Connecting to localhost:9000 as user default.
Code: 210. DB::NetException: Connection refused (localhost:9000)
```

1. 如果检查节点端口未开放，则可能是clickhouse-server启动的错误。
2. 如果检查节点端口已经开放，则可能是--host后边加的参数的问题，试一下localhose、127.0.0.1等。



## 集群部署

以上所述为单机版部署模式，集群版本的部署模式如下。

### 步骤

1. 参照单机版教程，在每个节点上安装clickhouse。

2. 在每个节点上安装zookeeper，并确保集群模式下的zookeeper服务启动完毕[zookeeper集群部署](https://blog.csdn.net/u014110320/article/details/112532876)。

3. 更改配置文件，具体如下：

   1. 增加如下内容至**每个节点**上的/etc/clickhouse-server/config.xml文件。

      ```xml
      <yandex>
          <!-- 以下为添加内容-->
          <include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
          <include_from>/etc/clickhouse-server/config.d/macros.xml</include_from>
          <listen_host>0.0.0.0</listen_host>
           <!-- 上面为添加内容-->
      </yandex>
      ```

   2. 在文件夹/etc/clickhouse-server/config.d/文件中新增文件metrika.xml（**每个节点上的此配置都相同**），内容如下：(注意节点的hostname和port，以及zookeeper的相关配置)。

      ```xml
      <?xml version="1.0"?>
      <yandex>
          <clickhouse_remote_servers>
              <cluster_1>
                  <shard>
                      <internal_replication>true</internal_replication>
                      <weight>1</weight>
                      <replica>
                          <host>node2</host>
                          <port>9000</port>
                      </replica>
                  </shard>
                  <shard>
                      <internal_replication>true</internal_replication>
                      <weight>1</weight>
                      <replica>
                          <host>node3</host>
                          <port>9000</port>
                      </replica>
                  </shard>
                  <shard>
                      <internal_replication>true</internal_replication>
                      <weight>1</weight>
                      <replica>
                          <host>node4</host>
                          <port>9000</port>
                      </replica>
                  </shard>
              </cluster_1>
          </clickhouse_remote_servers>
       
          <zookeeper-servers>
              <node index="2">
                  <host>node2</host>
                  <port>2181</port>
              </node>
              <node index="3">
                  <host>node3</host>
                  <port>2181</port>
              </node>
              <node index="4">
                  <host>node4</host>
                  <port>2181</port>
              </node>
          </zookeeper-servers>
       
          <clickhouse_compression>
              <case>
                  <min_part_size>10000000000</min_part_size>
                  <min_part_size_ratio>0.01</min_part_size_ratio>
                  <method>lz4</method>
              </case>
          </clickhouse_compression>
      </yandex>
      ```

      

   3. 在文件夹/etc/clickhouse-server/config.d/文件中新增文件macros.xml(**每个节点上的replica配置不同，分别设置为自己的主机名即可**。)

      ```xml
      <yandex>
          <macros>
              <cluster>cluster_1</cluster>
              <shard1>01</shard1>
              <shard2>02</shard2>
              <replica>node4</replica>
          </macros>
      </yandex>
      ```

4. 在每个节点上启动clickhouse-server，并查看/var/log/clickhouse-server/clickhouse-server.log文件确保其正常启动。

###  验证

clickhouse-client连接集群测试部署是否成功。

```shell
$ clickhouse-client --host node2 --port 9000

$ clickhouse-client --host node3 --port 9000

$ clickhouse-client --host node4 --port 9000
```

查看所有集群信息

```sql
select * from system.clusters;
```









