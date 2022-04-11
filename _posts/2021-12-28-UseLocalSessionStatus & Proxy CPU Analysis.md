---
layout: post
title: UseLocalSessionStatus & Proxy CPU Analysis
tags: MySQL
---
## 背景

1. Proxy CPU冲高，默认分片CPU明显高于其他，查询binlog发现update和insert语句前出现`select @@session.tx_read_only`到默认分片。由useLocalSessionState=false引起。
2. Proxy CPU冲高，查询binlog发现update和insert语句`select @@...`。由未使用连接池，断链建链造成。

## 情景1
### 原因描述
当应用调用JDBC接口（com.mysql.jdbc包中setAutoCommit, setTransactionIsolation 和 setReadOnly）设置参数值时，会与远程服务器同步，取决于useLocalSessionState。useLocalSessionState参数默认false，作用是驱动程序是否使用autocommit，read_only和transaction isolation的内部值（JDBC端的本地值）。

- 如果设置为true，若值与本地值不一致，则发往远程更新；
- 如果设置为false，无论设置值与本地值是否一致，每次都发往远程更新。

### 修改建议
```java
boolean needsSetOnServer = true;
if (this.getUseLocalSessionState() && this.autoCommit == autoCommitFlag) {
    needsSetOnServer = false;
} else if (!this.getHighAvailability()) {
    needsSetOnServer = this.getIO().isSetNeededForAutoCommitMode(autoCommitFlag);
}

this.autoCommit = autoCommitFlag;
if (needsSetOnServer) {
    this.execSQL((StatementImpl)null, autoCommitFlag ? "SET autocommit=1" : "SET autocommit=0", -1, (Buffer)null, 1003, 1007, false, this.database, (Field[])null, false);
}
```

- **当未显式(未调用API)修改三个参数时，默认与服务端保持一致，可以设置为true**
- 考虑原始为自动提交，修改为非自动提交后再修改回自动提交场景。涉及到两次修改会出错。

1. 第一次修改时，若用户设置参数时不通过JDBC接口，而是执行语句 Statement.execute("set auto_commit=0")设置非自动提交， JDBC接口本地没有感知，因此JDBC本地的autocommit值没有变化。
2. 第二次修改时
- 当useLocalSessionState为true时，再次调用setAutoCommit(ture)设置参数时，本地参数由于之前是true，认为没有变化，不发往远程，远程仍然为非自动提交；
- 当useLocalSessionState为false时，每次设置都会发往远程，设置生效，远程连接会被设置为自动提交。

参考[Re: Why is useLocalSessionState false by default](https://forums.mysql.com/read.php?39,626495,626511)官方答复，**出于安全性考虑useLocalSessionStat设置为false**

## 情景2
### 原因描述
建链语句
```
2021-11-29T19:37:12.093515+08:00        5362837 Query   /* mysql-connector-java-5.1.46 ( Revision: 9cc87a48e75c2d2e87c1a293b2862ce651cb256e ) */SELECT  @@session.auto_increment_increment AS auto_increment_increment, @@character_set_client AS character_set_client, @@character_set_connection AS character_set_connection, @@character_set_results AS character_set_results, @@character_set_server AS character_set_server, @@collation_server AS collation_server, @@init_connect AS init_connect, @@interactive_timeout AS interactive_timeout, @@license AS license, @@lower_case_table_names AS lower_case_table_names, @@max_allowed_packet AS max_allowed_packet, @@net_buffer_length AS net_buffer_length, @@net_write_timeout AS net_write_timeout, @@query_cache_size AS query_cache_size, @@query_cache_type AS query_cache_type, @@sql_mode AS sql_mode, @@system_time_zone AS system_time_zone, @@time_zone AS time_zone, @@tx_isolation AS transaction_isolation, @@wait_timeout AS wait_timeout
```
### 修改建议
不使用
```
org.springframework.jdbc.datasource.DriverManagerDataSource
```
```
Class.forName("com.mysql.jdbc.Driver");
Connection conn = DriverManager.getConnection(url,user,pwd);
Statement state = conn.createStatement();
state.executeUpdate("insert ...");
state.close();
conn.close();
```
改为
```
com.alibaba.druid.pool.DruidDataSource
```