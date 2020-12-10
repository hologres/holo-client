# 通过holo-client读写Hologres

## 功能介绍
holo-client适用于大批量数据写入（批量、实时同步至holo）和高QPS点查（维表关联）场景。holo-client基于JDBC实现，使用时请确认实例剩余可用连接数。

- 查看最大连接数
```sql
show max_connections;
```

- 查看已使用连接数
```sql
select count(*) from pg_stat_activity;
```
## holo-client引入
holo-client依赖特殊的pgjdbc版本，请确认项目中没有依赖org.postgresql:postgresql，避免出现冲突

- Maven
```xml
<dependency>
  <groupId>com.alibaba.hologres</groupId>
  <artifactId>holo-client</artifactId>
  <version>1.2.3</version>
</dependency>
```

- Gradle
```
implementation 'com.alibaba.hologres:holo-client:1.2.3'
```
## 数据写入
holo-client写入是非线程安全的，如需并发写入，请创建多个HoloClient对象
### 写入普通表
```java
// 配置参数,url格式为 jdbc:postgresql://host:port/db
HoloConfig config = new HoloConfig();
config.setJdbcUrl(url);
config.setUsername(username);
config.setPassword(password);

config.setWriteBatchByteSize(10*1024*1024L);
config.setWriteBatchSize(1024);
try (HoloClient client = new HoloClient(config)) {
    TableSchema schema0 = client.getTableSchema("t0");
    Put put = new Put(schema0);
    put.setObject(0, 1);
    put.setObject(1, "name0");
    put.setObject(2, "address0");
    client.put(put); 
    ...
    client.flush(); //强制提交所有未提交put请求；当未提交put请求满足WriteBatchSize或WriteBatchByteSize也会自动提交
catch(HoloClientException e){
}
```
### 写入分区表
注1：若分区已存在，不论DynamicPartition为何值，写入数据都将插入到正确的分区表中；若分区不存在，DynamicPartition设置为true时，将会自动创建不存在的分区，否则抛出异常
注2: 写入分区表在HOLO 0.9及以后版本才能获得较好的性能，0.8建议先写到临时表，再通过insert into xxx select ...的方式写入到分区表
```java
// 配置参数,url格式为 jdbc:postgresql://host:port/db
HoloConfig config = new HoloConfig();
config.setJdbcUrl(url);
config.setUsername(username);
config.setPassword(password);

config.setWriteBatchByteSize(10*1024*1024L);
config.setWriteBatchSize(1024);

config.setDynamicPartition(true); //当分区不存在时，将自动创建分区

try (HoloClient client = new HoloClient(config)) {
    //create table t0(id int not null,region text not null,name text,primary key(id,region)) partition by list(region)
    TableSchema schema0 = client.getTableSchema("t0");
    Put put = new Put(schema0);
    put.setObject(0, 1);
    put.setObject(1, "SH");
    put.setObject(2, "name0");
    client.put(put); 
    ...
    client.flush(); //强制提交所有未提交put请求；当未提交put请求满足WriteBatchSize或WriteBatchByteSize也会自动提交
catch(HoloClientException e){
}
```
### 写入含主键表
```java
// 配置参数,url格式为 jdbc:postgresql://host:port/db
HoloConfig config = new HoloConfig();
config.setJdbcUrl(url);
config.setUsername(username);
config.setPassword(password);

config.setWriteBatchByteSize(10*1024*1024L);
config.setWriteBatchSize(1024);

config.setWriteMode(WriteMode.INSERT_OR_REPLACE);//配置主键冲突时策略

try (HoloClient client = new HoloClient(config)) {
    //create table t0(id int not null,name0 text,address text,primary key(id))
    TableSchema schema0 = client.getTableSchema("t0");
    Put put = new Put(schema0);
    put.setObject(0, 1);
    put.setObject(1, "name0");
    put.setObject(2, "address0");
    client.put(put); 
    ...
    put = new Put(schema0);
    put.setObject(0, 1);
    put.setObject(1, "newName");
    put.setObject(2, "newAddress");
    client.put(put);
    ...
    client.flush();//强制提交所有未提交put请求；当未提交put请求量满足WriteBatchSize或WriteBatchByteSize也会自动提交
catch(HoloClientException e){
}
```
## 数据查询
holo-client查询是线程安全的，可根据实际吞吐需求，创建多个HoloClient对象
### 基于完整主键查询
```java
// 配置参数,url格式为 jdbc:postgresql://host:port/db
HoloConfig config = new HoloConfig();
config.setJdbcUrl(url);
config.setUsername(username);
config.setPassword(password);
try (HoloClient client = new HoloClient(config)) {
    //create table t0(id int not null,name0 text,address text,primary key(id))
    TableSchema schema0 = client.getTableSchema("t0");
    
    Object[] primaryKeys=new Object[]{1}; // where id=1;
    client.get(new Get(schema0,primaryKeys)).thenAcceptAsync((record)->{
        // do something after get result
    });
catch(HoloClientException e){
}
    
```


## 附录
### HoloConfig参数说明
| 参数名 | 默认值 | 说明 |
| --- | --- | --- |
| jdbcUrl | 无 | 必填
 |
| username | 无 | 必填 |
| password | 无 | 必填 |
| dynamicPartition | false | 若为true，当分区不存在时自动创建分区 |
| writeMode | INSERT_OR_REPLACE | 当INSERT目标表为有主键的表时采用不同策略<br>INSERT_OR_IGNORE 当主键冲突时，不写入<br>INSERT_OR_UPDATE 当主键冲突时，更新相应列<br>INSERT_OR_REPLACE当主键冲突时，更新所有列|
| writeBatchSize | 512 | 在经过WriteMode合并后的Put数量达到writeBatchSize时进行一次批量提交 |
| writeBatchByteSize | 100MB | 在经过WriteMode合并后的Put数据字节数达到writeBatchByteSize时进行一次批量提交 |
| retryCount | 3 | 当连接故障时，写入和查询的重试次数 |
| retrySleepInitMs | 1000 ms | 每次重试的等待时间=retrySleepInitMs+retry*retrySleepStepMs |
| retrySleepStepMs | 10*1000 ms | 每次重试的等待时间=retrySleepInitMs+retry*retrySleepStepMs |



