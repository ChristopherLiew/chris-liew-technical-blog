# Spark Core

## What is Spark

Apache spark is an open sourced fault tolerant data processing framework for Big Data workloads which unifies:

1. Batch processing
2. Real-Time / Stream processing
3. Machine learning
4. Graph computation

Spark is largely used as a distributed batch computation engine that explciitly handles an entire workflow as a single job (hence a *dataflow* engine like *Tez* and *Flink*)

### Overview of APIs

:::mermaid
graph LR
    SparkSQL ;
    SparkEngine ;
    YARN ;
:::

## Why Spark

### Optimizations over MapReduce

1. Sorting only needs to be performed when it is actually required rather than by default between every Map and Reduce stage
2. No unnecessary map tasks - Map tasks can be incorporated into a preceding reduce operator
3. Locality optimizations since all joins and data dependencies in a workflow are explicitly declared - Tasks that consume the same data can be placed on the same machine to reduce network overhead
4. Intermediate state between operators can be kept in memory or written to local disk to reduce I/O to HDFS
5. Operators can execute once input is ready without waiting for the preceding stage to complete
6. Reduce startup overhead by reusing existing JVM processes to run new operators

### Fault Tolerance

Spark enables fault tolerance by

1. Tracking RDD block creation process to rebuild a dataset when a partition fails
2. Uses DAGs to rebuild data flow across worker nodes upon failure

## Architecture

::: mermaid
graph TD
    A --> B;
    A --> C;
    B --> D;
    C --> D;
:::

### Driver

The spark driver unsurprisingly acts as the "driver" your Spark App. The driver:

1. Controls the execution of the Spark application
2. Maintains all states of the Spark cluster (including its executors)
3. Interfaces with the Cluster Manager to negotiate for physical resources (i.e. Memory and Cores) to launch executors

#### Spark Context, Session and SQL Context (WIP)

Within the Driver Program sits a Spark Context or Session (Spark 2.0 onwards). The Context is used by the Driver to establish communication with the Cluster and Resource managers to coordinate and execute jobs.

An instance of a Spark Context / Session must be instantiated in every application. For example in Spark 2.0 we can do the following when executing locally:

```scala
import org.apache.spark.sql.SparkSession

val spark = {
    SparkSession.builder()
    .appName("Spark101")
    .config("spark.master", "local")
    .getOrCreate()
  }
```

<br>

| Spark Context | Spark Session | SQL Context|
| --- | --- | --- |
| TBD | TBD | TBD |

### Spark Executors

Spark executors are processes that perform tasks assigned by the Driver process. Executors simply take tasks assigned by the Driver, run them and report back with the state and results (i.e. Success or Failure)

### Cluster Manager

Cluster managers allocate physicaly resources (E.g. RAM and Cores) to our Spark Application based on its needs. Some common resource managers are detailed below:

#### YARN and Mesos

#### K8s

<br>

## Execution Modes

| Cluster Mode | Client Mode | Local Mode |
| --- | --- | --- |
| Submit JAR or Script to Cluster Manager which then launches a Driver Process on a Worker Node within the Cluster. | Same as Cluster mode except Driver reamins on the Client machine that submitted the Spark Application | Complete departure from Cluster and Client modes. Runs entire Spark Application on a Single machine |
| Cluster Manager is reponsible for maintaining Spark Applications. | Client machine (Gateway machine or Edge node) is responsible for maintaining Driver process whilst Cluster manages Executor processes. | Achieves Parallelism through multithreading |

<br>

## Spark APIs Overview

### Languages

::: mermaid
graph TD
    Python-API
    ---py4j--> Scala/Java-API
    --> Spark-Core
:::

<br>

### RDDs vs DataSets vs DataFrames vs SQL

| | RDD | DataSets | DataFrames | SQL |
| --- | --- | --- | --- | --- |
| Usecase | | | |
| Features | | | |
| Syntax Errors | | Compile Time | Compile Time | Runtime |
| Analysis Errors | | Compile Time | Runtime | Runtime |

## Spark Join Strategies

### 1. Broadcast Hash Join

### 2. Shuffle Hash Join

### 3. Shuffle Sort-Merge Join

Default join strategy in Spark

### How to Select a Spark Join Strategy


## Memory Allocation & Management

## Performance Tuning & Optimisation Basics

## Spark SQL FTW?
