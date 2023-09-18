# （五）Seata Saga 模式简单整理、Docker 部署 Nacos 单机（基于 Jpom）相关配置

---


## 前言
这篇文章的内容比较简单，没有像前几篇做 Demo 的流程分析，因为 Saga 实际应用比较少，下文只是简单整理一下，顺便补充一下基于 Jpom 安装的 Nacos 单机的相关配置。

## 参考目录
- [Seata 官方文档](https://seata.io/zh-cn/docs/overview/what-is-seata.html)
- [Seata Saga 模式](https://seata.io/zh-cn/docs/user/saga.html)
- [《阿里云云原生架构实践》](https://weread.qq.com/web/bookDetail/731327d07248f2dd7319578)<br>
书本章节 3.5 介绍了分布式事务模式的相关内容。

在官方文档中，对于 Saga 模式的使用作了比较详细的说明，从原理到使用方法，还有操作视频，文末还有常见问题以及回答，如果对 Saga 模式感兴趣的朋友可以自行查看。下文也会截取一些内容进行整理。

## Saga 模式知识点简单整理
### 1、适用场景、优缺点

> （截图自官方文档）<br>
> ![在这里插入图片描述](images/05_SeataSaga/274f52eb0a6e4219b7b81a99b85ce882.png)

来简单比较一下 AT 模式、TCC 模式和 Saga 模式三种 **弱一致性模型** 的调用方式：
| 模式      | 调用方式                                                     |
| --------- | ------------------------------------------------------------ |
| AT 模式   | 底层生成前后镜像，不需要编写任何回滚代码，由 Seata 完成      |
| TCC 模式  | 需要自定义提交以及回滚代码，由注解指定相关方法，Seata 进行调用 |
| Saga 模式 | 需要自定义提交以及回滚代码，并且在状态图（流程图）中指定相关方法 |

### 2、Saga 模式的使用
> （截图自官方文档）<br>
> ![在这里插入图片描述](images/05_SeataSaga/063f04508b30476ab7644e9738d0e022.png)<br>
> ![在这里插入图片描述](images/05_SeataSaga/b3c1af6e30d246148437a9bf7b7bb287.png)

简单点来说，需要完成以下操作：

 1. 编写好相关业务逻辑，包括提交以及回滚的相关操作。
 2. 使用 Seata 提供的状态机设计器画好状态图（流程图），变成 json。
 3. 使用 Seata 引擎读取解析相关 json，执行相关业务。

状态机设计器传送门：
- [代码传送门](https://github.com/seata/seata/tree/develop/saga/seata-saga-statemachine-designer)
- [状态机设计器演示地址](http://seata.io/saga_designer/index.html)
- [状态机设计器视频教程](http://seata.io/saga_designer/vedio.html)

### 3、可能出现的问题以及解决方法
Saga 模式和 TCC 模式比较相似，因此也存在着和 TCC 模式类似的问题：
- 幂等性问题
- 空回滚问题
- 悬挂问题

而官方文档也对此进行了说明：
> （截图自官方文档）<br>![在这里插入图片描述](images/05_SeataSaga/77857c128bb74a888d978537e05cc4fd.png)

隔离性问题以及性能优化问题：
>（截图自官方文档）<br>
>![在这里插入图片描述](images/05_SeataSaga/5492d9e492344568a9b0e94c8edead8a.png)

## Docker 部署 Nacos 单机（基于 Jpom）
在之前说明 Seata-Server 部署的博客里面就有用到 Jpom，但是没有讲到关于 Nacos 相关配置，这篇文章来补充说明一下。

### 步骤 1：拉取镜像
![在这里插入图片描述](images/05_SeataSaga/2adbc26a7e5f4185a852dc4f51d1e703.png)
### 步骤 2：构建容器
![在这里插入图片描述](images/05_SeataSaga/60d377a553784acd96343cbcf5e1368d.png)
上图一开始环境变量没有设置 ip 以及 mysql 相关配置。

如果没有指定 ip，则启动之后会变成内网 ip 导致无法注册。

参考：[seata注册到nacos服务无法访问问题](https://blog.csdn.net/weixin_44471080/article/details/125188477)

![在这里插入图片描述](images/05_SeataSaga/e088a0b4dccf4ffa88b21e64d9e26b17.png)

Nacos 单机模式支持 MySQL，因此加入了 MySQL 相关配置：

![在这里插入图片描述](images/05_SeataSaga/86c429f5d7424802b1cf27ae98e265f5.png)

相关配置sql（[传送门](https://github.com/alibaba/nacos/blob/2.2.3/config/src/main/resources/META-INF/nacos-db.sql)）：

```sql
/*
 * Copyright 1999-2018 Alibaba Group Holding Ltd.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info   */
/******************************************/
CREATE TABLE `config_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) DEFAULT NULL,
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `c_desc` varchar(256) DEFAULT NULL,
  `c_use` varchar(64) DEFAULT NULL,
  `effect` varchar(64) DEFAULT NULL,
  `type` varchar(64) DEFAULT NULL,
  `c_schema` text,
  `encrypted_data_key` text NOT NULL COMMENT '秘钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfo_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_aggr   */
/******************************************/
CREATE TABLE `config_info_aggr` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `datum_id` varchar(255) NOT NULL COMMENT 'datum_id',
  `content` longtext NOT NULL COMMENT '内容',
  `gmt_modified` datetime NOT NULL COMMENT '修改时间',
  `app_name` varchar(128) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfoaggr_datagrouptenantdatum` (`data_id`,`group_id`,`tenant_id`,`datum_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='增加租户字段';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_beta   */
/******************************************/
CREATE TABLE `config_info_beta` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `beta_ips` varchar(1024) DEFAULT NULL COMMENT 'betaIps',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '秘钥',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfobeta_datagrouptenant` (`data_id`,`group_id`,`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_beta';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_info_tag   */
/******************************************/
CREATE TABLE `config_info_tag` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `tag_id` varchar(128) NOT NULL COMMENT 'tag_id',
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL COMMENT 'content',
  `md5` varchar(32) DEFAULT NULL COMMENT 'md5',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  `src_user` text COMMENT 'source user',
  `src_ip` varchar(20) DEFAULT NULL COMMENT 'source ip',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_configinfotag_datagrouptenanttag` (`data_id`,`group_id`,`tenant_id`,`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_info_tag';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = config_tags_relation   */
/******************************************/
CREATE TABLE `config_tags_relation` (
  `id` bigint(20) NOT NULL COMMENT 'id',
  `tag_name` varchar(128) NOT NULL COMMENT 'tag_name',
  `tag_type` varchar(64) DEFAULT NULL COMMENT 'tag_type',
  `data_id` varchar(255) NOT NULL COMMENT 'data_id',
  `group_id` varchar(128) NOT NULL COMMENT 'group_id',
  `tenant_id` varchar(128) DEFAULT '' COMMENT 'tenant_id',
  `nid` bigint(20) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`nid`),
  UNIQUE KEY `uk_configtagrelation_configidtag` (`id`,`tag_name`,`tag_type`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='config_tag_relation';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = group_capacity   */
/******************************************/
CREATE TABLE `group_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `group_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Group ID，空字符表示整个集群',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数，，0表示使用默认值',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_group_id` (`group_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='集群、各Group容量信息表';

/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = his_config_info   */
/******************************************/
CREATE TABLE `his_config_info` (
  `id` bigint(64) unsigned NOT NULL,
  `nid` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `data_id` varchar(255) NOT NULL,
  `group_id` varchar(128) NOT NULL,
  `app_name` varchar(128) DEFAULT NULL COMMENT 'app_name',
  `content` longtext NOT NULL,
  `md5` varchar(32) DEFAULT NULL,
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00',
  `src_user` text,
  `src_ip` varchar(20) DEFAULT NULL,
  `op_type` char(10) DEFAULT NULL,
  `tenant_id` varchar(128) DEFAULT '' COMMENT '租户字段',
  `encrypted_data_key` text NOT NULL COMMENT '秘钥',
  PRIMARY KEY (`nid`),
  KEY `idx_gmt_create` (`gmt_create`),
  KEY `idx_gmt_modified` (`gmt_modified`),
  KEY `idx_did` (`data_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='多租户改造';


/******************************************/
/*   数据库全名 = nacos_config   */
/*   表名称 = tenant_capacity   */
/******************************************/
CREATE TABLE `tenant_capacity` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `tenant_id` varchar(128) NOT NULL DEFAULT '' COMMENT 'Tenant ID',
  `quota` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '配额，0表示使用默认值',
  `usage` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '使用量',
  `max_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个配置大小上限，单位为字节，0表示使用默认值',
  `max_aggr_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '聚合子配置最大个数',
  `max_aggr_size` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '单个聚合数据的子配置大小上限，单位为字节，0表示使用默认值',
  `max_history_count` int(10) unsigned NOT NULL DEFAULT '0' COMMENT '最大变更历史数量',
  `gmt_create` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT '2010-05-05 00:00:00' COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='租户容量信息表';


CREATE TABLE `tenant_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'id',
  `kp` varchar(128) NOT NULL COMMENT 'kp',
  `tenant_id` varchar(128) default '' COMMENT 'tenant_id',
  `tenant_name` varchar(128) default '' COMMENT 'tenant_name',
  `tenant_desc` varchar(256) DEFAULT NULL COMMENT 'tenant_desc',
  `create_source` varchar(32) DEFAULT NULL COMMENT 'create_source',
  `gmt_create` bigint(20) NOT NULL COMMENT '创建时间',
  `gmt_modified` bigint(20) NOT NULL COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_tenant_info_kptenantid` (`kp`,`tenant_id`),
  KEY `idx_tenant_id` (`tenant_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='tenant_info';

CREATE TABLE users (
	username varchar(50) NOT NULL PRIMARY KEY,
	password varchar(500) NOT NULL,
	enabled boolean NOT NULL
);

CREATE TABLE roles (
	username varchar(50) NOT NULL,
	role varchar(50) NOT NULL,
	constraint uk_username_role UNIQUE (username,role)
);

CREATE TABLE permissions (
    role varchar(50) NOT NULL,
    resource varchar(512) NOT NULL,
    action varchar(8) NOT NULL,
    constraint uk_role_permission UNIQUE (role,resource,action)
);

INSERT INTO users (username, password, enabled) VALUES ('nacos', '$2a$10$EuWPZHzz32dJN7jexM34MOeYirDdFAZm2kuWj7VEOJhhZkDrxfvUu', TRUE);

INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
```

![在这里插入图片描述](images/05_SeataSaga/bf45e82ef0fd4364b7bf8b6a83d3c426.png)

再补充一下完整的配置：

- 容器名称：nacos-server
- 端口：8848:8848，9848:9848
- 挂载卷1：
	- 宿主：/usr/local/jpom/nacos/logs
	- 容器：/root/nacos/logs 
- 挂载卷2：
	- 宿主：/usr/local/jpom/nacos/conf
	- 容器：/root/nacos/conf

环境变量：
| 变量名                     | 变量值        | 备注                                             |
| -------------------------- | ------------- | ------------------------------------------------ |
| MODE                       | standalone    | 单机模式                                         |
| JVM_XMS                    | 256m          |                                                  |
| JVM_XMX                    | 512m          |                                                  |
| TIME_ZONE                  | Asia/Shanghai | 时区                                             |
| NACOS_SERVER_IP            | 192.168.2.158 | 指定 ip，如果不指定启动时 ip 为内网 ip，无法注册 |
| SPRING_DATASOURCE_PLATFORM | mysql         |                                                  |
| MYSQL_SERVICE_HOST         | 192.168.2.158 |                                                  |
| MYSQL_SERVICE_PORT         | 3326          |                                                  |
| MYSQL_SERVICE_DB_NAME      | nacos_config  |                                                  |
| MYSQL_SERVICE_USER         | root          |                                                  |
| MYSQL_SERVICE_PASSWORD     | root          |                                                  |

![在这里插入图片描述](images/05_SeataSaga/04ecd2308c164777b0490c2efe83d8d9.png)

![在这里插入图片描述](images/05_SeataSaga/72110ee6ced141cf8e92b7afd60f80d8.png)

访问 Nacos：（用户名：nacos，密码：nacos）
```url
http://192.168.2.158:8848/
```
### 步骤 3：Nacos 设置 Seata 配置文件
![在这里插入图片描述](images/05_SeataSaga/01c3b2e386174a9ca05cc11fb729126b.png)

```properties
service.vgroupMapping.my_test_tx_group=default

service.enableDegrade=false
service.disableGlobalTransaction=false

#Transaction storage configuration, only for the server. The file, DB, and redis configuration values are optional.
store.mode=db
store.lock.mode=db
store.session.mode=db
#Used for password encryption
store.publicKey=

#These configurations are required if the `store mode` is `db`. If `store.mode,store.lock.mode,store.session.mode` are not equal to `db`, you can remove the configuration block.
store.db.datasource=hikari
store.db.dbType=mysql
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://192.168.2.158:3326/seata?useUnicode=true&rewriteBatchedStatements=true&allowPublicKeyRetrieval=true
store.db.user=root
store.db.password=root
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.distributedLockTable=distributed_lock
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000

# redis 模式 store.mode=redis 开启 (控制台查询功能有限,不影响实际执行功能)
# store.redis.host=127.0.0.1
# store.redis.port=6379
# 最大连接数
# store.redis.maxConn=10
# 最小连接数
# store.redis.minConn=1
# store.redis.database=0
# store.redis.password=
# store.redis.queryLimit=100

#Transaction rule configuration, only for the server
server.recovery.committingRetryPeriod=60000
server.recovery.asynCommittingRetryPeriod=60000
server.recovery.rollbackingRetryPeriod=60000
server.recovery.timeoutRetryPeriod=60000
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.rollbackRetryTimeoutUnlockEnable=false
server.distributedLockExpireTime=100000
server.xaerNotaRetryTimeout=600000
server.session.branchAsyncQueueSize=5000
server.session.enableBranchAsyncRemove=false

#Transaction rule configuration, only for the client
client.rm.asyncCommitBufferLimit=60000
client.rm.lock.retryInterval=10
client.rm.lock.retryTimes=30
client.rm.lock.retryPolicyBranchRollbackOnConflict=true
client.rm.reportRetryCount=5
client.rm.tableMetaCheckEnable=true
client.rm.tableMetaCheckerInterval=60000
client.rm.sqlParserType=druid
client.rm.reportSuccessEnable=false
client.rm.sagaBranchRegisterEnable=false
client.rm.sagaJsonParser=fastjson
client.rm.tccActionInterceptorOrder=-2147482648
client.tm.commitRetryCount=5
client.tm.rollbackRetryCount=5
client.tm.defaultGlobalTransactionTimeout=600000
client.tm.degradeCheck=false
client.tm.degradeCheckAllowTimes=10
client.tm.degradeCheckPeriod=2000
client.tm.interceptorOrder=-2147482648
client.undo.dataValidation=true
client.undo.logSerialization=jackson
client.undo.onlyCareUpdateColumns=true
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
client.undo.logTable=undo_log
client.undo.compress.enable=true
client.undo.compress.type=zip
client.undo.compress.threshold=64k

#For TCC transaction mode
tcc.fence.logTableName=tcc_fence_log
tcc.fence.cleanPeriod=1h

#Log rule configuration, for client and server
log.exceptionRate=100

#Metrics configuration, only for the server
metrics.enabled=false
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898

#For details about configuration items, see https://seata.io/zh-cn/docs/user/configurations.html
#Transport configuration, for client and server
transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enableTmClientBatchSendRequest=false
transport.enableRmClientBatchSendRequest=true
transport.enableTcServerBatchSendResponse=false
transport.rpcRmRequestTimeout=300000
transport.rpcTmRequestTimeout=300000
transport.rpcTcRequestTimeout=300000
transport.threadFactory.bossThreadPrefix=NettyBoss
transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
transport.threadFactory.shareBossWorker=false
transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
transport.threadFactory.clientSelectorThreadSize=1
transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
transport.threadFactory.bossThreadSize=1
transport.threadFactory.workerThreadSize=default
transport.shutdown.wait=3
transport.serialization=seata
transport.compressor=none
```

### 步骤 4：修改 Seata Server 相关配置
![在这里插入图片描述](images/05_SeataSaga/480eb60024834d98b4ed4c2a4e19546c.png)

### 步骤5：修改 Seata Client 相关配置
如果使用的是 XA 模式 Demo，修改以下内容：

配置文件：
```properties
seata.registry.type=nacos
seata.registry.nacos.application=seata-server
seata.registry.nacos.server-addr=192.168.2.158:8848
seata.registry.nacos.group=SEATA_GROUP

seata.data-source-proxy-mode=XA
```

pom 文件：

![在这里插入图片描述](images/05_SeataSaga/d29de56c2dc24ff5a717b9f8fe1dbdd0.png)

把数据源配置注释掉（前面在配置文件里面指定了）：

![在这里插入图片描述](images/05_SeataSaga/340a1d0cd59d468b8d847dafb415400c.png)

（完）