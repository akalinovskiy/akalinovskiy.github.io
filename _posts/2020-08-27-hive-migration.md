---
layout: post
title: Hive 2 -> Hive 3 migration guide
tags: [hive, metastore, hdp, hadoop, migration]
comments: false
category: blog
---

# Миграция Hive Metastore (Hive 2 to Hive 3)
Переносим Hive Metastore 2 на новый кластер с Hive 3.
Сначала необходимо перенести данные в /apps/hive/warehouse на новый кластер, например так:
```
hadoop distcp -m 200 hdfs://old-cluster-nn:8020/apps/hive/warehouse hdfs://new-cluster-nn:8020/apps/hive/warehouse
```
Тк на старом кластере для metastore использовался MySQL на новом кластере необходимо установить MySQL (Из коробки установили Postgres).
Можно попробовать использовать **mysqldump --compatible=postgresql** https://en.wikibooks.org/wiki/Converting_MySQL_to_PostgreSQL но сходу не получилось.

### Установка MySQL
Есть вариант не устанавливать MySQL а поставить просто mariadb
Скачиваем rpm (https://downloads.mysql.com/archives/community/ выбираем нужные Product Version, Operating System, OS Version)
```
# Add '-x socks5h://socks5_proxy_server:22020' if needed
[root@prd ~]# curl -L -O -k https://downloads.mysql.com/archives/get/file/mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar
```
Разжимаем tar
```
[root@prd ~]# tar xvf mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar -C mysql_rpms
```
Устанавливаем скачанные rpm
```
[root@prd mysql_rpms]# yum localinstall *.rpm
```
Удаляем старые mariadb-libs
```
rpm --nodeps -e mariadb-libs
```
Запускаем сервис
```
[root@prd]# service mysqld start
```
Смотрим пароль
```
[root@prd ~]# grep 'temporary password' /var/log/mysqld.log
```
Меняем рутовый пароль и настройки 
```
[root@prd ~]# mysql_secure_installation
```
Создаём пользователя hive
```
mysql -p
 
mysql> CREATE USER 'hive'@'%' IDENTIFIED BY 'user_password';
mysql> CREATE DATABASE hive;
mysql> GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'%';
```
Делаем dump базы данных которую собираемся переносить и копируем на наш сервер
```
[admin@old-prd ~]$ sudo mysqldump hive > mysql_hive_database_dump_20190913.sql
[admin@old-prd ~]$ scp mysql_hive_database_dump_20190913.sql root@prd:~/
```
Необходимо поменять имя Namenode в дампе, например через vi (Esc :%s/hdfscluster/mlk-prd/g) меняем все old-cluster на new-cluster
Имя кластера можно посмотреть в Ambari: HDFS → HDFS config в поиске ищем defaultFS
![Ambari](https://github.com/akalinovskiy/tips/blob/master/imgs/hm1.png?raw=true)

Накатываем изменённый дамп
```
[root@prd ~]# mysql -u hive -p hive < mysql_hive_database_dump_20190913.sql
```
Находим ссылку на JDBC driver https://dev.mysql.com/downloads/connector/j/ для 5.7

<img src="https://github.com/akalinovskiy/tips/blob/master/imgs/hm2.png?raw=true" width="300"/>
<br/>
<img src="https://github.com/akalinovskiy/tips/blob/master/imgs/hm2_2.png?raw=true" width="50%"/>


Переходим на ноду с Амбари и ставим JDBC драйвер
```
# add -x socks5h://socks5-proxy:22020 if needed
[root@new-ambari ~]# curl -L -O -k https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.48.tar.gz
[root@new-ambari ~]# tar xzvf mysql-connector-java-5.1.48.tar.gz
[root@new-ambari ~]# ambari-server setup --jdbc-db=mysql --jdbc-driver=mysql-connector-java-5.1.48/mysql-connector-java-5.1.48.jar
```
Возвращаемся на ноду с Metastore
```
[root@new-metastore ~]# cp mysql-connector-java-5.1.48.jar /usr/hdp/current/hive-server2/lib/
```
Запускаем валидацию схемы
```
[root@new-metastore ~]# /usr/hdp/current/hive-server2/bin/schematool -validate -dbType mysql -driver com.mysql.jdbc.Driver -url jdbc:mysql://new-metastore/hive -userName hive -passWord "<password>" -verbose
 
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/hdp/3.1.0.0-78/hive/lib/log4j-slf4j-impl-2.10.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/hdp/3.1.0.0-78/hadoop/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Starting metastore validation
 
Fri Sep 13 20:08:21 MSK 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Validating schema version
Fri Sep 13 20:08:21 MSK 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Metastore schema version is not compatible. Hive Version: 3.1.0, Database Schema Version: 2.1.2000
Failed in schema version validation.
[FAIL]
 
Validating sequence number for SEQUENCE_TABLE
Succeeded in sequence number validation for SEQUENCE_TABLE.
[SUCCESS]
 
Fri Sep 13 20:08:21 MSK 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Validating metastore schema tables
Fri Sep 13 20:08:21 MSK 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
Fri Sep 13 20:08:22 MSK 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
org.apache.hadoop.hive.metastore.HiveMetaException: Unknown version specified for initialization: 2.1.2000
org.apache.hadoop.hive.metastore.HiveMetaException: Unknown version specified for initialization: 2.1.2000
        at org.apache.hadoop.hive.metastore.MetaStoreSchemaInfo.generateInitFileName(MetaStoreSchemaInfo.java:137)
        at org.apache.hive.beeline.HiveSchemaTool.validateSchemaTables(HiveSchemaTool.java:809)
        at org.apache.hive.beeline.HiveSchemaTool.doValidate(HiveSchemaTool.java:636)
        at org.apache.hive.beeline.HiveSchemaTool.main(HiveSchemaTool.java:1546)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.hadoop.util.RunJar.run(RunJar.java:318)
        at org.apache.hadoop.util.RunJar.main(RunJar.java:232)
*** schemaTool failed ***
```
Видим различия версий схемы
```
Metastore schema version is not compatible. Hive Version: 3.1.0, Database Schema Version: 2.1.2000
```
Запускам upgrade
```
[root@new-metastore ~]# /usr/hdp/current/hive-server2/bin/schematool -upgradeSchemaFrom 2.1.2000 -dbType mysql -driver com.mysql.jdbc.Driver -url jdbc:mysql://new-metastore/hive -userName hive -passWord "<password>" -verbose
```

Идем в Ambari -> Services/Hive останавливаем сервисы Actions/Stop 
<br/>
<img src="https://github.com/akalinovskiy/tips/blob/master/imgs/hm3.png?raw=true" width="300"/>
<br/>
Переключаем Metastore на новый MySQL Hive/Configs/Database
<br/>
<img src="https://github.com/akalinovskiy/tips/blob/master/imgs/hm4.png?raw=true" width="50%"/>
<br/>
Подтверждаем предложенные параметры
<br/>
<img src="https://github.com/akalinovskiy/tips/blob/master/imgs/hm5.png?raw=true" width="50%"/>
<br/>
Запускаем сервисы
<br/>
<img src="https://github.com/akalinovskiy/tips/blob/master/imgs/hm6.png?raw=true" width="300"/>
<br/>
На данном этапе уже всё работает, но в Hive 3 разнесли каталоги hive и spark и если нужно использовать [Hive Warehouse Connector](https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.4/integrating-hive/content/hive-etl-example.html) или как раньше использовать один)
Идем в настройки Spark ищем в поиске "catalog" и меняем параметр metastore.catalog.default
![Ambari](https://github.com/akalinovskiy/tips/blob/master/imgs/hm7.png?raw=true)
Сохраняем и перезагружаем сервисы

### Обновление таблиц
Обновляем все таблицы через Zeppelin, следующим скриптом:
```
import org.apache.spark.sql._
import org.apache.spark.sql.expressions._
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._
import scala.util._
 
def refresh(df: DataFrame) = {
    val column = "table"
    val tables = df.select(concat_ws(".", 'database, 'tableName).as(column))
        .collect.map(_.getAs[String](column)).toList
     
    val allTablesCount = tables.size   
    println(s"All tables count: $allTablesCount")
    var lastPrcnt = 0L
    val tries = tables.zipWithIndex map { case (table, i) =>
        val res = Try {
            spark.sql(s"refresh $table")
            spark.sql(s"ANALYZE TABLE $table COMPUTE STATISTICS")
            spark.sql(s"ANALYZE TABLE $table COMPUTE STATISTICS FOR COLUMNS")   
        }
         
        val prcnt = scala.math.round(i.toDouble*100/allTablesCount)
        if(lastPrcnt != prcnt) {
            lastPrcnt = prcnt
            println(s"$prcnt%\t$i/$allTablesCount")
        }   
         
        res
    }
     
    val (successes, failures) = tries.partition(_.isSuccess)
    println(s"All tables count: $allTablesCount")
    println(s"Successes: ${successes.size}")
    println(s"Failures: ${failures.size}")
     
    (successes, failures)
}
 
val databases = Seq("default", "another_db")
databases foreach { db =>
    spark.sql(s"use $db")
    refresh(spark.sql("show tables"))
}
```
