## 特性

* 与MapReduce不同，Spark的数据处理工作全部在内存中进行，只在读入和保存最终结果读写存储。
* Spark囊括了离线批处理、流处理、即时查询和图计算4大功能
  * 离线批处理替代 Hadoop MapReduce
  * 流处理替代 Apache Storm
  * 即时查询替代 Impala 或者 Tez
  * 图计算替代 Neo4j 或者 Apache Giraph

## 核心组件

* Spark core：实现spark的基本功能：任务调度，内存管理，错误恢复，存储交互，API等模块

- Spark Sql：Spark操作结构化数据的程序包，可使用Sql或HSql查询数据，支持多种数据源
- Spark Streaming：实时流计算组件，API

- MLlib：机器学习库,丰富强大
- Graphx：图计算,图像算法
- SparkR：SparkR 是一个 R 语言包
- 集群管理器：类似于Hadoop的Yarn，从单节点到上千节点伸缩计算，简易调度器

## 数据集

### 分布式数据集

- RDD（Resilient Distributed Dataset）：分布式弹性数据集，只读，具有内存计算能力，数据容错性。RDD 支持两种操作：
  - 转换(transformations)：从已存在数据集中通过转换创建新数据集，是惰性的，如：map
  - 动作(actions)：在数据集上进行计算之后聚合数据返回，如：reduce 返回聚合结果，reduceByKey 返回分布式数据集。
- Dataframe：分布式数据集合，只读，可按列查询，类似于数据库的表。可以对数据指定数据模式（schema）。
- Dataset：Dataframe扩展，提供类型安全、面向对象的编程接口。DataFrame是Dataset的一种特殊形式。

#### 三者区别

* 都有惰性机制，创建、转换（如map）时，不会立即执行，只有在遇到Action如foreach时，三者才会开始遍历运算。极端情况如果只有创建没有Action中使用结果，会被直接跳过。

* 自动缓存运算，不用担心内存溢出。

* 都有partition概念。

* 有许多共同的函数，如filter，排序等。

* 对DataFrame和Dataset进行操作需要这个包：```import spark.implicits._```

* DataFrame和Dataset均可使用模式匹配获取各个字段的值和类型

* ```java
  # Dataframe
  testDF.map{
        case Row(col1:String,col2:Int)=>
          println(col1);println(col2)
          col1
        case _=>
          ""
      }
  
  # Dataset
  case class Coltest(col1:String,col2:Int)extends Serializable //定义字段名和类型
      testDS.map{
        case Coltest(col1:String,col2:Int)=>
          println(col1);println(col2)
          col1
        case _=>
          ""
      }
  ```

* RDD一般和Spark MLlib同时使用，DataFrame与Dataset一般与Spark ML同时使用

* RDD不支持SparkSql操作，RDD是分布式的 Java对象的集合，所以SparkSql不知道对象里的数据结构，而DataFrame是有schema，所以SparkSql知道详细数据结构。

* DataFrame是分布式的Row对象的集合。

* Dataset是DataFrame的特例，record存储的是一个强类型值而不是一个Row。

* ```java
  #DataFrame 每一列的值没法直接访问，需要解释获取
  testDF.foreach{
    line =>
      val col1=line.getAs[String]("col1")
      val col2=line.getAs[String]("col2")
  }
  ```

* DataFrame与Dataset支持支持带header的csv保存。

* ```java
  //保存
  val woptions = Map("header" -> "true", "delimiter" -> "\t", "path" -> "hdfs://172.xx:9000/test")
  datawDF.write.format("com.databricks.spark.csv").mode(SaveMode.Overwrite).options(woptions).save()
  //读取
  val roptions = Map("header" -> "true", "delimiter" -> "\t", "path" -> "hdfs://172.xx.xx.xx:9000/test")
  val datarDF= spark.read.options(roptions).format("com.databricks.spark.csv").load()
  ```

* DataFrame等于Dataset[Row]，Dataset[指定转换class]后使用比较方便，但是如果类型很多不好转换就用Dataset[Row]。

#### 转换

```java
// DataFrame/Dataset 转 RDD
val rdd1=testDF.rdd
val rdd2=testDS.rdd

// RDD 转 DataFrame
import spark.implicits._
val testDF = rdd.map {line=>
      (line._1,line._2)		// 一般用元组把一行的数据写在一起
    }.toDF("col1","col2") // 然后在toDF中指定字段名

// RDD 转 Dataset
import spark.implicits._
case class Coltest(col1:String,col2:Int)extends Serializable //定义字段名和类型
val testDS = rdd.map {line=>
      Coltest(line._1,line._2) // 填充值
    }.toDS

// Dataset 转 DataFrame，把case class封装成Row
import spark.implicits._
val testDF = testDS.toDF

// DataFrame 转 Dataset，Row转case class
import spark.implicits._
case class Coltest(col1:String,col2:Int)extends Serializable //定义字段名和类型
val testDS = testDF.as[Coltest]
```

#### 缓存

通过 ```cache``` 和 ```persist``` 把PDD放置到内存中，加速后面使用，cache是缓存，persist是持久化可以通过StorageLevel设置缓存到磁盘。可以通过 ```unpersist``` 进行释放。

```java
val lines = sc.textFile("data.txt")
val lineLengths = lines.map(s => s.length)
lineLengths.persist() // 这样这个结果就可以重复使用
val totalLength = lineLengths.reduce((a, b) => a + b)
```

#### 算子

##### 转换算子

| Transformation算子                                   | 含义                                                         |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| map(func)                                            | 返回新RDD，每个元素都是由源RDD中元素经func转换得到。         |
| filter(func)                                         | 返回新RDD，元素来自源RDD中元素经func过滤后（func返回true选中）结果 |
| flatMap(func)                                        | 类似于map，每个输入元素映射到0到n个输出元素（所以要求func必须返回一个Seq而不是单个元素） |
| mapPartitions(func)                                  | 类似于map，但基于每个RDD分区（或者数据block）独立运行，所以如果RDD包含元素类型为T，则 func 必须是 Iterator<T> => Iterator<U> 的映射函数。 |
| mapPartitionsWithIndex(func)                         | 类似于 mapPartitions，只是func 多了一个整型的分区索引值，因此如果RDD包含元素类型为T，则 func 必须是 Iterator<T> => Iterator<U> 的映射函数。 |
| sample(withReplacement, fraction, seed)              | 采样部分（比例取决于 fraction ）数据，同时可以指定是否使用回置采样（withReplacement），以及随机数种子(seed) |
| union(otherDataset)                                  | 返回源数据集和otherDataset数据集并集                         |
| intersection(otherDataset)                           | 返回源数据集和otherDataset数据集交集                         |
| distinct([numTasks]))                                | 返回元素去重后的新RDD                                        |
| groupByKey([numTasks])                               | 如源RDD包含 (K, V) 对，则该算子返回新RDD包含 (K, Iterable<V>) 对。注意：如果按key分组聚合（如sum或average），推荐 reduceByKey 或 aggregateByKey 获得更好性能。默认输出计算并行度等于源RDD分区数。也可以设置参数 numTasks 来指定 |
| reduceByKey(func, [numTasks])                        | 如源RDD包含 (K, V) 对，则该算子返回包含(K, V) 对的RDD，其中每个key对应的value是经过func聚合后结果，func本身是一个 (V, V) => V 映射函数。 |
| aggregateByKey(zeroValue)(seqOp, combOp, [numTasks]) | 如源RDD包含 (K, V) 对，则返回新RDD包含 (K, U) 对，其中每个key对应value都是由 combOp 函数 和 一个“0”值zeroValue 聚合得到。允许聚合后value类型和输入value类型不同，避免不必要的开销。 |
| sortByKey([ascending], [numTasks])                   | 如源RDD包含 (K, V) 对，其中K可排序，则返回新的RDD包含 (K, V) 对，并按照 K 排序（ascending 决定升降序） |
| join(otherDataset, [numTasks])                       | 如源RDD包含 (K, V) 且otherDataset RDD包含元素类型(K, W)，则返回的新RDD中包含内联 (K, (V, W)) 对。外关联(Outer joins)操作请参考 leftOuterJoin、rightOuterJoin 以及 fullOuterJoin 算子。 |
| cogroup(otherDataset, [numTasks])                    | 如源RDD包含 (K, V) 且otherDataset RDD包含元素类型(K, W)，则返回的新RDD中包含 (K, (Iterable<V>, Iterable<W>))。算子还有别名：groupWith |
| cartesian(otherDataset)                              | 如源RDD包含元素类型 T 且otherDataset RDD包含元素类型 U，则返回的新RDD包含前二者的笛卡尔积，其元素类型为 (T, U) 对。 |
| pipe(command, [envVars])                             | 以shell命令行管道处理RDD每个分区，如：Perl 或者 bash 脚本。RDD中每个元素都将依次写入进程标准输入（stdin），然后按行输出到标准输出（stdout），每一行输出字符串即成为一个新RDD元素。 |
| coalesce(numPartitions)                              | 将RDD分区数减少到numPartitions。当大数据集被过滤成小数据集后，减少分区数可提升效率。 |
| repartition(numPartitions)                           | 将RDD数据重新混洗（reshuffle）并随机分布到新分区中，使数据分布更均衡，新分区个数取决于numPartitions。总是需要通过网络混洗所有数据。 |
| repartitionAndSortWithinPartitions(partitioner)      | 根据partitioner（spark自带有HashPartitioner和RangePartitioner等）重新分区RDD，并在每个结果分区中按key排序。功能等于先 repartition 再分区内排序，但这个算子内部有优化，将排序过程下推到混洗同时进行，性能更好。 |

##### 动作算子

| Action算子                               | 作用                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| reduce(func)                             | 将RDD中元素按func进行聚合（func是一个 (T,T) => T 映射函数，其中T为源RDD元素类型，并且func需要满足 交换律 和 结合律 以便支持并行计算） |
| collect()                                | 将数据集中所有元素以数组形式返回驱动器（driver）程序。通常在RDD进行了filter或其他过滤操作后，将一个足够小数据子集返回到驱动器内存中。 |
| count()                                  | 返回数据集元素个数                                           |
| first()                                  | 返回数据集首个元素（类似于 take(1) ）                        |
| take(n)                                  | 返回数据集前 n 个元素                                        |
| takeSample(withReplacement,num, [seed])  | 返回数据集随机采样子集，最多包含 num 个元素，withReplacement 表示是否使用回置采样，seed为随机种子。 |
| takeOrdered(n, [ordering])               | 返回排序（可以通过 ordering 自定义排序）后前 n 个元素        |
| saveAsTextFile(path)                     | 将数据集元素保存到指定文件或多个文件，支持本地文件系统、HDFS 或其他Hadoop支持文件系统。保存过程中，Spark会调用每个元素toString方法，并将结果保存成文件一行 |
| saveAsSequenceFile(path)(Java and Scala) | 将数据集元素保存到指定目录下Hadoop Sequence文件中，支持本地文件系统、HDFS 或其他Hadoop支持文件系统。适用于实现了Writable接口键值对RDD。也适用于Scala中能够隐式转换为Writable类型（Spark实现了所有基本类型隐式转换，如：Int，Double，String 等） |
| saveAsObjectFile(path)(Java and Scala)   | 将RDD元素以Java序列化格式保存成文件，保存结果文件可以使用 SparkContext.objectFile 来读取。 |
| countByKey()                             | 只适用于包含键值对(K, V)的RDD，返回哈希表，包含 (K, Int) 对，表示每个key个数。 |
| foreach(func)                            | 在RDD的每个元素上运行 func 函数。通常被用于累加操作，如：更新一个累加器（Accumulator ） 或者 和外部存储系统互操作。 |

##### 混洗算子 Shuffle

混洗机制用于数据重新分布，其结果是所有数据将在各个分区间重新分组。一般情况下，混洗需要跨执行器（Executor）或跨机器复制数据，这也是混洗操作一般都比较复杂而且开销大的原因。因为混洗操作需要引入磁盘I/O，数据序列化以及网络I/O等操作。为了组织好混洗数据，Spark需要生成对应的任务集 – 一系列map任务用于组织数据，再用一系列reduce任务来聚合数据。

计算时用到其他分区数据，Spark需要读取所有分区，查找所有key对应values，然后跨区传输values，把key对应的所有values放同一分区，以便后续计算values的reduce结果，这个过程叫混洗。

虽然混洗好后，各个分区中的元素和分区自身的顺序都是确定的，但是分区中元素的顺序并非确定的。如果需要混洗后分区内的元素有序，可以参考使用以下混洗操作：

- mapPartitions 使用 .sorted 对每个分区排序
- repartitionAndSortWithinPartitions 重分区的同时，对分区进行排序，比自行组合repartition和sort更高效
- sortBy 创建一个全局有序的RDD

会导致混洗的算子有：

- 重分区（repartition）类算子，如： repartition 和 coalesce；
- ByKey 类算子(除了计数类的，如 countByKey) 如：groupByKey 和 reduceByKey；
- Join类算子，如：cogroup 和 join。

### 并行集合 Parallelized collections

可以被分布式计算的集合。调用 SparkContext 的 `parallelize` 方法生成。

```java
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)
```

#### 切片数

数据集切分的份数，每个切片运行一个任务。通过第二个参数指定 parallelize(data, 10) 。

### 外部数据集

#### textFile

通过textFile加载，支持 hdfs://，s3n:// 等，第二个参数指定切片数

```java
val distFile = sc.textFile("data.txt")
var distFile = sc.textFile("hdfs://localhost:9000/inp")
# 进行map-reduce  
distFile.map(s => s.length).reduce((a, b) => a + b)
```

#### sholeTextFiles

读取一个包含多个小文本文件的文件目录并且返回每一个(filename, content)对

```java
```

#### sequenceFiles

创建 KV对 数据集，分别对应的是 key 和 values 的类型

```java

```

## 目录结构

- python,R目录：源代码
- README.md：入门帮助
- bin：可执行命令
- examples：包含有java,python,r语言的入门Demo源码

### Shell

```bash
cd spark/bin

# python shell
./python-shell

# scala shell
./spark-shell
./spark-shell --master spark://IP:PORT

val lines = sc.textFile("/usr/local/spark/spark-2.2.3-bin-hadoop2.7/README.md")
lines.count()         //输出总count行数
lines.first()         //文件第一行

# 创建并行集合，转化了才可以分布式运行
val no = Array(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
val noData = sc.parallelize(no)

val newRDD = no.map(data => (data * 2))
val DFData = data.filter(line => line.contains("Elephant"))
// RDD 分区数
data.partitions.length
// 缓存数据到内存中，懒执行的
data.cache()
// 单词统计，前五位排名
val wc = hFile.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
wc.take(5)
wc.saveAsTextFile("hdfs://localhost:9000/out")

# ctrl+D退出shell
```

### 示例程序

```bash
# For Scala and Java, use run-example:
./bin/run-example SparkPi

# For Python examples, use spark-submit directly:
./bin/spark-submit examples/src/main/python/pi.py
```

## 打包执行

```bash
find .
.
./simple.sbt
./src
./src/main
./src/main/scala
./src/main/scala/SimpleApp.scala

# Package a jar containing your application
$ sbt package
...
[info] Packaging {..}/{..}/target/scala-2.10/simple-project_2.10-1.0.jar

# Use spark-submit to run your application
$ YOUR_SPARK_HOME/bin/spark-submit \
  --class "SimpleApp" \
  --master local[4] \
  target/scala-2.10/simple-project_2.10-1.0.jar
```

## Spark Streaming

### DStream

离散流（discretized stream）代表连续数据流，由一系列RDD组成。

### 窗口 Window

- 窗口长度：窗口的持续时间
- 滑动的时间间隔：窗口操作执行的时间间隔

```java
// Reduce 最近30秒的Windows数据, 每 10 秒进行一次
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
```

| Transformation                                               | 解释                                                |
| ------------------------------------------------------------ | --------------------------------------------------- |
| window(windowLength, slideInterval)                          | 最近windowLength的数据，每间隔slideInterval进行一次 |
| countByWindow(windowLength, slideInterval)                   |                                                     |
| reduceByWindow(func, windowLength, slideInterval)            |                                                     |
| reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks]) |                                                     |
| reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks]) |                                                     |
| countByValueAndWindow(windowLength, slideInterval, [numTasks]) |                                                     |

### 快速例子

```java
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._
// 2线程 1秒批间隔，local表示本机运行，还可以是Spark、Mesos、YARN集群的URL
val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
// 监听本机的文本流 localhost:9999，lines words都是DStream对象
val lines = ssc.socketTextStream("localhost", 9999)
val words = lines.flatMap(_.split(" "))
// Count each word in each batch
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)
wordCounts.print()
// 开始执行
ssc.start()
ssc.awaitTermination()
```

### checkpoint 恢复检查点

```java
def functionToCreateContext(): StreamingContext = {
    val ssc = new StreamingContext(...)   // new context
    ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
    ...
}
// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)
...
// Start the context
context.start()
context.awaitTermination()
```

## SparkSql

```java
import sqlContext.createSchemaRDD

val people: RDD[Person] = ... // An RDD of case class objects, from the previous example.

// The RDD is implicitly converted to a SchemaRDD by createSchemaRDD, allowing it to be stored using Parquet.
people.saveAsParquetFile("people.parquet")

// Read in the parquet file created above.  Parquet files are self-describing so the schema is preserved.
// The result of loading a Parquet file is also a SchemaRDD.
val parquetFile = sqlContext.parquetFile("people.parquet")

//Parquet files can also be registered as tables and then used in SQL statements.
parquetFile.registerTempTable("parquetFile")
val teenagers = sqlContext.sql("SELECT name FROM parquetFile WHERE age >= 13 AND age <= 19")
teenagers.map(t => "Name: " + t(0)).collect().foreach(println)
```

### 加载数据源

#### RDDs

##### 反射推断方式加载

```java
// sc is an existing SparkContext.
val sqlContext = new org.apache.spark.sql.SQLContext(sc)
// createSchemaRDD is used to implicitly convert an RDD to a SchemaRDD.
import sqlContext.createSchemaRDD
// 用 case class 定义 schema
// Scala 2.10 只支持22个fields，需要使用自定义类实现Product interface
case class Person(name: String, age: Int)
// Create an RDD of Person objects and register it as a table.
val people = sc.textFile("examples/src/main/resources/people.txt").map(_.split(",")).map(p => Person(p(0), p(1).trim.toInt))
people.registerTempTable("people")
// 查询
val teenagers = sqlContext.sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")
// SQL查询的结果是SchemaRDD，并支持所有正常的RDD操作，结果中的某列可以通过序数进行访问。
teenagers.map(t => "Name: " + t(0)).collect().foreach(println)
```

##### 编程指定模式

如果样本类不能提前确定，通过以下方式创建：

- 从原来的RDD创建一个行的RDD
- 创建由一个`StructType`表示的模式与第一步创建的RDD的行结构相匹配
- 在行RDD上通过`applySchema`方法应用模式

```java
// sc is an existing SparkContext.
val sqlContext = new org.apache.spark.sql.SQLContext(sc)
val people = sc.textFile("examples/src/main/resources/people.txt")
import org.apache.spark.sql._
// 通过定义的字符串，创建 schema
val schemaString = "name age"
val schema =
  StructType(
    schemaString.split(" ").map(fieldName => StructField(fieldName, StringType, true)))
// Convert records of the RDD (people) to Rows.
val rowRDD = people.map(_.split(",")).map(p => Row(p(0), p(1).trim))
// Apply the schema to the RDD.
val peopleSchemaRDD = sqlContext.applySchema(rowRDD, schema)
// Register the SchemaRDD as a table.
peopleSchemaRDD.registerTempTable("people")
val results = sqlContext.sql("SELECT name FROM people")
// 结果中的某列可以通过序数进行访问
results.map(t => "Name: " + t(0)).collect().foreach(println)
```

#### Parquet

```java
import sqlContext.createSchemaRDD
val people: RDD[Person] = ... // An RDD of case class objects, from the previous example.
// createSchemaRDD 可以隐含转换RDD到SchemaRDD，并保存为Parquet
people.saveAsParquetFile("people.parquet")
// 加载Parquet 为 SchemaRDD，Parquet 带有 schema 信息
val parquetFile = sqlContext.parquetFile("people.parquet")
//Parquet文件也可以注册为Table供SQL查询
parquetFile.registerTempTable("parquetFile")
val teenagers = sqlContext.sql("SELECT name FROM parquetFile WHERE age >= 13 AND age <= 19")
// 打印
teenagers.map(t => "Name: " + t(0)).collect().foreach(println)
```

##### 配置项

可以在SQLContext上使用setConf方法设置，或执行SQL`SET key=value`命令来配置Parquet。

| 属性                                   | 默认值 | 用途                                                         |
| :------------------------------------- | :----- | :----------------------------------------------------------- |
| spark.sql.parquet.binaryAsString       | false  | 其它系统，特别是Impala或其它版本Spark SQL，写Parquet时无法区分二进制数据和字符串。该标记告诉Spark SQL将二进制数据解释为字符串来提高兼容性。 |
| spark.sql.parquet.cacheMetadata        | true   | 打开parquet元数据缓存，可以提高静态数据查询速度              |
| spark.sql.parquet.compression.codec    | gzip   | 设置写parquet文件压缩算法，如：uncompressed, snappy, gzip, lzo |
| spark.sql.parquet.filterPushdown       | false  | 打开Parquet过滤下推优化（filter很可能过滤了大批数据，下推可以把大量数据在读取时过滤掉）。因为已知的Parquet错误，这个特征默认关闭。如果表不包含任何空字符串或者二进制列，打开这个特征仍是安全的 |
| spark.sql.hive.convertMetastoreParquet | true   | 当设置为false时，Spark SQL将使用Hive SerDe代替内置的支持     |

#### Json

```java
// 加载json文件
val sqlContext = new org.apache.spark.sql.SQLContext(sc)
// 路径支持单一文件或文件夹
val path = "examples/src/main/resources/people.json"
// 创建 SchemaRDD
val people = sqlContext.jsonFile(path)
// printSchema() 打印推断的schema
people.printSchema()
// root
//  |-- age: integer (nullable = true)
//  |-- name: string (nullable = true)
// 注册为表
people.registerTempTable("people")
val teenagers = sqlContext.sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")

// 加载json对象
// RDD[String] 一个字符串保存一个json对象
val anotherPeopleRDD = sc.parallelize(
  """{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}""" :: Nil)
val anotherPeople = sqlContext.jsonRDD(anotherPeopleRDD)
```

#### Hive

注意 Hive本身有大量依赖，默认不包含在Spark集合中。可以通过`-Phive`和`-Phive-thriftserver`参数构建Spark，使其支持Hive。这些重新构建的jar包需要在所有worker节点中。

开发者需要提供HiveContext。HiveContext从SQLContext继承而来，它增加了在MetaStore中发现表以及利用HiveSql写查询的功能。

没有Hive部署的用户也可以创建HiveContext。当没有通过`hive-site.xml`配置，上下文将会在当前目录自动地创建`metastore_db`和`warehouse`

```java
// sc 是 SparkContext.
val sqlContext = new org.apache.spark.sql.hive.HiveContext(sc)
sqlContext.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)")
sqlContext.sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")
// 使用 HiveQL 进行查询
sqlContext.sql("FROM src SELECT key, value").collect().foreach(println)
```

## Spark on YARN

```bash
./spark-submit --class path.to.your.Class --master yarn-cluster [options] <app jar> [app options]

# yarn-cluster
./spark-submit --class org.apache.spark.examples.SparkPi \
    --master yarn-cluster \
    --num-executors 3 \
    --driver-memory 4g \
    --executor-memory 2g \
    --executor-cores 1 \
    --queue thequeue \
    --jars my-other-jar.jar,my-other-other-jar.jar
    lib/spark-examples*.jar \
    10 app_arg2

# --files和--archives 支持文件改名#，如 --files local.txt#remote.txt 本地文件local.txt拷贝到hdfs会叫remote.txt


# yarn-client
./spark-shell --master yarn-client
```

