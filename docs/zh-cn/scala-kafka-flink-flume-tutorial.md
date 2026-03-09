# 📚 Scala + Kafka + Flink + Flume 大数据实时处理教程

> 🔥 **本教程涵盖大数据实时处理完整技术栈：Flume 数据采集 → Kafka 消息队列 → Flink 实时计算 → Scala 编程**

---

## 目录

1. [技术栈概览](#1-技术栈概览)
2. [环境准备](#2-环境准备)
3. [Flume 数据采集](#3-flume-数据采集)
4. [Kafka 消息队列](#4-kafka-消息队列)
5. [Flink 实时计算](#5-flink-实时计算)
6. [完整实战案例](#6-完整实战案例)
7. [监控和调优](#7-监控和调优)
8. [常见问题排查](#8-常见问题排查)
9. [扩展阅读](#9-扩展阅读)
10. [总结](#10-总结)

---

## 1. 技术栈概览

| 组件 | 作用 | 推荐版本 | 官网 |
|------|------|---------|------|
| **Scala** | 编程语言 | 2.13+ | https://scala-lang.org |
| **Flume** | 日志采集 | 1.11+ | https://flume.apache.org |
| **Kafka** | 消息队列 | 3.x | https://kafka.apache.org |
| **Flink** | 流式计算 | 1.18+ | https://flink.apache.org |

### 数据流向

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Flume     │ →  │   Kafka     │ →  │   Flink     │ →  │  结果存储   │
│  日志采集   │    │  消息队列   │    │  实时计算   │    │ ES/MySQL 等  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

---

## 2. 环境准备

### 2.1 系统要求

- **操作系统**: Linux (Ubuntu/CentOS) 或 macOS
- **内存**: 至少 8GB (建议 16GB+)
- **磁盘**: 至少 50GB 可用空间
- **JDK**: 11 或 17

### 2.2 JDK 安装

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install openjdk-11-jdk

# CentOS/RHEL
sudo yum install java-11-openjdk

# 验证安装
java -version
javac -version
```

### 2.3 Scala 安装

```bash
# 方法 1: 使用 SDKMAN (推荐)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install scala 2.13.12

# 方法 2: 直接下载
wget https://github.com/scala/scala/releases/download/v2.13.12/scala-2.13.12.tgz
sudo tar -xzf scala-2.13.12.tgz -C /opt/
sudo ln -s /opt/scala-2.13.12 /opt/scala

# 验证安装
scala -version
```

### 2.4 Apache Flume 安装

```bash
# 下载
cd /opt
wget https://dlcdn.apache.org/flume/1.11.0/apache-flume-1.11.0-bin.tar.gz

# 解压
sudo tar -xzf apache-flume-1.11.0-bin.tar.gz
sudo ln -s apache-flume-1.11.0-bin flume

# 配置环境变量
echo 'export FLUME_HOME=/opt/flume' >> ~/.bashrc
echo 'export PATH=$PATH:$FLUME_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# 验证安装
flume-ng version
```

### 2.5 Apache Kafka 安装

```bash
# 下载
cd /opt
wget https://dlcdn.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz

# 解压
sudo tar -xzf kafka_2.13-3.7.0.tgz
sudo ln -s kafka_2.13-3.7.0 kafka

# 配置环境变量
echo 'export KAFKA_HOME=/opt/kafka' >> ~/.bashrc
echo 'export PATH=$PATH:$KAFKA_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# 启动 ZooKeeper (Kafka 依赖)
$KAFKA_HOME/bin/zookeeper-server-start.sh -daemon $KAFKA_HOME/config/zookeeper.properties

# 启动 Kafka
$KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties

# 验证
$KAFKA_HOME/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### 2.6 Apache Flink 安装

```bash
# 下载
cd /opt
wget https://dlcdn.apache.org/flink/flink-1.18.1/flink-1.18.1-bin-scala_2.12.tgz

# 解压
sudo tar -xzf flink-1.18.1-bin-scala_2.12.tgz
sudo ln -s flink-1.18.1 flink

# 配置环境变量
echo 'export FLINK_HOME=/opt/flink' >> ~/.bashrc
echo 'export PATH=$PATH:$FLINK_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# 启动 Flink 集群
$FLINK_HOME/bin/start-cluster.sh

# 验证 (访问 Web UI)
# http://localhost:8081

# 停止集群
$FLINK_HOME/bin/stop-cluster.sh
```

---

## 3. Flume 数据采集

### 3.1 Flume 架构

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Source    │ →  │   Channel   │ →  │    Sink     │
│  数据源     │    │   通道      │    │  输出目标   │
└─────────────┘    └─────────────┘    └─────────────┘
```

### 3.2 配置文件示例

创建 `flume-conf.properties`:

```properties
# 定义组件
agent1.sources = source1
agent1.channels = channel1
agent1.sinks = sink1

# Source 配置 (监控日志文件)
agent1.sources.source1.type = TAILDIR
agent1.sources.source1.positionFile = /var/log/flume/taildir_position.json
agent1.sources.source1.filegroups = f1
agent1.sources.source1.filegroups.f1 = /var/log/app/*.log

# Channel 配置 (内存通道)
agent1.channels.channel1.type = memory
agent1.channels.channel1.capacity = 10000
agent1.channels.channel1.transactionCapacity = 1000

# Sink 配置 (Kafka)
agent1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
agent1.sinks.sink1.kafka.bootstrap.servers = localhost:9092
agent1.sinks.sink1.kafka.topic = flume-logs
agent1.sinks.sink1.kafka.flumeBatchSize = 100

# 绑定组件
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
```

### 3.3 启动 Flume Agent

```bash
flume-ng agent \
  --conf /opt/flume/conf \
  --conf-file /opt/flume/conf/flume-conf.properties \
  --name agent1 \
  -Dflume.root.logger=INFO,console
```

### 3.4 测试数据采集

```bash
# 在另一个终端生成测试日志
echo "Test log entry 1" >> /var/log/app/test.log
echo "Test log entry 2" >> /var/log/app/test.log

# 查看 Kafka 中的消息
$KAFKA_HOME/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic flume-logs \
  --from-beginning
```

---

## 4. Kafka 消息队列

### 4.1 创建 Topic

```bash
# 创建日志主题
$KAFKA_HOME/bin/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --replication-factor 1 \
  --partitions 3 \
  --topic flume-logs

# 创建处理结果主题
$KAFKA_HOME/bin/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --replication-factor 1 \
  --partitions 3 \
  --topic processed-logs

# 查看主题列表
$KAFKA_HOME/bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# 查看主题详情
$KAFKA_HOME/bin/kafka-topics.sh --describe \
  --topic flume-logs \
  --bootstrap-server localhost:9092
```

### 4.2 生产者和消费者测试

```bash
# 启动生产者
$KAFKA_HOME/bin/kafka-console-producer.sh \
  --broker-list localhost:9092 \
  --topic test-topic

# 启动消费者
$KAFKA_HOME/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning
```

---

## 5. Flink 实时计算

### 5.1 Maven 项目配置

创建 `pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>flink-streaming</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <flink.version>1.18.1</flink.version>
        <scala.version>2.13.12</scala.version>
        <kafka.version>3.7.0</kafka.version>
    </properties>

    <dependencies>
        <!-- Flink Core -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.13</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <!-- Flink Kafka Connector -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_2.13</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <!-- Flink JSON -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-json</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <!-- Flink CEP (复杂事件处理) -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-cep-scala_2.13</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <!-- Logger -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.9</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>11</source>
                    <target>11</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>4.8.1</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.5.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.example.FlinkStreamingJob</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### 5.2 Flink Streaming 示例代码

创建 `src/main/scala/com/example/FlinkStreamingJob.scala`:

```scala
package com.example

import org.apache.flink.api.common.functions.FilterFunction
import org.apache.flink.api.common.serialization.SimpleStringSchema
import org.apache.flink.connector.kafka.sink.{KafkaRecordSerializationSchema, KafkaSink}
import org.apache.flink.connector.kafka.source.KafkaSource
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows
import org.apache.flink.streaming.api.windowing.time.Time

/**
 * Flink 实时日志处理示例
 * 从 Kafka 读取 Flume 采集的日志 → 过滤 → 窗口聚合 → 写回 Kafka
 */
object FlinkStreamingJob {
  
  def main(args: Array[String]): Unit = {
    // 创建流处理环境
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(2)
    
    // 配置 Kafka Source
    val kafkaSource = KafkaSource.builder[String]()
      .setBootstrapServers("localhost:9092")
      .setTopics("flume-logs")
      .setGroupId("flink-consumer-group")
      .setValueOnlyDeserializer(new SimpleStringSchema())
      .build()
    
    // 读取 Kafka 数据
    val logStream = env.fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), "Kafka Source")
    
    // 数据转换：过滤空行
    val filteredStream = logStream.filter(new FilterFunction[String] {
      override def filter(value: String): Boolean = {
        value != null && value.trim.nonEmpty
      }
    })
    
    // 数据转换：添加时间戳
    val mappedStream = filteredStream.map(value => (value, System.currentTimeMillis()))
    
    // 窗口聚合：每 5 秒统计一次
    val windowedStream = mappedStream
      .keyBy(_._1)
      .window(TumblingEventTimeWindows.of(Time.seconds(5)))
      .count()
    
    // 配置 Kafka Sink
    val kafkaSink = KafkaSink.builder[String]()
      .setBootstrapServers("localhost:9092")
      .setRecordSerializer(KafkaRecordSerializationSchema.builder()
        .setTopic("processed-logs")
        .setValueSerializationSchema(new SimpleStringSchema())
        .build())
      .build()
    
    // 写入 Kafka
    windowedStream.map(t => s"Count: ${t._2}, Key: ${t._1}")
      .sinkTo(kafkaSink)
    
    // 打印到控制台 (调试用)
    windowedStream.print()
    
    // 执行作业
    env.execute("Flink Real-time Log Processing")
  }
}
```

### 5.3 编译和提交作业

```bash
# 编译项目
mvn clean package

# 提交 Flink 作业
$FLINK_HOME/bin/flink run \
  -c com.example.FlinkStreamingJob \
  ./target/flink-streaming-1.0-SNAPSHOT.jar

# 查看作业状态
$FLINK_HOME/bin/flink list --running

# 取消作业
$FLINK_HOME/bin/flink cancel <job_id>
```

---

## 6. 完整实战案例：实时日志分析系统

### 6.1 系统架构

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  应用服务器   │    │    Flume     │    │    Kafka     │    │    Flink     │
│  生成日志    │ →  │   采集日志   │ →  │  消息队列    │ →  │  实时计算    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                          ↓
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  数据可视化   │ ←  │   Elasticsearch │ ←  │  结果输出    │
│  Grafana 等  │    │   存储索引    │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
```

### 6.2 日志格式示例

假设应用日志格式为 JSON:

```json
{
  "timestamp": "2026-03-09T09:00:00Z",
  "level": "ERROR",
  "service": "user-service",
  "message": "Database connection timeout",
  "userId": "12345",
  "duration": 5000
}
```

### 6.3 高级 Flink 处理代码

```scala
package com.example

import org.apache.flink.api.common.eventtime.{SerializableTimestampAssigner, WatermarkStrategy}
import org.apache.flink.api.common.functions.RichFlatMapFunction
import org.apache.flink.api.common.state.{ValueState, ValueStateDescriptor}
import org.apache.flink.configuration.Configuration
import org.apache.flink.streaming.api.scala._
import org.apache.flink.util.Collector
import org.json4s._
import org.json4s.jackson.JsonMethods._

/**
 * 实时日志分析 - 错误检测和告警
 */
object LogAnalysisJob {
  
  case class LogEvent(
    timestamp: String,
    level: String,
    service: String,
    message: String,
    userId: String,
    duration: Long
  )
  
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(3)
    
    // 隐式转换
    implicit val formats = DefaultFormats
    
    // 读取 Kafka 数据
    val kafkaSource = KafkaSource.builder[String]()
      .setBootstrapServers("localhost:9092")
      .setTopics("flume-logs")
      .setGroupId("log-analysis-group")
      .setValueOnlyDeserializer(new SimpleStringSchema())
      .build()
    
    val logStream = env.fromSource(kafkaSource, 
      WatermarkStrategy.forBoundedOutOfOrderness(java.time.Duration.ofSeconds(5)), 
      "Kafka Source")
    
    // 解析 JSON
    val parsedStream = logStream.map { json =>
      val parsed = parse(json)
      LogEvent(
        timestamp = (parsed \ "timestamp").extract[String],
        level = (parsed \ "level").extract[String],
        service = (parsed \ "service").extract[String],
        message = (parsed \ "message").extract[String],
        userId = (parsed \ "userId").extract[String],
        duration = (parsed \ "duration").extract[Long]
      )
    }
    
    // 过滤错误日志
    val errorStream = parsedStream.filter(_.level == "ERROR")
    
    // 使用状态检测连续错误
    val alertStream = errorStream
      .keyBy(_.service)
      .flatMap(new RichFlatMapFunction[LogEvent, String] {
        private var errorCount: ValueState[Int] = _
        
        override def open(parameters: Configuration): Unit = {
          errorCount = getRuntimeContext.getState(
            new ValueStateDescriptor[Int]("errorCount", classOf[Int])
          )
        }
        
        override def flatMap(value: LogEvent, out: Collector[String]): Unit = {
          val currentCount = Option(errorCount.value()).getOrElse(0)
          val newCount = currentCount + 1
          errorCount.update(newCount)
          
          // 如果 1 分钟内错误超过 5 次，发送告警
          if (newCount >= 5) {
            out.collect(s"🚨 ALERT: Service ${value.service} has $newCount errors! Last: ${value.message}")
            errorCount.update(0)  // 重置计数器
          }
        }
      })
    
    // 打印告警
    alertStream.print("ALERT")
    
    // 执行
    env.execute("Real-time Log Analysis with Alerting")
  }
}
```

---

## 7. 监控和调优

### 7.1 Flink Web UI

访问：http://localhost:8081

**监控指标:**
- 作业运行状态
- Task 吞吐量
- 反压情况
- Checkpoint 状态

### 7.2 Kafka 监控

```bash
# 查看消费者组
$KAFKA_HOME/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list

# 查看消费者组详情
$KAFKA_HOME/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group flink-consumer-group
```

### 7.3 性能调优建议

**Flink 调优:**
```scala
// 增加并行度
env.setParallelism(4)

// 启用 Checkpoint
val checkpointConfig = env.getCheckpointConfig
checkpointConfig.setCheckpointInterval(60000)  // 1 分钟
checkpointConfig.setMinPauseBetweenCheckpoints(30000)

// 配置状态后端
val stateBackend = new RocksDBStateBackend("hdfs://namenode:8020/flink/checkpoints")
env.setStateBackend(stateBackend)
```

**Kafka 调优:**
```properties
# server.properties
num.partitions=6
default.replication.factor=3
log.retention.hours=168
```

**Flume 调优:**
```properties
# 增加 Channel 容量
agent1.channels.channel1.capacity = 100000
agent1.channels.channel1.transactionCapacity = 10000

# 使用 File Channel (更可靠)
agent1.channels.channel1.type = file
agent1.channels.channel1.checkpointDir = /var/flume/checkpoint
agent1.channels.channel1.dataDirs = /var/flume/data
```

---

## 8. 常见问题排查

### 8.1 Flume 问题

**问题:** Flume 无法读取日志文件
```bash
# 检查文件权限
ls -la /var/log/app/

# 检查 Flume 配置
flume-ng config --conf /opt/flume/conf \
  --conf-file /opt/flume/conf/flume-conf.properties \
  --name agent1
```

### 8.2 Kafka 问题

**问题:** 无法连接 Kafka
```bash
# 检查 Kafka 是否运行
ps aux | grep kafka

# 查看 Kafka 日志
tail -f $KAFKA_HOME/logs/server.log

# 测试连接
$KAFKA_HOME/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

### 8.3 Flink 问题

**问题:** 作业提交失败
```bash
# 检查 Flink 集群
$FLINK_HOME/bin/flink list --running

# 查看 JobManager 日志
tail -f $FLINK_HOME/log/flink-*-jobmanager-*.log

# 查看 TaskManager 日志
tail -f $FLINK_HOME/log/flink-*-taskexecutor-*.log
```

---

## 9. 扩展阅读

### 官方文档
- [Scala 官方文档](https://docs.scala-lang.org/)
- [Apache Flume 文档](https://flume.apache.org/releases/content/1.11.0/FlumeUserGuide.html)
- [Apache Kafka 文档](https://kafka.apache.org/documentation/)
- [Apache Flink 文档](https://flink.apache.org/documentation/)

### 进阶主题
- Flink CEP 复杂事件处理
- Flink SQL 和 Table API
- Kafka Streams
- Exactly-Once 语义实现
- 状态管理和 Checkpoint 机制

### 部署方案
- Docker 容器化部署
- Kubernetes 集群部署
- 云上托管 (AWS MSK, Confluent Cloud)

---

## 10. 总结

本教程涵盖了从数据采集到实时处理的完整流程：

✅ **Flume** - 可靠的日志采集系统
✅ **Kafka** - 高吞吐消息队列
✅ **Flink** - 强大的流式计算引擎
✅ **Scala** - 简洁高效的编程语言

**下一步建议:**
1. 在测试环境完整部署并运行示例
2. 根据实际业务需求调整配置
3. 添加监控告警系统
4. 逐步优化性能参数

---

*教程版本：1.0 | 更新时间：2026-03-09 | 作者：OpenClaw*
