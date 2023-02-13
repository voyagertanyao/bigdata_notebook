
## 组件版本概览

| 组件                      | 版本                       | 说明                     |
| ------------------------- | -------------------------- | ------------------------ |
| hadoop                    | 3.2.0                      | 依赖aws相关包            |
| hive                      | 3.1.3                      | jar包非必需依赖        |
| hive standalone metastore | 3.0.0                      | 非必需依赖，但需要metastore服务         |
| minio docker              | quay.io/minio/minio latest |                          |
| spark                     | 3.1.1                      | 依赖aws包，hive-site.xml |
| kylin                     | 4.0.3                      |                          |
| zookeeper                 | 正常版本                   |                          |
| mysql                          |         5.7                   |                          |

**说明**： 无hadoop环境主要指安装节点不属于hadoop集群，hdfs、yarn、hive等功能不可用，所以该种方案利用对象存储替换hdfs，利用spark standalone替换yarn，依赖hadoop、hive相关jar包和配置。ZK可以任意选择，基本不存在兼容性问题，本例选择的CDH6.2中的实例。hive安装包可能并不需要，metastore需要提前启动服务，致于是否是standalone方式，kylin并无感知。



## 逻辑架构图


## 部署流程

### 1. 下载安装包
[hadoop-3.2.0](https://archive.apache.org/dist/hadoop/common/hadoop-3.2.0/hadoop-3.2.0.tar.gz) 

[spark-3.1.1-with-hadoop3](https://archive.apache.org/dist/spark/spark-3.1.1/spark-3.1.1-bin-hadoop3.2.tgz) 

[kylin-4.0.3-spark-3.1](https://www.apache.org/dyn/closer.cgi/kylin/apache-kylin-4.0.3/apache-kylin-4.0.3-bin-spark3.tar.gz) 

[hive-3.1.3](https://dlcdn.apache.org/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz) 

[hive-standalone-metastore-3.0.0](https://dlcdn.apache.org/hive/hive-standalone-metastore-3.0.0/hive-standalone-metastore-3.0.0-bin.tar.gz) 

mysql-connector-java.jar  


### 2. hadoop配置
1. 配置HADOOP_HOME：`export HADOOP_HOME=/data/hadoop3.2.0`,`export PATH=$PATH:${HADOOP_HOME}/bin`
2. 配置HADOOP_CONF_DIR:  `export HADOOP_CONF_DIR=/data/hadoop3.2.0/etc/hadoop`
3. 复制aws相关包: 
	1. `cp ${HADOOP_HOME}/share/hadoop/tools/lib/aws-java-sdk-bundle-1.11.375.jar ${HADOOP_HOME}/share/hadoop/common/lib/`
	2. `cp ${HADOOP_HOME}/share/hadoop/tools/lib/hadoop-aws-3.2.0.jar ${HADOOP_HOME}/share/hadoop/common/lib/`
4. 配置文件修改`${HADOOP_HOME}/etc/hadoop/core-site.xml`
```xml
<configuration>
    <property>
        <name>fs.s3a.access.key</name>
        <value>metastoreaccesskey</value>
    </property>
    <property>
        <name>fs.s3a.secret.key</name>
        <value>metastoresecretkey</value>
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
        <value>10.12.21.58:9876</value>
    </property>
    <!-- 这个一定要加上，指定文件系统为s3系统 -->
	<property>
        <name>fs.defaultFS</name>
        <value>s3a://metastore</value>
    </property>
</configuration>
```
5. 使用命令行`hadoop version`验证一下版本信息

### 3. hive包配置
1. 配置`HIVE_HOME`: `export HIVE_HOME=/data/hive3.1.3`,`export PATH=$PATH:${HIVE_HOME}/bin`
2. 配置`${HIVE_HOME}/conf/hive-site.xml`文件
```xml
<configuration>
   <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://10.12.21.22:3306/meta3</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>!QAZxsw2</value>
    </property>
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>
    <property>
        <name>metastore.thrift.uris</name>
        <value>thrift://10.12.21.58:9083</value>
        <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
    </property>
    <property>
        <name>metastore.task.threads.always</name>
        <value>org.apache.hadoop.hive.metastore.events.EventCleanerTask</value>
    </property>
    <property>
        <name>metastore.expression.proxy</name>
        <value>org.apache.hadoop.hive.metastore.DefaultPartitionExpressionProxy</value>
    </property>
    <property>
        <name>metastore.warehouse.dir</name>
        <value>s3a://metastore/warehouse</value>
    </property>
    <property>
    	<name>datanucleus.schema.autoCreateAll</name>
    	<value>true</value>
    </property>

    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
    <property>
        <name>hive.txn.manager</name>
        <value>org.apache.hadoop.hive.ql.lockmgr.DbTxnManager</value>
    </property>
```


### 4. 安装spark standalone
1. 配置SPARK_HOME: `export SPARK_HOME=/data/spark`,`export PATH=$PATH:${SPARK_HOME}/bin`
2. 将aws相关包、mysql驱动复制到`${SPARK_HOME}/jars/`下面
3. 配置`${SPARK_HOME}/conf/spark-default.conf`文件
```plain
# hiveaccesskey
spark.hadoop.fs.s3a.access.key metastoreaccesskey
# hivesecretkey
spark.hadoop.fs.s3a.secret.key metastoresecretkey

spark.hadoop.fs.s3a.connection.ssl.enabled false
spark.hadoop.fs.s3a.path.style.access true
spark.hadoop.fs.s3a.endpoint 10.12.21.58:9876
spark.hadoop.fs.s3a.impl org.apache.hadoop.fs.s3a.S3AFileSystem
spark.hadoop.fs.s3a.aws.credentials.provider org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider

# spark连接hive standalone metastore(不指定metastore version等信息)
spark.sql.warehouse.dir s3a://metastore/warehouse
spark.hadoop.hive.metastore.uris thrift://10.12.21.58:9083
```
4. 配置`${SPARK_HOME}/conf/spark-env.sh`文件
```bash
SPARK_HOME=/data/spark
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.322.b06-1.el7_9.x86_64
HADOOP_CONF_DIR=/data/hadoop3.2.0/etc/hadoop
HADOOP_HOME=/data/hadoop3.2.0
```
5. 启动master和worker
```bash
${SPARK_HOME}/sbin/start-master.sh
${SPARK_HOME}/sbin/start-worker.sh spark://slave010:7077
```
6. 验证spark与metastore、s3的连通性
`${SPARK_HOME}/bin/spark-shell --master spark://slave010:7077`
>scala> spark.sql("show databases").show(false)
>scala> spark.sql("insert into bigdata.order partition (dt = '2023-02-10') select 1, '2023', 1.2")

### 5.安装kylin
1. 配置${KYLIN_HOME}: `export KYLIN_HOME=/data/kylin4.0.3`
2. 配置`${KYLIN_HOME}/conf/kylin.properties`文件
```plain
kylin.metadata.url=metastore@jdbc,url=jdbc:mysql://10.12.21.22:3306/kylin_s3,username=root,password=!QAZxsw2,maxActive=10,maxIdle=10
kylin.env.zookeeper-connect-string=slave003:2181
kylin.engine.spark-conf.spark.master=spark://slave010:7077
kylin.engine.spark-conf.spark.submit.deployMode=client
kylin.env.hdfs-working-dir=/kylin_s3
kylin.env.zookeeper-base-path=/kylin_s3
kylin.engine.spark-conf.spark.eventLog.dir=s3a:/metastore/kylin_s3/spark-history
kylin.engine.spark-conf.spark.history.fs.logDirectory=s3a:/metastore/kylin_s3/spark-history
kylin.query.spark-conf.spark.master=spark://slave010:7077
```
3. 拷贝mysql驱动到`${KYLIN_HOME}/ext`目录下，没有则新建目录
4. 拷贝aws相关包到`${KYLIN_HOME}/tomcat/webapps/kylin/WEB-INF/lib`目录下，没有则新建目录（webapps下目录为kylin启动时自动将war包解压后所得，可以先启动一次kylin）
5. 修改`${KYLIN_HOME}/bin/check-env.sh`文件，将所有类似`{xxx/s3a/s3}`替换改为不替换`{xxx/s3a/s3a}`
6. 在`/etc/bashrc`中指定`HADOOP_HOME`,`HIVE_HOME`,执行命令`${KYLIN_HOME}/bin/check-env.sh`，检查s3上是否新增对应目录
7. 启动kylin：`${KYLIN_HOME}/bin/kylin.sh start`


## 当前的问题
1. 兼容性
	1. kylin4.x只在spark3.1.1上做了功能验证，其他高版本不支持（spark3.2.1构建失败）
	2. 支持CDH6、AWS EMR 6.3、hadoop 3.2 等hadoop集成环境，无hadoop环境的支持有限
	3. 对s3的支持有限，仅在AWS s3环境上做了功能验证，且属于试验阶段，不推荐生产部署
![[s3在kylin使用中的局限性.png]]
2. 计算集群
	1. 目前没有明确的指导文档指出，kylin支持在spark master 为k8s集群下运行，当前仅支持yarn和standalone模式，由于在无hadoop环境下yarn不可得，standalone应用于生产环境还存在诸多问题

3. 集群部署
	1. kylin的集群部署暂未验证

## 参考文献
1. [kylin4支持矩阵](https://cwiki.apache.org/confluence/display/KYLIN/Support+Hadoop+Version+Matrix+of+Kylin+4)
2. [无hadoop环境安装kylin4](https://kylin.apache.org/cn/docs/install/deploy_without_hadoop.html)
3. [kylin的云原生路线](https://blog.51cto.com/u_15458633/4832308)
4. [JuiceFS在kylin上的实践](https://juicefs.com/zh-cn/blog/solutions/optimize-kylin-on-juicefs)