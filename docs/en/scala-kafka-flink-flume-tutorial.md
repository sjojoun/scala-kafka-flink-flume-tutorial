# 📚 Scala + Kafka + Flink + Flume Big Data Real-Time Processing Tutorial

> 🔥 **This tutorial covers the complete big data real-time processing technology stack: Flume Data Collection → Kafka Message Queue → Flink Real-Time Computing → Scala Programming**

---

## Table of Contents

1. [Technology Stack Overview](#1-technology-stack-overview)
2. [Environment Setup](#2-environment-setup)
3. [Flume Data Collection](#3-flume-data-collection)
4. [Kafka Message Queue](#4-kafka-message-queue)
5. [Flink Real-Time Computing](#5-flink-real-time-computing)
6. [Complete Practical Case](#6-complete-practical-case)
7. [Monitoring and Tuning](#7-monitoring-and-tuning)
8. [Troubleshooting](#8-troubleshooting)
9. [Further Reading](#9-further-reading)
10. [Summary](#10-summary)

---

## 1. Technology Stack Overview

| Component | Role | Recommended Version | Official Website |
|-----------|------|-------------------|------------------|
| **Scala** | Programming Language | 2.13+ | https://scala-lang.org |
| **Flume** | Log Collection | 1.11+ | https://flume.apache.org |
| **Kafka** | Message Queue | 3.x | https://kafka.apache.org |
| **Flink** | Stream Processing | 1.18+ | https://flink.apache.org |

### Data Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Flume     │ →  │   Kafka     │ →  │   Flink     │ →  │   Storage   │
│  Collection │    │  Message Q  │    │  Processing │    │ ES/MySQL    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

---

## 2. Environment Setup

### 2.1 System Requirements

- **OS**: Linux (Ubuntu/CentOS) or macOS
- **Memory**: At least 8GB (16GB+ recommended)
- **Disk**: At least 50GB free space
- **JDK**: 11 or 17

### 2.2 JDK Installation

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install openjdk-11-jdk

# CentOS/RHEL
sudo yum install java-11-openjdk

# Verify
java -version
javac -version
```

### 2.3 Scala Installation

```bash
# Method 1: Using SDKMAN (Recommended)
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install scala 2.13.12

# Method 2: Direct Download
wget https://github.com/scala/scala/releases/download/v2.13.12/scala-2.13.12.tgz
sudo tar -xzf scala-2.13.12.tgz -C /opt/
sudo ln -s /opt/scala-2.13.12 /opt/scala

# Verify
scala -version
```

### 2.4 Apache Flume Installation

```bash
# Download
cd /opt
wget https://dlcdn.apache.org/flume/1.11.0/apache-flume-1.11.0-bin.tar.gz

# Extract
sudo tar -xzf apache-flume-1.11.0-bin.tar.gz
sudo ln -s apache-flume-1.11.0-bin flume

# Environment Variables
echo 'export FLUME_HOME=/opt/flume' >> ~/.bashrc
echo 'export PATH=$PATH:$FLUME_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# Verify
flume-ng version
```

### 2.5 Apache Kafka Installation

```bash
# Download
cd /opt
wget https://dlcdn.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz

# Extract
sudo tar -xzf kafka_2.13-3.7.0.tgz
sudo ln -s kafka_2.13-3.7.0 kafka

# Environment Variables
echo 'export KAFKA_HOME=/opt/kafka' >> ~/.bashrc
echo 'export PATH=$PATH:$KAFKA_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# Start ZooKeeper
$KAFKA_HOME/bin/zookeeper-server-start.sh -daemon $KAFKA_HOME/config/zookeeper.properties

# Start Kafka
$KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties

# Verify
$KAFKA_HOME/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### 2.6 Apache Flink Installation

```bash
# Download
cd /opt
wget https://dlcdn.apache.org/flink/flink-1.18.1/flink-1.18.1-bin-scala_2.12.tgz

# Extract
sudo tar -xzf flink-1.18.1-bin-scala_2.12.tgz
sudo ln -s flink-1.18.1 flink

# Environment Variables
echo 'export FLINK_HOME=/opt/flink' >> ~/.bashrc
echo 'export PATH=$PATH:$FLINK_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# Start Cluster
$FLINK_HOME/bin/start-cluster.sh

# Web UI: http://localhost:8081

# Stop Cluster
$FLINK_HOME/bin/stop-cluster.sh
```

---

## 3. Flume Data Collection

### 3.1 Flume Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Source    │ →  │   Channel   │ →  │    Sink     │
│  Data Input │    │   Channel   │    │  Output     │
└─────────────┘    └─────────────┘    └─────────────┘
```

### 3.2 Configuration Example

Create `flume-conf.properties`:

```properties
# Define components
agent1.sources = source1
agent1.channels = channel1
agent1.sinks = sink1

# Source Configuration (Monitor log files)
agent1.sources.source1.type = TAILDIR
agent1.sources.source1.positionFile = /var/log/flume/taildir_position.json
agent1.sources.source1.filegroups = f1
agent1.sources.source1.filegroups.f1 = /var/log/app/*.log

# Channel Configuration (Memory)
agent1.channels.channel1.type = memory
agent1.channels.channel1.capacity = 10000
agent1.channels.channel1.transactionCapacity = 1000

# Sink Configuration (Kafka)
agent1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
agent1.sinks.sink1.kafka.bootstrap.servers = localhost:9092
agent1.sinks.sink1.kafka.topic = flume-logs
agent1.sinks.sink1.kafka.flumeBatchSize = 100

# Bind components
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
```

### 3.3 Start Flume Agent

```bash
flume-ng agent \
  --conf /opt/flume/conf \
  --conf-file /opt/flume/conf/flume-conf.properties \
  --name agent1 \
  -Dflume.root.logger=INFO,console
```

---

## 4. Kafka Message Queue

### 4.1 Create Topics

```bash
# Create log topic
$KAFKA_HOME/bin/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --replication-factor 1 \
  --partitions 3 \
  --topic flume-logs

# Create result topic
$KAFKA_HOME/bin/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --replication-factor 1 \
  --partitions 3 \
  --topic processed-logs

# List topics
$KAFKA_HOME/bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# Describe topic
$KAFKA_HOME/bin/kafka-topics.sh --describe \
  --topic flume-logs \
  --bootstrap-server localhost:9092
```

---

## 5. Flink Real-Time Computing

### 5.1 Maven Project Configuration

Create `pom.xml`:

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

### 5.2 Flink Streaming Example

Create `src/main/scala/com/example/FlinkStreamingJob.scala`:

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
 * Flink Real-Time Log Processing Example
 * Read from Kafka → Filter → Window Aggregation → Write to Kafka
 */
object FlinkStreamingJob {
  
  def main(args: Array[String]): Unit = {
    // Create stream processing environment
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(2)
    
    // Configure Kafka Source
    val kafkaSource = KafkaSource.builder[String]()
      .setBootstrapServers("localhost:9092")
      .setTopics("flume-logs")
      .setGroupId("flink-consumer-group")
      .setValueOnlyDeserializer(new SimpleStringSchema())
      .build()
    
    // Read from Kafka
    val logStream = env.fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), "Kafka Source")
    
    // Filter empty lines
    val filteredStream = logStream.filter(new FilterFunction[String] {
      override def filter(value: String): Boolean = {
        value != null && value.trim.nonEmpty
      }
    })
    
    // Add timestamp
    val mappedStream = filteredStream.map(value => (value, System.currentTimeMillis()))
    
    // Window aggregation: every 5 seconds
    val windowedStream = mappedStream
      .keyBy(_._1)
      .window(TumblingEventTimeWindows.of(Time.seconds(5)))
      .count()
    
    // Configure Kafka Sink
    val kafkaSink = KafkaSink.builder[String]()
      .setBootstrapServers("localhost:9092")
      .setRecordSerializer(KafkaRecordSerializationSchema.builder()
        .setTopic("processed-logs")
        .setValueSerializationSchema(new SimpleStringSchema())
        .build())
      .build()
    
    // Write to Kafka
    windowedStream.map(t => s"Count: ${t._2}, Key: ${t._1}")
      .sinkTo(kafkaSink)
    
    // Print to console (debug)
    windowedStream.print()
    
    // Execute job
    env.execute("Flink Real-time Log Processing")
  }
}
```

### 5.3 Build and Submit

```bash
# Build project
mvn clean package

# Submit Flink job
$FLINK_HOME/bin/flink run \
  -c com.example.FlinkStreamingJob \
  ./target/flink-streaming-1.0-SNAPSHOT.jar

# Check job status
$FLINK_HOME/bin/flink list --running

# Cancel job
$FLINK_HOME/bin/flink cancel <job_id>
```

---

## 6. Complete Practical Case

### 6.1 System Architecture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   App Server │ →  │    Flume     │ →  │    Kafka     │ →  │    Flink     │
│  Log Output  │    │  Collection  │    │  Message Q   │    │  Processing  │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                    ↓
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Visualization│ ←  │ Elasticsearch│ ←  │   Output     │
│  Grafana     │    │   Storage    │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
```

### 6.2 Log Format Example (JSON)

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

### 6.3 Advanced Flink Processing with Alerting

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
 * Real-time Log Analysis with Error Detection and Alerting
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
    
    implicit val formats = DefaultFormats
    
    // Read from Kafka
    val kafkaSource = KafkaSource.builder[String]()
      .setBootstrapServers("localhost:9092")
      .setTopics("flume-logs")
      .setGroupId("log-analysis-group")
      .setValueOnlyDeserializer(new SimpleStringSchema())
      .build()
    
    val logStream = env.fromSource(kafkaSource, 
      WatermarkStrategy.forBoundedOutOfOrderness(java.time.Duration.ofSeconds(5)), 
      "Kafka Source")
    
    // Parse JSON
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
    
    // Filter ERROR logs
    val errorStream = parsedStream.filter(_.level == "ERROR")
    
    // Detect consecutive errors using state
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
          
          // Alert if more than 5 errors in 1 minute
          if (newCount >= 5) {
            out.collect(s"🚨 ALERT: Service ${value.service} has $newCount errors! Last: ${value.message}")
            errorCount.update(0)  // Reset counter
          }
        }
      })
    
    // Print alerts
    alertStream.print("ALERT")
    
    // Execute
    env.execute("Real-time Log Analysis with Alerting")
  }
}
```

---

## 7. Monitoring and Tuning

### 7.1 Flink Web UI

Access: http://localhost:8081

**Monitor:**
- Job running status
- Task throughput
- Backpressure
- Checkpoint status

### 7.2 Kafka Monitoring

```bash
# List consumer groups
$KAFKA_HOME/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list

# Describe consumer group
$KAFKA_HOME/bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group flink-consumer-group
```

### 7.3 Performance Tuning

**Flink Tuning:**
```scala
// Increase parallelism
env.setParallelism(4)

// Enable Checkpoint
val checkpointConfig = env.getCheckpointConfig
checkpointConfig.setCheckpointInterval(60000)  // 1 minute
checkpointConfig.setMinPauseBetweenCheckpoints(30000)

// Configure State Backend
val stateBackend = new RocksDBStateBackend("hdfs://namenode:8020/flink/checkpoints")
env.setStateBackend(stateBackend)
```

**Kafka Tuning:**
```properties
# server.properties
num.partitions=6
default.replication.factor=3
log.retention.hours=168
```

**Flume Tuning:**
```properties
# Increase Channel capacity
agent1.channels.channel1.capacity = 100000
agent1.channels.channel1.transactionCapacity = 10000

# Use File Channel (more reliable)
agent1.channels.channel1.type = file
agent1.channels.channel1.checkpointDir = /var/flume/checkpoint
agent1.channels.channel1.dataDirs = /var/flume/data
```

---

## 8. Troubleshooting

### 8.1 Flume Issues

**Problem:** Flume cannot read log files
```bash
# Check file permissions
ls -la /var/log/app/

# Check Flume configuration
flume-ng config --conf /opt/flume/conf \
  --conf-file /opt/flume/conf/flume-conf.properties \
  --name agent1
```

### 8.2 Kafka Issues

**Problem:** Cannot connect to Kafka
```bash
# Check if Kafka is running
ps aux | grep kafka

# Check Kafka logs
tail -f $KAFKA_HOME/logs/server.log

# Test connection
$KAFKA_HOME/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

### 8.3 Flink Issues

**Problem:** Job submission fails
```bash
# Check Flink cluster
$FLINK_HOME/bin/flink list --running

# Check JobManager logs
tail -f $FLINK_HOME/log/flink-*-jobmanager-*.log

# Check TaskManager logs
tail -f $FLINK_HOME/log/flink-*-taskexecutor-*.log
```

---

## 9. Further Reading

### Official Documentation
- [Scala Documentation](https://docs.scala-lang.org/)
- [Apache Flume Documentation](https://flume.apache.org/releases/content/1.11.0/FlumeUserGuide.html)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Apache Flink Documentation](https://flink.apache.org/documentation/)

### Advanced Topics
- Flink CEP (Complex Event Processing)
- Flink SQL and Table API
- Kafka Streams
- Exactly-Once Semantics
- State Management and Checkpointing

### Deployment Options
- Docker Containerization
- Kubernetes Cluster Deployment
- Cloud Managed Services (AWS MSK, Confluent Cloud)

---

## 10. Summary

This tutorial covers the complete data pipeline from collection to real-time processing:

✅ **Flume** - Reliable log collection system
✅ **Kafka** - High-throughput message queue
✅ **Flink** - Powerful stream processing engine
✅ **Scala** - Concise and efficient programming language

**Next Steps:**
1. Deploy and run examples in test environment
2. Adjust configuration based on business requirements
3. Add monitoring and alerting
4. Gradually optimize performance parameters

---

*Tutorial Version: 1.0 | Last Updated: 2026-03-09 | Author: OpenClaw*
