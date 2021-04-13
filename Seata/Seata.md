# Seata

#### 引包

```gradle
implementation('com.alibaba.cloud:spring-cloud-starter-alibaba-seata:2.2.5.RELEASE')
```

#### 启动报错

> failed to access class org.springframework.cloud.openfeign.loadbalancer.FeignBlockingLoadBalancerClient from class com.alibaba.cloud.seata.feign.SeataFeignObjectWrapper 

原因：openfeign 版本冲突

```gradle
implementation 'org.springframework.cloud:spring-cloud-openfeign-core:2.2.1.RELEASE'
```

> no available service 'null' found, please make sure registry config correct

配置不正确

```shell
seata:
  enabled: true
  data-source-proxy-mode: AT
  application-id: user-access
  tx-service-group: my_test_tx_group # 事务群组（可以每个应用独立取名，也可以使用相同的名字）
  enable-auto-data-source-proxy: true
  service:
    vgroupMapping:
      my_test_tx_group: default
  registry:
    type: nacos
    nacos:
      server-addr: ${NACOS_CFG_SERVER_ADDR:127.0.0.1:8848}
      group: SEATA_GROUP
  config:
    nacos:
      server-addr: 127.0.0.1:8848
      group: ${NACOS_CFG_SERVER_ADDR:127.0.0.1:8848}
```



其他配置说明：

```shell
seata:
  enabled: true
  data-source-proxy-mode: AT
  application-id: account-seata-example
  tx-service-group: my_test_tx_group # 事务群组（可以每个应用独立取名，也可以使用相同的名字）
  enable-auto-data-source-proxy: true
  use-jdk-proxy: false
  client:
    rm:
      report-success-enable: true
      table-meta-check-enable: false # 自动刷新缓存中的表结构（默认false）
      report-retry-count: 5 # 一阶段结果上报TC重试次数（默认5）
      async-commit-buffer-limit: 10000 # 异步提交缓存队列长度（默认10000）
      lock:
        retry-interval: 10 # 校验或占用全局锁重试间隔（默认10ms）
        retry-times:    30 # 校验或占用全局锁重试次数（默认30）
        retry-policy-branch-rollback-on-conflict: true # 分支事务与其它全局回滚事务冲突时锁策略（优先释放本地锁让回滚成功）
    tm:
      commit-retry-count:   3 # 一阶段全局提交结果上报TC重试次数（默认1次，建议大于1）
      rollback-retry-count: 3 # 一阶段全局回滚结果上报TC重试次数（默认1次，建议大于1）
      degrade-check: false
      degrade-check-period: 2000
      degrade-check-allow-times: 10
    undo:
      data-validation: true # 二阶段回滚镜像校验（默认true开启）
      log-serialization: jackson # undo序列化方式（默认jackson）
      log-table: undo_log  # 自定义undo表名（默认undo_log）
    log:
      exceptionRate: 100 # 日志异常输出概率（默认100）

  service:
    vgroupMapping:
      my_test_tx_group: default  # TC 集群（必须与seata-server保持一致）
    enable-degrade: false # 降级开关
    disable-global-transaction: false # 禁用全局事务（默认false）
    grouplist:
      default: 127.0.0.1:8091
  transport:
    shutdown:
      wait: 3
    thread-factory:
      boss-thread-prefix: NettyBoss
      worker-thread-prefix: NettyServerNIOWorker
      server-executor-thread-prefix: NettyServerBizHandler
      share-boss-worker: false
      client-selector-thread-prefix: NettyClientSelector
      client-selector-thread-size: 1
      client-worker-thread-prefix: NettyClientWorkerThread
    type: TCP
    server: NIO
    heartbeat: true
    serialization: seata
    compressor: none
    enable-client-batch-send-request: true # 客户端事务消息请求是否批量合并发送（默认true）
  registry:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace:
      group: DEFAULT_GROUP
      cluster: default
  config:
#    file:  # 使用 nacos 后就不能打开，否则报错
#      name: file.conf
#    type: nacos  # 不能打开，否则报错，有可能是新版已经没有该配置
    nacos:
      namespace:
      server-addr: localhost:8848
      group: DEFAULT_GROUP
```



#### 数据库表

```shell
CREATE TABLE `branch_table` (
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(128) NOT NULL,
  `transaction_id` bigint(20) DEFAULT NULL,
  `resource_group_id` varchar(32) DEFAULT NULL,
  `resource_id` varchar(256) DEFAULT NULL,
  `branch_type` varchar(8) DEFAULT NULL,
  `status` tinyint(4) DEFAULT NULL,
  `client_id` varchar(64) DEFAULT NULL,
  `application_data` varchar(2000) DEFAULT NULL,
  `gmt_create` datetime(6) DEFAULT NULL,
  `gmt_modified` datetime(6) DEFAULT NULL,
  PRIMARY KEY (`branch_id`),
  KEY `idx_xid` (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `global_table` (
  `xid` varchar(128) NOT NULL,
  `transaction_id` bigint(20) DEFAULT NULL,
  `status` tinyint(4) NOT NULL,
  `application_id` varchar(32) DEFAULT NULL,
  `transaction_service_group` varchar(32) DEFAULT NULL,
  `transaction_name` varchar(128) DEFAULT NULL,
  `timeout` int(11) DEFAULT NULL,
  `begin_time` bigint(20) DEFAULT NULL,
  `application_data` varchar(2000) DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL,
  `gmt_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`xid`),
  KEY `idx_gmt_modified_status` (`gmt_modified`,`status`),
  KEY `idx_transaction_id` (`transaction_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `lock_table` (
  `row_key` varchar(128) NOT NULL,
  `xid` varchar(96) DEFAULT NULL,
  `transaction_id` bigint(20) DEFAULT NULL,
  `branch_id` bigint(20) NOT NULL,
  `resource_id` varchar(256) DEFAULT NULL,
  `table_name` varchar(32) DEFAULT NULL,
  `pk` varchar(36) DEFAULT NULL,
  `gmt_create` datetime DEFAULT NULL,
  `gmt_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`row_key`),
  KEY `idx_branch_id` (`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `undo_log` (
  `branch_id` bigint(20) NOT NULL COMMENT 'branch transaction id',
  `xid` varchar(100) NOT NULL COMMENT 'global transaction id',
  `context` varchar(128) NOT NULL COMMENT 'undo_log context,such as serialization',
  `rollback_info` longblob NOT NULL COMMENT 'rollback info',
  `log_status` int(11) NOT NULL COMMENT '0:normal status,1:defense status',
  `log_created` datetime(6) NOT NULL COMMENT 'create datetime',
  `log_modified` datetime(6) NOT NULL COMMENT 'modify datetime',
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='AT transaction mode undo table';
```



