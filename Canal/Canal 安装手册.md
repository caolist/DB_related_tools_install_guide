[TOC]



# 环境

- 操作系统：
- JDK：1.8.0_112
- Elasticsearch：6.2.3

# Canal 安装包

- Canal Deployer：[canal.deployer-1.1.3-SNAPSHOT.tar.gz](https://github.com/alibaba/canal/releases/download/canal-1.1.3-alpha-3/canal.deployer-1.1.3-SNAPSHOT.tar.gz)
- Client Adapter：[canal.adapter-1.1.3-SNAPSHOT.tar.gz](https://github.com/alibaba/canal/releases/download/canal-1.1.3-alpha-3/canal.adapter-1.1.3-SNAPSHOT.tar.gz)

# Canal 部署

## Canal

### MySQL 初始化

canal 的原理是基于 mysql binlog 技术，所以这里一定需要开启 mysql 的 binlog 写入功能，建议配置 binlog 模式为 row。

在 MySQL 数据库配置文件中加入以下内容：

```
[mysqld]
log-bin=mysql-bin #添加这一行就ok
binlog-format=ROW #选择row模式
server_id=1 #配置mysql replaction需要定义，不能和canal的slaveId重复
```

canal 的原理是模拟自己为 mysql slave，所以这里一定需要做为 mysql slave 的相关权限。

在 MySQL 客户端执行以下命令：

```sql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

### 解压缩

```shell
mkdir /tmp/canal
tar zxvf canal.deployer-$version.tar.gz  -C /tmp/canal
```

解压完成后，进入 /tmp/canal 目录，可以看到如下结构：

```
drwxr-xr-x 2 root root   93 Mar 26 16:38 bin
drwxr-xr-x 7 root root  124 Mar 26 15:52 conf
drwxr-xr-x 2 root root 4096 Mar 18 17:30 lib
drwxrwxrwx 6 root root   65 Mar 26 15:55 logs
```

### 修改 canal 配置文件

注意：conf 路径下 一个文件夹代表一个 instance，也就是一个数据库实例，每个文件夹下都有 instance.properties 配置文件。

```shell
vi conf/example/instance.properties
```

```properties
#################################################
## mysql serverId , v1.0.26+ will autoGen
# canal.instance.mysql.slaveId=0

# enable gtid use true/false
canal.instance.gtidon=false

# position info 需要改成自己的数据库信息
canal.instance.master.address=172.16.101.112:3306
canal.instance.master.journal.name=
canal.instance.master.position=
canal.instance.master.timestamp=
canal.instance.master.gtid=

# rds oss binlog
canal.instance.rds.accesskey=
canal.instance.rds.secretkey=
canal.instance.rds.instanceId=

# table meta tsdb info
canal.instance.tsdb.enable=true
#canal.instance.tsdb.url=jdbc:mysql://127.0.0.1:3306/canal_tsdb
#canal.instance.tsdb.dbUsername=canal
#canal.instance.tsdb.dbPassword=canal

#canal.instance.standby.address =
#canal.instance.standby.journal.name =
#canal.instance.standby.position =
#canal.instance.standby.timestamp =
#canal.instance.standby.gtid=

# username/password 需要改成自己的数据库信息
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal12#$asDF

# 代表数据库的编码方式对应到java中的编码类型
canal.instance.connectionCharset = UTF-8 
canal.instance.defaultDatabaseName =test
# enable druid Decrypt database password
canal.instance.enableDruid=false
#canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

# table regex
canal.instance.filter.regex=.*\\..*
# table black regex
canal.instance.filter.black.regex=

# mq config
canal.mq.topic=example
# dynamic topic route by table regex
#canal.mq.dynamicTopic=.*,mytest\\..*,mytest2.user
canal.mq.partition=0
# hash partition config
#canal.mq.partitionsNum=3
#canal.mq.partitionHash=test.table:id^name,.*\\..*
#################################################

```

### 启动

```shell
sh bin/startup.sh
```

### 查看日志

canal 服务器日志：

```shell
vim logs/canal/canal.log
```

```
2019-03-26 16:38:56.124 [main] INFO  com.alibaba.otter.canal.deployer.CanalStater - ## start the canal server.
2019-03-26 16:38:56.183 [main] INFO  com.alibaba.otter.canal.deployer.CanalController - ## start the canal server[172.16.101.112:11111]
2019-03-26 16:38:57.261 [main] INFO  com.alibaba.otter.canal.deployer.CanalStater - ## the canal server is running now ......
```

具体 instance 日志：

```shell
vim logs/example/example.log
```

```
2019-03-26 14:29:53.521 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [canal.properties]
2019-03-26 14:29:53.522 [main] INFO  c.a.o.c.i.spring.support.PropertyPlaceholderConfigurer - Loading properties file from class path resource [example/instance.properties]
2019-03-26 14:29:54.280 [main] INFO  c.a.otter.canal.instance.spring.CanalInstanceWithSpring - start CannalInstance for 1-example
2019-03-26 14:29:54.478 [main] INFO  c.a.otter.canal.instance.core.AbstractCanalInstance - start successful....
```

### 关闭方法

```shell
sh bin/stop.sh
```

至此，单机版 Canal 已经成功部署并启动。

补充：Canal 是可以通过 zookeeper 实现分布式高可用的，关于这一点还在研究中。

## Canal Adapter（适配器）

canal 1.1.1版本之后，内置增加客户端数据同步功能。

这里以 Elasticsearch 适配器为例进行说明。

canal adapter 的 Elasticsearch 版本支持 6.x.x 以上，如需其它版本的 es 可替换依赖重新编译 client-adapter.elasticsearch 模块。

### 解压缩

```shell
mkdir /tmp/canal
tar zxvf canal.adapter-$version.tar.gz  -C /tmp/canal.adapter
```

解压完成后，进入 /tmp/canal.adapter 目录，可以看到如下结构：

```
drwxr-xr-x 2 root root   95 Mar 29 16:17 bin
drwxrwxrwx 6 root root  100 Mar 29 16:02 conf
drwxr-xr-x 2 root root 4096 Mar 25 14:11 lib
drwxrwxrwx 2 root root   25 Mar 25 15:26 logs
drwxrwxrwx 2 root root  289 Jan  7 14:01 plugin
```

### 修改启动器配置： application.yml

```shell
vim conf/application.yml
```

```yaml
server:
  port: 8081
logging:
  level:
    org.springframework: WARN
    com.alibaba.otter.canal.client.adapter.hbase: DEBUG
    com.alibaba.otter.canal.client.adapter.es: DEBUG
    com.alibaba.otter.canal.client.adapter.rdb: DEBUG
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null

canal.conf:
  canalServerHost: 172.16.101.112:11111
#  zookeeperHosts: slave1:2181
#  mqServers: 127.0.0.1:9092 #or rocketmq
#  flatMessage: true
  batchSize: 500
  syncBatchSize: 1000
  retries: 0
  timeout:
  accessKey:
  secretKey:
  mode: tcp # kafka rocketMQ
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://172.16.101.112:3306/canal_test?useUnicode=true
      username: root
      password: root
    defaultDS_plus:
      url: jdbc:mysql://172.16.101.112:3306/canal_test_plus?useUnicode=true
      username: root
      password: root
    113_TestDS:
      url: jdbc:mysql://172.16.101.113:3306/test?useUnicode=true
      username: root
      password: root
    bipDS:
      url: jdbc:mysql://100.50.1.47:3306/bipdat_1119?useUnicode=true
      username: root
      password: 123456
  canalAdapters:
  - instance: example # canal instance Name or mq topic name
    groups:
    - groupId: g1
      outerAdapters:
      - name: logger
#      - name: rdb
#        key: mysql1
#        properties:
#          jdbc.driverClassName: com.mysql.jdbc.Driver
#          jdbc.url: jdbc:mysql://127.0.0.1:3306/mytest2?useUnicode=true
#          jdbc.username: root
#          jdbc.password: 121212
#      - name: rdb
#        key: oracle1
#        properties:
#          jdbc.driverClassName: oracle.jdbc.OracleDriver
#          jdbc.url: jdbc:oracle:thin:@localhost:49161:XE
#          jdbc.username: mytest
#          jdbc.password: m121212
#      - name: rdb
#        key: postgres1
#        properties:
#          jdbc.driverClassName: org.postgresql.Driver
#          jdbc.url: jdbc:postgresql://localhost:5432/postgres
#          jdbc.username: postgres
#          jdbc.password: 121212
#          threads: 1
#          commitSize: 3000
#      - name: hbase
#        properties:
#          hbase.zookeeper.quorum: 127.0.0.1
#          hbase.zookeeper.property.clientPort: 2181
#          zookeeper.znode.parent: /hbase
      - name: es
        hosts: 172.16.101.114:19300,172.16.101.115:19300 # es 集群地址, 逗号分隔
        properties:
          cluster.name: solr_test # es cluster name
  - instance: 113_test # canal instance Name or mq topic name
    groups:
    - groupId: g2
      outerAdapters:
      - name: logger
      - name: es
        hosts: 172.16.101.114:19300,172.16.101.115:19300
        properties:
          cluster.name: solr_test
  - instance: bip_log # canal instance Name or mq topic name
    groups:
    - groupId: g3
      outerAdapters:
      - name: logger
      - name: es
        hosts: 172.16.101.114:19300,172.16.101.115:19300
        properties:
          cluster.name: solr_test
```

说明：这里配置的是多 instance 也就是多库多表到多 es 索引的例子，可根据实际情况做调整。

### 关于适配器表映射文件

adapter 将会自动加载 conf/es 下的所有 .yml 结尾的配置文件。

修改 conf/es/mytest_user.yml 文件:

```shell
vim conf/es/mytest_canal.yml
```

```yaml
dataSourceKey: defaultDS # 源数据源的key, 对应上面配置的srcDataSources中的值
destination: example # cannal的instance或者MQ的topic
esMapping:
  _index: mytest_canal # es 的索引名称
  _type: user # es 的doc名称
  _id: id # es 的_id, 如果不配置该项必须配置下面的pk项_id则会由es自动分配
  pk: id # 如果不需要_id, 则需要指定一个属性为主键属性
  sql: "SELECT test.id as id, test.name as name FROM test"
#  objFields:
#    _labels: array:; # 数组或者对象属性, array:; 代表以;字段里面是以;分隔的
#  etlCondition: # etl 的条件参数
  commitBatch: 3000 # 提交批大小
```

### 建立 es 索引 mytest_canal

```shell
curl -H "Content-Type:application/json" -XPUT http://172.16.101.114:19200/mytest_canal -d '{
	"settings": {
		"number_of_shards": 5,
		"number_of_replicas": 2,
		"max_result_window": 100000000,
		"refresh_interval": "1s",
		"index.translog.durability": "async"
	},
	"mappings": {
		"user": {
			"properties": {
				"id": {
					"type": "keyword"
				},
				"name": {
					"type": "keyword"
				}
			}
		}
	}
}'
```

### 启动 ES 数据同步

启动 canal-adapter

```shell
sh bin/startup.sh
```

### 关闭 ES 数据同步

```shell
sh bin/stop.sh
```

### 验证

1. 新增 mysql canal_test.test 表的数据, 将会自动同步到 es 的 mytest_canal 索引下面, 并会打出 DML 的 log。
2. 修改 mysql canal_test.test 表的数据, 将会自动同步到 es 的 mytest_canal 索引下面, 并会打出 DML 的 log。

# 补充

## 关于多表关联实现

建议参考官网：https://github.com/alibaba/canal/wiki/Sync-ES 

支持以下场景：

- 一对一
- 一对多
- 多对多