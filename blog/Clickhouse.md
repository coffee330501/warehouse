[官方文档](https://clickhouse.com/docs/en/install)



# 安装

首先，您需要添加官方存储库：

```shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo
```



**安装ClickHouse服务器和客户端**

```shell
sudo yum install -y clickhouse-server clickhouse-client
```



**添加密码**

参考: [Clickhouse安装和配置密码](https://blog.csdn.net/qq_46480020/article/details/124388282)

```shell
echo -n <需要加密的密码> | sha256sum | tr -d '-'
```

```shell
vim /etc/clickhouse-server/users.xml
```

![image-20240312143716453](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240312143716453.png)

注意把原有的 `password` 标签注释掉。



**开启远程访问**

```shell
vim /etc/clickhouse-server/config.xml
```

将 `<listen_host>::</listen_host>` 从注释中打开



**启动ClickHouse服务器**

```shell
sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server
clickhouse-client # or "clickhouse-client --password" if you set up a password.
```



**测试**

> ip:8123

![image-20240312143859518](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240312143859518.png)



>  ip:8123/play

![image-20240312143923120](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240312143923120.png)

# 是什么让 ClickHouse 如此之快

- **面向列的存储：** 源数据通常包含数百甚至数千列，而报表只能使用其中的几个列。**系统需要避免读取不必要的列**，以避免昂贵的磁盘读取操作。
- **索引：** 内存驻留的 ClickHouse 数据结构允许仅读取必要的列，以及这些列的必要的行范围。
- **数据压缩：** 将同一列的不同值存储在一起通常会带来更好的压缩比（与面向行的系统相比），因为在实际数据中，列的相邻行通常具有相同或没有那么多不同的值。除了通用压缩之外，ClickHouse 还支持[专用编解码器](https://clickhouse.com/docs/en/sql-reference/statements/create/table#specialized-codecs)，可以使数据更加紧凑。（人话：相比于一行，一列数据会有更多相似的内容，更好压缩）
- **向量化查询执行：** ClickHouse不仅按列存储数据，还按列处理数据。这会带来更好的 CPU 缓存利用率并允许使用[SIMD](https://en.wikipedia.org/wiki/SIMD) CPU 指令。
- **可扩展性：** ClickHouse 可以**利用所有可用的 CPU 核心和磁盘来执行单个查询**。不仅在单个服务器上，而且在集群的所有 CPU 核心和磁盘上。



# 什么是OLAP？

[OLAP](https://en.wikipedia.org/wiki/Online_analytical_processing)代表在线分析处理。这是一个广泛的术语，可以从两个角度来看待：技术和业务。在最高级别，您可以向后阅读这些单词：

Processing : 处理

Analytical : 分析

Online :在线



所有数据库管理系统都可以分为两类：OLAP（在线**分析**处理）和OLTP（在线**事务**处理）。前者专注于构建报告，每个报告都基于大量历史数据，但频率较低。后者通常处理连续的事务流，不断修改数据的当前状态。



# 使用Canal监听MySQL数据变化

[github](https://github.com/alibaba/canal)

[官方文档](https://github.com/alibaba/canal/wiki/QuickStart)

## 服务端

### 下载

这里下了 `1.1.5` 的版本

```shell
wget https://github.com/alibaba/canal/releases/download/canal-1.1.5/canal.deployer-1.1.5.tar.gz
```





### 修改Canal配置文件

> 不填binlog和possition也可以

查看MySQL binlog文件

```sql
show master status;
```

![image-20240321150557234](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240321150557234.png)



> vim canal.deployer-1.1.5/conf/example/instance.properties

![image-20240321135337298](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240321135337298.png)

```properties
# enable gtid use true/false
canal.instance.gtidon=false

# position info
canal.instance.master.address=121.196.247.62:3306
canal.instance.master.journal.name=binlog.000003
canal.instance.master.position=609798717
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

# username/password
canal.instance.dbUsername=qd-test
canal.instance.dbPassword=Qybh@123
canal.instance.defaultDatabaseName=tnqd
canal.instance.connectionCharset = UTF-8
# enable druid Decrypt database password
canal.instance.enableDruid=false
#canal.instance.pwdPublicKey=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALK4BUxdDltRRE5/zXpVEVPUgunvscYFtEip3pmLlhrWpacX7y7GCMo2/JM6LeHmiiNdH1FWgGCpUfircSwlWKUCAwEAAQ==

# table regex
canal.instance.filter.regex=.*\\..*
# table black regex
canal.instance.filter.black.regex=mysql\\.slave_.*
# table field filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.field=test1.t_product:id/subject/keywords,test2.t_company:id/name/contact/ch
# table field black filter(format: schema1.tableName1:field1/field2,schema2.tableName2:field1/field2)
#canal.instance.filter.black.field=test1.t_product:subject/product_image,test2.t_company:id/name/contact/ch

# mq config
canal.mq.topic=example
# dynamic topic route by schema or table regex
#canal.mq.dynamicTopic=mytest1.user,mytest2\\..*,.*\\..*
canal.mq.partition=0
# hash partition config
#canal.mq.partitionsNum=3
#canal.mq.partitionHash=test.table:id^name,.*\\..*
#canal.mq.dynamicTopicPartitionNum=test.*:4,mycanal:6
#################################################

```



### 启动

```shell
sh bin/startup.sh
```

查看启动日志

```shell
tail -100f logs/canal
```



## 客户端

创建maven项目

### 引入依赖

这里引入 1.1.7 版本是因为我在Windows本地安装了 1.1.7 的 Canal服务端

```xml
<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>1.1.7</version>
</dependency>

<dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.protocol</artifactId>
    <version>1.1.7</version>
</dependency>
```



### 代码

```java
import com.alibaba.fastjson2.JSONObject;
import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry;
import com.alibaba.otter.canal.protocol.Message;
import com.google.protobuf.ByteString;
import com.google.protobuf.InvalidProtocolBufferException;

import java.net.InetSocketAddress;
import java.util.List;
import java.util.Objects;

public class SimpleCanalClientExample {


    public static void main(String args[]) throws InterruptedException, InvalidProtocolBufferException {
        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("127.0.0.1", 11111), "example", "", "");

        while (true) {
            connector.connect();
            // 订阅数据库
            connector.subscribe("tnqd.*");
            // 拿100条数据
            Message message = connector.get(100);

            List<CanalEntry.Entry> entries = message.getEntries();
            if (entries.isEmpty()) {
                System.out.println("无数据...");
                Thread.sleep(1000);
            } else {
                for (CanalEntry.Entry entry : entries) {
                    // 获取表名
                    String tableName = entry.getHeader().getTableName();
                    // 获取类型
                    CanalEntry.EntryType entryType = entry.getEntryType();
                    // 获取序列化后的数据
                    ByteString storeValue = entry.getStoreValue();

                    if (Objects.equals(entryType, CanalEntry.EntryType.ROWDATA)) {
                        // 反序列化数据
                        CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(storeValue);
                        // 事件类型
                        CanalEntry.EventType eventType = rowChange.getEventType();
                        // 数据集
                        List<CanalEntry.RowData> rowDatasList = rowChange.getRowDatasList();
                        for (CanalEntry.RowData rowData : rowDatasList) {
                            JSONObject beforeData = new JSONObject();
                            List<CanalEntry.Column> beforeColumnsList = rowData.getBeforeColumnsList();
                            for (CanalEntry.Column column : beforeColumnsList) {
                                beforeData.put(column.getName(), column.getValue());
                            }

                            JSONObject afterData = new JSONObject();
                            List<CanalEntry.Column> afterColumnsList = rowData.getAfterColumnsList();
                            for (CanalEntry.Column column : afterColumnsList) {
                                afterData.put(column.getName(), column.getValue());
                            }

                            System.out.println("Table: " + tableName + ",EventType: " + eventType + ",Before: " + beforeData + ",After: " + afterData);
                        }
                    } else {
                        System.out.println("当前操作类型为: " + entryType);
                    }
                }
            }
        }
    }
}
```



# Clickhouse结合Canal实现数据同步

