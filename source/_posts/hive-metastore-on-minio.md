---
title: hive metastore on minio
date: 2022-04-23
toc: true #是否显示文章目录
categories: "Hive" #分类
tags:   #标签
    - 大数据
    - Hive
---
### hive metastore install step by step
目标是安装好hive metastore stand alone的服务。
本文还使用了s3存储，使用的是minio，把warehouse.dir配置到了对象存储上。
#### 安装mysql
因为存储要使用到mysql,所以直接使用了flink cdc 的一个demo，这个demo中有mysql,pg,es,kibana.
https://ververica.github.io/flink-cdc-connectors/master/content/quickstart/mysql-postgres-tutorial.html
```yaml
version: '2.1'
services:
  postgres:
    image: debezium/example-postgres:1.1
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
  mysql:
    image: debezium/example-mysql:1.1
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_USER=mysqluser
      - MYSQL_PASSWORD=mysqlpw
  elasticsearch:
    image: elastic/elasticsearch:7.6.0
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.type=single-node
    ports:
      - "9200:9200"
      - "9300:9300"
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
  kibana:
    image: elastic/kibana:7.6.0
    ports:
      - "5601:5601"
```
以上内容存储为docker-compose.yml即可，如果不需要pg,es,kibana可以去掉.
docker compose up -d 即可启动mysql.
####  mysql 添加用户hive
```bash
docker-compose exec mysql mysql -uroot -p123456
grant all privileges on *.* to hive@'%' identified by 'hive';
flush privileges ;
```
####  下载安装 hive metastore
```bash
cd ~/app
wget https://repo1.maven.org/maven2/org/apache/hive/hive-standalone-metastore/3.1.2/hive-standalone-metastore-3.1.2-bin.tar.gz
tar -xvf hive-standalone-metastore-3.1.2-bin.tar.gz
rm -f hive-standalone-metastore-3.1.2-bin.tar.gz
mv apache-hive-metastore-3.1.2-bin metastore
cd metastore/lib
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.49/mysql-connector-java-5.1.49.jar  
####metastore 依赖hadoop 
cd ~/app/
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.1/hadoop-3.3.1.tar.gz  
tar -xvf hadoop-3.3.1.tar.gz
rm hadoop-3.3.1.tar.gz
ln -s ./hadoop-3.3.1 hadoop
```
####  更新hive metastore的配置
   metastore-site.xml, mysql 连接信息
```xml
<property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
</property>
<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false</value>
</property>
<property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
</property>
<property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hive</value>
</property>

<property>
    <name>hive.metastore.warehouse.dir</name>
    <value>s3a://hive/warehouse</value>
</property>

  <property>
    <name>fs.defaultFS</name>
    <value>s3a://</value>
  </property>
  <property>
    <name>fs.s3a.access.key</name>
    <value>minio</value>
  </property>
  <property>
    <name>fs.s3a.secret.key</name>
    <value>minio123</value>
  </property>
  <property>
    <name>fs.s3a.connection.ssl.enabled</name>
    <value>false</value>
  </property>
  <property>
    <name>fs.s3a.path.style.access</name>
    <value>true</value>
  </property>
  <property>
    <name>fs.s3a.endpoint</name>
    <value>http://localhost:9000</value>
  </property>
  <property>
    <name>fs.s3a.impl</name>
    <value>org.apache.hadoop.fs.s3a.S3AFileSystem</value>
  </property>

```
####  初始化元数据 
```bash
rm -f ~/app/metastore/lib/guava-19.0.jar
bin/schematool -dbType mysql -initSchema
```
> Exception in thread "main" java.lang.NoSuchMethodError: 'void com.google.common.base.Preconditions.checkArgument(boolean, java.lang.String, java.lang.Object)'
因为guava jar包的冲突，需要提前处理一下。
    
然后检查mysql数据库，发现有一个新建的hive db,里面有多张相关的表。

#### minio的安装使用
minio是一个兼容s3的开源存储服务，非常适合在本地测试对象存储。
```yml
version: '3'
services:
  minio:
    image: minio/minio
    hostname: minio
    ports:
      - 9000:9000 #api 端口
      - 9001:9001 #控制台端口
    environment:
      - MINIO_ROOT_USER=minio
      - MINIO_ROOT_PASSWORD=minio123
    volumes:
      - data-1:/data
    command: server --console-address ":9001" /data
volumes:
  data-1:%
```
存储为docker-compose.yml
docker compose up -d即可启动minio服务。
可以在浏览器中localhost:9001使用前面配置的用户名密码登录即可查看。
####  开启服务进程
```bash
cp conf/metastore-log4j2.properties conf/log4j2.properties
bin/start-metastore
```
> MetaException(message:java.lang.ClassNotFoundException: Class org.apache.hadoop.fs.s3a.S3AFileSystem not found)
因为配置了
<name>hive.metastore.warehouse.dir</name><value>s3a://hive/warehouse</value>
复制aws相关的jar包到lib/目录下。
cd ~/app/hadoop
find . -iname "*aws*.jar" |xargs -I _ cp _ ~/app/metastore/lib/
接着重启服务进程遇到如下错误：
Caused by: MetaException(message:Got exception: org.apache.hadoop.fs.s3a.AWSClientIOException initializing  on s3a://hive/warehouse: com.amazonaws.SdkClientException: Unable to find a region via the region provider chain. Must provide an explicit region in the builder or setup environment to supply a region.: Unable to find a region via the region provider chain. Must provide an explicit region in the builder or setup environment to supply a region.)
把minio相关的访问信息配置到metastore-site.xml 或单独配置一个core-site.xml到conf下都可以。

下章节使用spark-sql or spark-shell来使用hive metastore。


