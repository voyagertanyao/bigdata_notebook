# [基本概念](https://parquet.apache.org/docs/concepts/)
- file
- row group：一个parquet文件在逻辑上被按行分隔为多个row group，`MapReduce并发操作的基本单元`
- column chunk：每个row group中，每一列都构成一个column chunk，`IO的基本单元`
- page：每个column chunk中，划分多个page，`encoding/compression的基本单元`

---------

# 基本结构图

### 一、逻辑结构图

![[parquet.png]]


###  二、物理结构图


![[parquet physical.png]]


### 三、数据结构


![[parquet structure.png]]


------------------

# 编码与压缩

When a Parquet file is written, the column data is assembled into pages, which are the unit for encoding and compression. Parquet provides a variety of encoding mechanisms, including plain, dictionary encoding, run-length encoding (RLE), delta encoding, and byte split encoding. Most encodings are in favor of continuous zeros because that can improve encoding efficiency. After encoding the compression will make the data size smaller without losing information. There are several compression methods in Parquet, including **SNAPPY**, GZIP, LZO, BROTLI, LZ4, and ZSTD. Generally, choosing the right compression method is a trade-off between compression ratio and speed for reading and writing.


# 列加密([spark on parquet](https://spark.apache.org/docs/latest/sql-data-sources-parquet.html))

```scala
spark.sparkContext.hadoopConfiguration.set("parquet.encryption.kms.client.class","org.apache.parquet.crypto.keytools.mocks.InMemoryKMS")
spark.sparkContext.hadoopConfiguration.set("parquet.encryption.key.list","keyA:AAECAwQFBgcICQoLDA0ODw== ,  keyB:AAECAAECAAECAAECAAECAA==")
spark.sparkContext.hadoopConfiguration.set("parquet.crypto.factory.class","org.apache.parquet.crypto.keytools.PropertiesDrivenCryptoFactory")

# 写入文件
spark.sql("select id,name,phone,age from bigdata.user_info").repartition(1).write.option("parquet.encryption.column.keys" , "keyA:id,name,phone").option("parquet.encryption.footer.key" , "keyB").parquet("s3a://metastore/user.parquet.encrypted")

# 读取文件
spark.sql("select id,name,phone,age from bigdata.user_info").repartition(1).write.option("parquet.encryption.column.keys" , "keyA:id,name,phone").option("parquet.encryption.footer.key" , "keyB").parquet("user.parquet.encrypted")
```