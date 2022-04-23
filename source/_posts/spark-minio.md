---
title: spark s3 测试
date: 2022-04-23
toc: true #是否显示文章目录
categories: "Spark" #分类
tags:   #标签
    - 大数据
    - Spark
---
spark的本地安装与测试
环境：mac m1 
安装当前最新版的spark3.2.1并进行简单的测试,整合hive metastore。
hive metastore的安装请参考前面一文章即可。
####  下载安装jdk8
https://www.oracle.com/java/technologies/downloads/#java8-mac
user: hnyaoxh@126.com
pwd: xxxxxx
下载后一步一步安装即可。
因为之前安装了jdk18,所以现在有两个jdk版本。需要在.zshrc中指定JAVA_HOME
或者在spark相关的sh文件中指定也可以。
```bash
export PATH="/Users/student2028/app/spark/bin:$PATH"
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_331.jdk/Contents/Home
PATH=$JAVA_HOME/bin:$PATH:.
CLASSPATH=$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib/dt.jar:.
export JAVA_HOME
export PATH
export CLASSPATH
```
#### 下载安装spark
如果下载without hadoop版本因为缺少各种jar包，所以需要自己补充进来。
同样使用prebuilt hadoop的版本，一定要两者兼容，否则也会报相关错误。
当前最新版本spark3.2兼容 hadoop3.3.1。如果版本不兼容，常报错误有
> Exception in thread "main" java.lang.NoSuchFieldError: JAVA_9
```bash
wget https://dlcdn.apache.org/spark/spark-3.2.1/spark-3.2.1-bin-hadoop3.2.tgz
tar xvzf spark-3.2.1-bin-hadoop3.2.tgz
rm spark-3.2.1-bin-hadoop3.2.tgz
ln -s ./spark-3.2.1-bin-hadoop3.2 spark
##copy aws related jars to lib 注意此处的hadoop 需要是3.3版本
cd ~/app/hadoop
find . -iname "*aws*.jar" |xargs -I _ cp _ ~/app/spark/jars
```
#### 测试读写s3的数据
```bash
spark-shell \
--conf spark.hadoop.fs.s3a.bucket.all.committer.magic.enabled=true \
--conf spark.hadoop.fs.s3a.access.key=minio \
--conf spark.hadoop.fs.s3a.secret.key=minio123 \
--conf spark.hadoop.fs.s3a.endpoint=localhost:9000 \
--conf spark.hadoop.fs.s3a.path.style.access=true \
--conf spark.hadoop.fs.s3a.connection.ssl.enabled=false \
--conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \
--conf spark.hadoop.fs.s3a.bucket.all.committer.magic.enabled=true 

val df=Seq((1,"stu1","M",33),(2,"stu2","F",30)).toDF("id","name","gender","age")
df.write.format("json").mode("overwrite").save("s3a://test/student");

```
> WARN AbstractS3ACommitterFactory: Using standard FileOutputCommitter to commit work. This is slow and potentially unsafe.
  spark3.2开始提供了更好用的提交协议 ，请参考：
  https://spot.io/blog/improve-apache-spark-performance-with-the-s3-magic-committer/

> java.lang.ClassNotFoundException: org.apache.spark.internal.io.cloud.PathOutputCommitProtocol

解决方案：
```bash
1. cd  ~/app/spark/jars/
   wget https://repo1.maven.org/maven2/org/apache/spark/spark-hadoop-cloud_2.12/3.2.1/spark-hadoop-cloud_2.12-3.2.1.jar

2. spark-shell --packages org.apache.spark:spark-hadoop-cloud_2.12:3.2.1

```
> Class org.apache.hadoop.fs.s3a.auth.IAMInstanceCredentialsProvider not found
这个问题的原因是spark预置的hadoop版本和hadoop-aws配置的版本不一致导致的，先看一下spark项目依赖的hadoop版本，
修正hadoop-aws的版本即可，一个方法是在maven项目里面，先去掉hadoop-aws依赖，
先看一下External Libraries 中搜一下hadoop即可以发现之前依赖的hadoop版本。

#### 测试delta lake 
```bash
cd ~/app/spark/jars
wget  https://repo1.maven.org/maven2/io/delta/delta-core_2.12/1.2.0/delta-core_2.12-1.2.0.jar
wget  https://repo1.maven.org/maven2/io/delta/delta-storage/1.2.0/delta-storage-1.2.0.jar

spark-shell \
--conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
--conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog" \
--conf spark.hadoop.fs.s3a.bucket.all.committer.magic.enabled=true \
--conf spark.hadoop.fs.s3a.access.key=minio \
--conf spark.hadoop.fs.s3a.secret.key=minio123 \
--conf spark.hadoop.fs.s3a.endpoint=localhost:9000 \
--conf spark.hadoop.fs.s3a.path.style.access=true \
--conf spark.hadoop.fs.s3a.connection.ssl.enabled=false \
--conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \
--conf spark.hadoop.fs.s3a.bucket.all.committer.magic.enabled=true 

spark.range(5, 10).write.format("delta").mode("overwrite").save("s3a://test/delta1")
spark.read.format("delta").load("s3a://test/delta1").show(20)
配合hivemetastore 创建表
spark-sql \
--conf "spark.sql.warehouse.dir=s3a://hive/warehouse" \
--conf "spark.sql.extensions=io.delta.sql.DeltaSparkSessionExtension" \
--conf "spark.sql.catalog.spark_catalog=org.apache.spark.sql.delta.catalog.DeltaCatalog" \
--conf spark.hadoop.fs.s3a.access.key=minio \
--conf spark.hadoop.fs.s3a.secret.key=minio123 \
--conf spark.hadoop.fs.s3a.endpoint=localhost:9000 \
--conf spark.hadoop.fs.s3a.path.style.access=true \
--conf spark.hadoop.fs.s3a.connection.ssl.enabled=false \
--conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \
--conf spark.hadoop.fs.s3a.bucket.all.committer.magic.enabled=true \
--conf "spark.hadoop.hive.metastore.uris=thrift://localhost:9083" 
```
```sql
create database if not exists test;
use test;
drop table test2;
create table test2 (id int, name string) using delta;
insert into test2 values(1,'1');
insert into test2 values(2,'22');
select * from test2;
```

#### spark java project demo
```java

import org.apache.spark.SparkConf;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.functions;
import java.util.Random;
public class TestDelta {
    public static void main(String[] args) {

        SparkConf sparkConf = new SparkConf();
        sparkConf.set("spark.sql.warehouse.dir","s3a://hive/warehouse");
        sparkConf.set("spark.hadoop.fs.s3a.bucket.all.committer.magic.enabled","true");
        sparkConf.set("spark.sql.catalogImplementation","hive");//enabled hive support
        sparkConf.set("spark.sql.extensions","io.delta.sql.DeltaSparkSessionExtension");
        sparkConf.set("spark.sql.catalog.spark_catalog","org.apache.spark.sql.delta.catalog.DeltaCatalog");
        sparkConf.set("spark.hadoop.hive.metastore.uris","thrift://localhost:9083");
        sparkConf.set("spark.hadoop.hive.metastore.warehouse.dir","s3a://hive/warehouse");

        sparkConf.set("fs.s3a.access.key", "minio");
        sparkConf.set("fs.s3a.secret.key", "minio123");
        sparkConf.set("fs.s3a.endpoint","localhost:9000");
        sparkConf.set("fs.s3a.path.style.access", "true");
        sparkConf.set("fs.s3a.connection.ssl.enabled", "false");
        sparkConf.set("fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem");

        SparkSession spark = SparkSession.builder().config(sparkConf).master("local").getOrCreate();
//        spark.sparkContext().setLogLevel("warn");
//        spark.sql("create database test");
//        spark.sql("use test");
//        spark.sql("drop table if exists test2");
//        spark.sql("create table if not exists test2(id int,name string)");
//        spark.sql("insert into test2 values(1,'1')");
        spark.sql("use test");
        spark.sql("show tables").show();
        spark.sql("select * from test.test2").show();
        spark.stop();
    }

}

```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.examples</groupId>
    <artifactId>sparktest</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>io.delta</groupId>
            <artifactId>delta-core_2.12</artifactId>
            <version>1.2.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hadoop-cloud_2.12</artifactId>
            <version>3.2.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-aws</artifactId>
            <version>3.3.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_2.12</artifactId>
            <version>3.2.1</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.12</artifactId>
            <version>3.2.1</version>
        </dependency>
    </dependencies>
</project>
```