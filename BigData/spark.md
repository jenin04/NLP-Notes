# RDD

## 概述

在较高级别上，每个Spark应用程序都包含一个驱动程序（driver program），该程序运行用户的主要功能并在集群上执行各种并行操作。 Spark提供的主要抽象是弹性分布式数据集（Resilient distributed dataset, RDD），它是跨集群节点划分的元素的集合，可以并行操作。通过从Hadoop文件系统（或任何其他Hadoop支持的文件系统）中的文件或驱动程序中现有的Scala集合开始并进行转换来创建RDD。用户还可以要求Spark将RDD持久存储在内存中，从而使其可以在并行操作中高效地重用。最后，RDD会自动从节点故障中恢复。 

Spark中的第二个抽象是可以在并行操作中使用的共享变量。默认情况下，当Spark作为一组任务在不同节点上并行运行一个函数时，它会将函数中使用的每个变量的副本传送给每个任务。有时，需要在任务之间或任务与驱动程序之间共享变量。 Spark支持两种类型的共享变量：广播变量（broadcast variables，可用于在所有节点上的内存中缓存值）和累加器（accumulators，累加器）是仅“加”到变量的变量，例如计数器和总和。

## Linking with Spark

```scala
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
```

## Initializing Spark

Spark程序必须做的第一件事是创建一个SparkContext对象，该对象告诉Spark如何访问集群。要创建一个SparkContext，您首先需要构建一个SparkConf对象，该对象包含有关您的应用程序的信息。

每个JVM只能激活一个SparkContext。在创建新的SparkContext之前，您必须stop（）活动的SparkContext。

```scala
val conf = new SparkConf().setAppName(appName).setMaster(master)
new SparkContext(conf)
```

相关参数：

- appName：是您的应用程序在集群UI上显示的名称。 
- master：是一个Spark、Mesos、YARN集群URL或特殊的“本地”字符串，以本地模式运行。实际上，当在集群上运行时，您将不希望对程序中的母版进行硬编码，而希望通过spark-submit启动应用程序并在其中接收它。但是，对于本地测试和单元测试，您可以传递“ local”以在内部运行Spark。

## Resilient Distributed Datasets（RDDs）

Spark围绕弹性分布式数据集（RDD）的概念展开，RDD是可并行操作的元素的容错集合。

创建RDD的方法有两种：（1）并行化驱动程序中的现有集合；（2）引用外部存储系统（例如共享文件系统，HDFS，HBase或提供Hadoop InputFormat的任何数据源）中的数据集。

### 创建方法一：Parallelized Collections

通过在驱动程序中的现有集合（Scala Seq）上调用SparkContext.parallelize方法来创建并行集合。复制集合的元素以形成可并行操作的分布式数据集。例如，以下是创建包含数字1到5的并行化集合的方法：

```scala
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)
```

相关参数：

- 并行集合的一个重要参数是将数据集切分的分区数。 Spark将为集群的每个分区运行一个任务。通常，群集中的每个CPU都需要2-4个分区。通常，Spark会尝试根据您的集群自动设置分区数。但是，您也可以通过将其作为第二个参数传递来进行手动设置（例如sc.parallelize（data，10））。注意：代码中的某些位置使用术语片（分区的同义词）来保持向后兼容性。

### 创建方法二：External Datasets

Spark可以从Hadoop支持的任何存储源（包括本地文件系统，HDFS，Cassandra，HBase，Amazon S3等）创建分布式数据集。Spark支持文本文件，SequenceFiles和任何其他Hadoop InputFormat。

```scala
scala> val distFile = sc.textFile("data.txt")
distFile: org.apache.spark.rdd.RDD[String] = data.txt MapPartitionsRDD[10] at textFile at <console>:26
```

创建distFile后，即可通过数据集操作（dataset operations）对其进行操作：

- 如果在本地文件系统上使用路径，则还必须在工作节点上的相同路径上访问该文件。 将文件复制给所有workers，或使用网络安装的共享文件系统。
- Spark的所有基于文件的输入方法（包括textFile()）都支持在目录，压缩文件和通配符的运行。 例如，您可以使用textFile（“/my/directory”），textFile（“/my/directory/\*.txt”）和textFile（“/my/directory/\*.gz”）。
- textFile()方法还采用一个可选的第二个参数来控制文件的分区数。 默认情况下，Spark为文件的每个块创建一个分区（HDFS中的块默认为128MB），但是您也可以通过传递更大的值来请求更大数量的分区。 请注意，分区不能少于块。

除文本文件外，Spark的Scala API还支持其他几种数据格式：

- SparkContext.wholeTextFiles：允许您读取包含多个小文本文件的目录，并将每个小文本文件作为（文件名，内容）对返回。这与textFile()相反，后者将在每个文件的每一行返回一条记录。分区由数据局部性决定，在某些情况下，数据局部性可能导致分区太少。在这种情况下，wholeTextFiles()提供了一个可选的第二个参数来控制最小数量的分区。

- 对于SequenceFiles，请使用 SparkContext.sequenceFile[K，V]\() 方法，其中K和V是文件中键的类型和值。这些应该是Hadoop可写接口的子类，例如IntWritable和Text。此外，Spark允许您为一些常见的可写对象指定本机类型。例如，sequenceFile[Int，String]将自动读取IntWritables和Texts。

- 对于其他Hadoop InputFormat，可以使用SparkContext.hadoopRDD方法，该方法采用任意JobConf和输入格式类，键类和值类。使用与输入源的Hadoop作业相同的方式设置这些内容。您还可以基于“新” MapReduce API（org.apache.hadoop.mapreduce）将SparkContext.newAPIHadoopRDD用于InputFormats。

- RDD.saveAsObjectFile和SparkContext.objectFile支持以包含序列化Java对象的简单格式保存RDD。尽管这不像Avro这样的专用格式有效，但它提供了一种保存任何RDD的简便方法。

### RDD操作

RDD支持两种类型的操作：转换（*transformations*，从现有操作创建新数据集）和动作（*actions*，在操作数据集上执行计算后将值返回到驱动程序）。

- 例如，map是一种转换，它将每个数据集元素通过一个函数传递，并返回代表结果的新RDD。另一方面，reduce是使用某些函数聚合RDD的所有元素并将最终结果返回给驱动程序的操作（尽管也有并行的reduceByKey返回分布式数据集）。

Spark中的所有转换都是惰性的，因为它们不会立即计算出结果。取而代之的是，他们只记得应用于某些基本数据集（例如文件）的转换。仅当操作要求将结果返回给驱动程序时才计算转换。这种设计使Spark可以更高效地运行。

- 例如，我们可以认识到通过map创建的数据集将用于reduce中，并且仅将reduce的结果返回给驱动程序，而不是将较大的maped数据集返回给驱动程序。

默认情况下，每次在其上执行操作时，可能都会重新计算每个转换后的RDD。但是，您也可以使用*persist*（或缓存）方法将RDD保留在内存中，在这种情况下，Spark会将元素保留在群集中，以便下次查询时可以更快地进行访问。还支持将RDD持久存储在磁盘上，或在多个节点之间复制。

#### Passing Functions to Spark

Spark的API在很大程度上依赖于在驱动程序中传递函数以在群集上运行。 有两种推荐做法：

- 匿名函数语法：可用于简短的代码段。

- 全局单例对象中的静态方法。

  例如，您可以定义对象MyFunctions，然后按如下所示传递MyFunctions.func1：

  ```scala
  object MyFunctions {
    def func1(s: String): String = { ... }
  }
  
  myRdd.map(MyFunctions.func1)
  ```

  注意，虽然也可以在类实例中传递对方法的引用（与单例对象相对），但这需要将包含该类的对象与方法一起发送。

  ```scala
  class MyClass {
    def func1(s: String): String = { ... }
    def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
  }
  ```

#### Understanding closures

```scala
var counter = 0
var rdd = sc.parallelize(data)

// Wrong: Don't do this!!
rdd.foreach(x => counter += x)

println("Counter value: " + counter)
```

上面的代码的行为是未定义的，可能无法按预期工作。

在集群模式下，为了执行作业，Spark将RDD操作的处理分解为任务，每个任务都由 executor 执行。在执行之前，Spark会计算任务的结束时间。闭包（closures）是 executor 在RDD上执行其计算时必须可见的那些变量和方法（在本例中为foreach()）。此闭包被序列化并发送给每个executor。发送给每个 executor 的闭包中的变量现在已被复制，因此，在foreach函数中引用counter时，它不再是driver node上的计数器。driver node的内存中仍然存在一个counter，但是executor将不再看到该counter！executor仅从序列化闭包中看到副本。因此，因为对counter的所有操作都引用了序列化闭包内的值，所以counter的最终值仍将为零。

在本地模式下，某些情况的foreach函数实际上将在与驱动程序相同的JVM中执行，并且将引用相同的原始counter，并且可能会对其进行实际更新。

为确保在此类情况下行为明确，应使用**Accumulators**。 Spark中的**Accumulators**专门用于提供一种机制，用于在集群中的各个工作节点之间拆分执行时安全地更新变量。

#### Printing elements of an RDD

在单台机器上，使用以下两种方法可打印出RDD的元素

```scala
rdd.foreach(println)
```

或

```scala
rdd.map(println)
```

在集群模式下，executor正在调用stdout的输出现在写入executor的stdout，而不是driver上的那个，因此driver上的stdout不会显示这些信息！要在驱动程序上打印所有元素，可以使用 `collect()` 方法首先将RDD带到驱动程序节点：

```scala
rdd.collect().foreach(println)
```

但是，这可能会导致驱动程序用尽内存，因为collect（）将整个RDD提取到一台计算机上。如果只需要打印RDD的一些元素，则更安全的方法是使用take（）：

```scala
rdd.take(100).foreach(println)
```

####  Key-Value Pairs（Pair RDD）

```scala
val lines = sc.textFile("data.txt")
val pairs = lines.map(s => (s, 1))
val counts = pairs.reduceByKey((a, b) => a + b)
```

#### Transformations

| Transformation                                               | Meaning                                                      |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| **map**(*func*)                                              | Return a new distributed dataset formed by passing each element of the source through a function *func*. |
| **filter**(*func*)                                           | Return a new dataset formed by selecting those elements of the source on which *func* returns true. |
| **flatMap**(*func*)                                          | Similar to map, but each input item can be mapped to 0 or more output items (so *func* should return a Seq rather than a single item). |
| **mapPartitions**(*func*)                                    | Similar to map, but runs separately on each partition (block) of the RDD, so *func* must be of type Iterator<T> => Iterator<U> when running on an RDD of type T. |
| **mapPartitionsWithIndex**(*func*)                           | Similar to mapPartitions, but also provides *func* with an integer value representing the index of the partition, so *func* must be of type (Int, Iterator<T>) => Iterator<U> when running on an RDD of type T. |
| **sample**(*withReplacement*, *fraction*, *seed*)            | Sample a fraction *fraction* of the data, with or without replacement, using a given random number generator seed. |
| **union**(*otherDataset*)                                    | Return a new dataset that contains the union of the elements in the source dataset and the argument. |
| **intersection**(*otherDataset*)                             | Return a new RDD that contains the intersection of elements in the source dataset and the argument. |
| **distinct**([*numPartitions*]))                             | Return a new dataset that contains the distinct elements of the source dataset. |
| **groupByKey**([*numPartitions*])                            | When called on a dataset of (K, V) pairs, returns a dataset of (K, Iterable<V>) pairs. **Note:** If you are grouping in order to perform an aggregation (such as a sum or average) over each key, using `reduceByKey` or `aggregateByKey` will yield much better performance. **Note:** By default, the level of parallelism in the output depends on the number of partitions of the parent RDD. You can pass an optional `numPartitions` argument to set a different number of tasks. |
| **reduceByKey**(*func*, [*numPartitions*])                   | When called on a dataset of (K, V) pairs, returns a dataset of (K, V) pairs where the values for each key are aggregated using the given reduce function *func*, which must be of type (V,V) => V. Like in `groupByKey`, the number of reduce tasks is configurable through an optional second argument. |
| **aggregateByKey**(*zeroValue*)(*seqOp*, *combOp*, [*numPartitions*]) | When called on a dataset of (K, V) pairs, returns a dataset of (K, U) pairs where the values for each key are aggregated using the given combine functions and a neutral "zero" value. Allows an aggregated value type that is different than the input value type, while avoiding unnecessary allocations. Like in `groupByKey`, the number of reduce tasks is configurable through an optional second argument. |
| **sortByKey**([*ascending*], [*numPartitions*])              | When called on a dataset of (K, V) pairs where K implements Ordered, returns a dataset of (K, V) pairs sorted by keys in ascending or descending order, as specified in the boolean `ascending` argument. |
| **join**(*otherDataset*, [*numPartitions*])                  | When called on datasets of type (K, V) and (K, W), returns a dataset of (K, (V, W)) pairs with all pairs of elements for each key. Outer joins are supported through `leftOuterJoin`, `rightOuterJoin`, and `fullOuterJoin`. |
| **cogroup**(*otherDataset*, [*numPartitions*])               | When called on datasets of type (K, V) and (K, W), returns a dataset of (K, (Iterable<V>, Iterable<W>)) tuples. This operation is also called `groupWith`. |
| **cartesian**(*otherDataset*)                                | When called on datasets of types T and U, returns a dataset of (T, U) pairs (all pairs of elements). |
| **pipe**(*command*, *[envVars]*)                             | Pipe each partition of the RDD through a shell command, e.g. a Perl or bash script. RDD elements are written to the process's stdin and lines output to its stdout are returned as an RDD of strings. |
| **coalesce**(*numPartitions*)                                | Decrease the number of partitions in the RDD to numPartitions. Useful for running operations more efficiently after filtering down a large dataset. |
| **repartition**(*numPartitions*)                             | Reshuffle the data in the RDD randomly to create either more or fewer partitions and balance it across them. This always shuffles all data over the network. |
| **repartitionAndSortWithinPartitions**(*partitioner*)        | Repartition the RDD according to the given partitioner and, within each resulting partition, sort records by their keys. This is more efficient than calling `repartition` and then sorting within each partition because it can push the sorting down into the shuffle machinery. |

#### Actions

| Action                                             | Meaning                                                      |
| :------------------------------------------------- | :----------------------------------------------------------- |
| **reduce**(*func*)                                 | Aggregate the elements of the dataset using a function *func* (which takes two arguments and returns one). The function should be commutative and associative so that it can be computed correctly in parallel. |
| **collect**()                                      | Return all the elements of the dataset as an array at the driver program. This is usually useful after a filter or other operation that returns a sufficiently small subset of the data. |
| **count**()                                        | Return the number of elements in the dataset.                |
| **first**()                                        | Return the first element of the dataset (similar to take(1)). |
| **take**(*n*)                                      | Return an array with the first *n* elements of the dataset.  |
| **takeSample**(*withReplacement*, *num*, [*seed*]) | Return an array with a random sample of *num* elements of the dataset, with or without replacement, optionally pre-specifying a random number generator seed. |
| **takeOrdered**(*n*, *[ordering]*)                 | Return the first *n* elements of the RDD using either their natural order or a custom comparator. |
| **saveAsTextFile**(*path*)                         | Write the elements of the dataset as a text file (or set of text files) in a given directory in the local filesystem, HDFS or any other Hadoop-supported file system. Spark will call toString on each element to convert it to a line of text in the file. |
| **saveAsSequenceFile**(*path*) (Java and Scala)    | Write the elements of the dataset as a Hadoop SequenceFile in a given path in the local filesystem, HDFS or any other Hadoop-supported file system. This is available on RDDs of key-value pairs that implement Hadoop's Writable interface. In Scala, it is also available on types that are implicitly convertible to Writable (Spark includes conversions for basic types like Int, Double, String, etc). |
| **saveAsObjectFile**(*path*) (Java and Scala)      | Write the elements of the dataset in a simple format using Java serialization, which can then be loaded using `SparkContext.objectFile()`. |
| **countByKey**()                                   | Only available on RDDs of type (K, V). Returns a hashmap of (K, Int) pairs with the count of each key. |
| **foreach**(*func*)                                | Run a function *func* on each element of the dataset. This is usually done for side effects such as updating an [Accumulator](https://spark.apache.org/docs/2.3.1/rdd-programming-guide.html#accumulators) or interacting with external storage systems. **Note**: modifying variables other than Accumulators outside of the `foreach()` may result in undefined behavior. See [Understanding closures ](https://spark.apache.org/docs/2.3.1/rdd-programming-guide.html#understanding-closures-a-nameclosureslinka)for more details. |

#### Shuffle operations

在Spark中，数据通常不会跨分区分布在特定操作的必要位置。在计算期间，单个任务将在单个分区上进行操作。因此，为了组织要执行的单个`reduceByKe`reduce任务的所有数据，Spark需要执行所有操作。它必须从所有分区读取以找到所有键的所有值，然后将各个分区中的值汇总在一起以计算每个键的最终结果，这称为**shuffle**。

尽管新改组后的数据的每个分区中的元素集都是确定性的，分区本身的顺序也是如此，但这些元素的顺序却不是。如果人们希望在洗牌后能有序地整理数据，则可以使用：

- `mapPartitions`对每个分区进行排序，例如.sorted。
- `repartitionAndSortWithinPartitions`可以有效地对分区进行排序，同时进行重新分区 
- `sortBy`生成全局有序的RDD

##### Performance Impact

shuffle是一项代价很大的操作，因为它涉及磁盘I/O，数据序列化和网络I/O。为了组织随机数据，Spark生成任务集-映射任务以组织数据，以及一组reduce任务来聚合数据。此术语来自MapReduce，与Spark的地map和reduce操作没有直接关系。

内部机制：类似map的任务结果会保留在内存中，直到无法容纳为止。然后，根据目标分区对它们进行排序并写入单个文件；类似reduce的任务会读取相关的已排序块。某些shuffle操作会占用大量的堆内存，因为它们在转移它们之前或之后采用内存中的数据结构来组织记录。具体来说，reduceByKey和aggregateByKey在map阶段时创建这些结构，而*ByKey操作在reduce阶段生成这些结构。当数据填满内存时，Spark会将这些表溢出到磁盘上，从而产生磁盘I/O的额外开销并增加垃圾回收。

shuffle还会在磁盘上生成大量中间文件。从Spark 1.3开始，将保留这些文件，直到不再使用相应的RDD并进行垃圾回收为止。这样做是为了在重新计算沿袭时无需重新创建shuffle文件。如果应用程序保留了对这些RDD的引用，或者如果GC不经常启动，则可能仅在很长一段时间后才发生垃圾回收。这意味着长时间运行的Spark作业可能会占用大量磁盘空间。在配置Spark上下文时，临时存储目录由spark.local.dir配置参数指定。

### RDD Persistence

Spark中最重要的功能之一是跨操作在内存中持久化（或缓存）数据集。当您保留RDD时，每个节点都会将其计算的任何分区存储在内存中，并在该数据集（或从该数据集派生的数据集）上的其他操作中重用它们。这样可以使以后的操作更快（通常快10倍以上）。缓存是迭代算法和快速交互使用的关键工具。 您可以使用其上的`persist()`或`cache()`方法将RDD标记为持久。第一次在操作中对其进行计算时，它将被保存在节点上的内存中。 Spark的缓存是容错的-如果RDD的任何分区丢失，它将使用最初创建它的转换自动重新计算。

此外，每个持久化的RDD可以使用不同的存储级别进行存储

- `persist()`方法：允许您将数据集持久化在磁盘上，持久化在内存中，但作为序列化的Java对象（以节省空间）在节点之间复制，通过将StorageLevel对象（Scala，Java，Python）传递给`persist()`来设置这些级别。 
- `cache()`方法：使用默认存储级别StorageLevel.MEMORY_ONLY（将反序列化的对象存储在内存中）的简写。完整的存储级别集是：

| Storage Level                          | Meaning                                                      |
| :------------------------------------- | :----------------------------------------------------------- |
| MEMORY_ONLY                            | Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, some partitions will not be cached and will be recomputed on the fly each time they're needed. This is the default level. |
| MEMORY_AND_DISK                        | Store RDD as deserialized Java objects in the JVM. If the RDD does not fit in memory, store the partitions that don't fit on disk, and read them from there when they're needed. |
| MEMORY_ONLY_SER (Java and Scala)       | Store RDD as *serialized* Java objects (one byte array per partition). This is generally more space-efficient than deserialized objects, especially when using a [fast serializer](https://spark.apache.org/docs/2.3.1/tuning.html), but more CPU-intensive to read. |
| MEMORY_AND_DISK_SER (Java and Scala)   | Similar to MEMORY_ONLY_SER, but spill partitions that don't fit in memory to disk instead of recomputing them on the fly each time they're needed. |
| DISK_ONLY                              | Store the RDD partitions only on disk.                       |
| MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc. | Same as the levels above, but replicate each partition on two cluster nodes. |
| OFF_HEAP (experimental)                | Similar to MEMORY_ONLY_SER, but store the data in [off-heap memory](https://spark.apache.org/docs/2.3.1/configuration.html#memory-management). This requires off-heap memory to be enabled. |

#### Which Storage Level to Choose?

Spark的存储级别旨在在内存使用量和CPU效率之间提供不同的权衡。我们建议通过以下过程选择一个：

-  如果您的RDD适合默认存储级别（MEMORY_ONLY），请保持这种状态。这是CPU效率最高的选项，允许RDD上的操作尽可能快地运行。 
- 如果不是，请尝试使用MEMORY_ONLY_SER并选择一个快速的序列化库，以使对象的空间效率更高，但访问速度仍然相当快。 （Java和Scala）
-  除非用于计算数据的函数很代价很大，不要轻易将数据存到磁盘中，否则它们会过滤掉大量数据。此外，重新计算分区可能与从磁盘读取分区一样快。 
- 如果要快速恢复故障（例如，如果使用Spark来处理来自Web应用程序的请求），请使用复制存储级别（replicated storage levels）。所有存储级别都能通过重新计算丢失的数据来提供完全的容错能力，但是复制存储级别（replicated storage levels）使您可以继续在RDD上运行任务，而不必等待重新计算丢失的分区。

#### Removing Data

Spark自动监视每个节点上的缓存使用情况，并以最近最少使用（LRU）的方式丢弃旧的数据分区。如果要手动删除RDD而不是等待它脱离缓存，请使用`RDD.unpersist()`方法。

## Shared Variables（共享变量）

### Broadcast Variables（广播变量）

广播变量使程序员可以在每台计算机上保留一个只读变量，而不用将其副本与任务一起发送。例如，可以使用它们以有效的方式为每个节点提供大型输入数据集的副本。 Spark还尝试使用有效的广播算法分配广播变量，以降低通信成本。 

spark动作是通过一组stage执行的，这些stage由分布式“随机”操作分开。 Spark自动广播每个阶段任务所需的通用数据。在运行每个任务之前，以这种方式广播的数据以序列化形式缓存并反序列化。这意味着仅当跨多个阶段的任务需要相同数据或以反序列化形式缓存数据非常重要时，显式创建广播变量才有用。 

广播变量是通过调用`SparkContext.broadcast(v)`从变量v创建的。广播变量是v的包装，可以通过调用`value`方法来访问其值。

```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```

### Accumulators(累加器)

累加器是仅通过关联和交换操作“添加”的变量，因此可以有效地并行支持。它们可用于实现计数器（如在MapReduce中）或总和。 Spark本机支持数字类型的累加器，程序员可以添加对新类型的支持。

可以通过调用`SparkContext.longAccumulator()`或`SparkContext.doubleAccumulator()`分别累积Long或Double类型的值来创建数字累加器。然后，可以使用add方法将在集群上运行的任务添加到集群中。但是，他们无法读取其值。只有driver可以使用value方法读取累加器的值。

```scala
scala> val accum = sc.longAccumulator("My Accumulator")
accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Long = 10
```

AccumulatorV2抽象类具有几种必须重写的方法：`reset`用于将累加器重置为零；`add`用于将另一个值添加到累加器；`merge`以将另一个相同类型的累加器合并到该方法中。 API文档中包含其他必须重写的方法。

```scala
class VectorAccumulatorV2 extends AccumulatorV2[MyVector, MyVector] {

  private val myVector: MyVector = MyVector.createZeroVector

  def reset(): Unit = {
    myVector.reset()
  }

  def add(v: MyVector): Unit = {
    myVector.add(v)
  }
  ...
}

// Then, create an Accumulator of this type:
val myVectorAcc = new VectorAccumulatorV2
// Then, register it into spark context:
sc.register(myVectorAcc, "MyVectorAcc1")
```

对于仅在操作内部执行的累加器更新，Spark保证每个任务对累加器的更新将仅应用一次，即重新启动的任务将不会更新该值。在转换中，用户应注意，如果重新执行任务或作业阶段，则可能不止一次应用每个任务的更新。

累加器不会更改Spark的惰性执行机制。如果在RDD上的操作中对其进行更新，则仅当将RDD计算为action的一部分时才更新其值。因此，在像`map()`这样的惰性转换中进行累加器更新时，不能保证会执行更新。

```scala
val accum = sc.longAccumulator
data.map { x => accum.add(x); x }
// Here, accum is still 0 because no actions have caused the map operation to be computed.
```

# Spark SQL, DataFrames and Datasets Guide

## 概述

dataset是数据的分布式集合。dataset是Spark 1.6中添加的新接口，它具有RDD的优点（强类型输入，使用强大的Lambda函数的能力）以及Spark SQL优化的执行引擎的优点。可以从JVM对象构造数据集，然后使用功能转换（map，flatMap，filter等）进行操作。 Dataset API在Scala和Java中可用。 Python不支持Dataset API。但是由于Python的动态特性，Dataset API的许多优点已经可用（即，您可以自然地通过名称row.columnName访问行的字段）。 R的情况类似。 

DataFrame是组织为命名列的数据集。从概念上讲，它等效于关系数据库中的表或R/Python中的数据框，但是在后台进行了更丰富的优化。可以从多种来源构造DataFrame，例如：结构化数据文件，Hive中的表，外部数据库或现有RDD。 DataFrame API在Scala，Java，Python和R中可用。在Scala和Java中，DataFrame由行的Dataset表示。在Scala API中，DataFrame只是`Dataset[Row]`的类型别名。而在Java API中，用户需要使用`Dataset <Row>`表示一个DataFrame。

## Getting Started

### Starting Point: SparkSession

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("Spark SQL basic example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()

// For implicit conversions like converting RDDs to DataFrames
import spark.implicits._
```

### 创建DataFrames

使用SparkSession，应用程序可以从现有RDD，Hive表或Spark数据源创建DataFrame。

```scala
val df = spark.read.json("examples/src/main/resources/people.json")
```

DataFrame为Scala、Java、Python和R中的结构化数据操作提供了一种特定于域的语言。 如上所述，在Spark 2.0中，DataFrame只是Scala和Java API中的行数据集。与强类型（strongly typed）的Scala/Java数据集附带的“类型转换”相反，这些操作也称为“非类型转换”（untyped transformations）。 

```scala
// This import is needed to use the $-notation
// spark不是某个包下面的东西，而是我们SparkSession.builder()对应的变量值
import spark.implicits._
// Print the schema in a tree format
df.printSchema()
// root
// |-- age: long (nullable = true)
// |-- name: string (nullable = true)

// Select only the "name" column
df.select("name").show()
// +-------+
// |   name|
// +-------+
// |Michael|
// |   Andy|
// | Justin|
// +-------+

// Select everybody, but increment the age by 1
df.select($"name", $"age" + 1).show()
// +-------+---------+
// |   name|(age + 1)|
// +-------+---------+
// |Michael|     null|
// |   Andy|       31|
// | Justin|       20|
// +-------+---------+

// Select people older than 21
df.filter($"age" > 21).show()
// +---+----+
// |age|name|
// +---+----+
// | 30|Andy|
// +---+----+

// Count people by age
df.groupBy("age").count().show()
// +----+-----+
// | age|count|
// +----+-----+
// |  19|    1|
// |null|    1|
// |  30|    1|
// +----+-----+
```

### 运行SQL查询编程

```scala
// Register the DataFrame as a SQL temporary view，必须注册为临时表才能供sql查询使用
df.createOrReplaceTempView("people")

val sqlDF = spark.sql("SELECT * FROM people")
sqlDF.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

### Global Temporary View

Spark SQL中的临时视图是会话作用域（session-scoped，即不同会话之间不会共享视图）的，如果创建它的会话终止，它将消失。如果要在所有会话之间共享一个临时视图并保持活动状态，直到Spark应用程序终止，则可以创建全局临时视图。全局临时视图与系统保留的数据库global_temp相关联，我们必须使用限定名称来引用它，例如 `SELECT * FROM global_temp.view1`。

```SCALA
// Register the DataFrame as a global temporary view
df.createGlobalTempView("people")

// Global temporary view is tied to a system preserved database `global_temp`
spark.sql("SELECT * FROM global_temp.people").show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// Global temporary view is cross-session
spark.newSession().sql("SELECT * FROM global_temp.people").show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

### 创建Datasets

Datasets与RDD相似，但是它们不是使用Java序列化或Kryo，而是使用专用的Encoder对对象进行序列化以进行网络处理或传输。虽然编码器和标准序列化都负责将对象转换为字节，但是编码器是动态生成的代码，并使用一种格式，该格式允许Spark执行许多操作（例如filtering、sorting和hashing），而无需将字节反序列化为对象。

```scala
case class Person(name: String, age: Long)

// Encoders are created for case classes
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
// +----+---+
// |name|age|
// +----+---+
// |Andy| 32|
// +----+---+

// Encoders for most common types are automatically provided by importing spark.implicits._
val primitiveDS = Seq(1, 2, 3).toDS()
primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)

// DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
val path = "examples/src/main/resources/people.json"
val peopleDS = spark.read.json(path).as[Person]
peopleDS.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

### Interoperating with RDDs

Spark SQL支持两种将现有RDD转换为数据集的方法。

- 第一种方法使用反射来推断包含特定对象类型的RDD的架构。这种基于反射的方法可以使代码更简洁，当您在编写Spark应用程序时已经了解架构时，可以很好地工作。
-  创建数据集的第二种方法是通过编程界面，该界面允许您构造模式，然后将其应用于现有的RDD。尽管此方法较为冗长，但可以在运行时才知道列及其类型的情况下构造数据集。

#### Inferring the Schema Using Reflection

```tsx
// sc 是已有的 SparkContext 对象
val sqlContext = new org.apache.spark.sql.SQLContext(sc)
// 为了支持RDD到DataFrame的隐式转换
import sqlContext.implicits._

// 定义一个case class.
// 注意：Scala 2.10的case class最多支持22个字段，要绕过这一限制，
// 你可以使用自定义class，并实现Product接口。当然，你也可以改用编程方式定义schema
case class Person(name: String, age: Int)

// 创建一个包含Person对象的RDD，并将其注册成table
val people = sc.textFile("examples/src/main/resources/people.txt").map(_.split(",")).map(p => Person(p(0), p(1).trim.toInt)).toDF()
people.registerTempTable("people")

// sqlContext.sql方法可以直接执行SQL语句
val teenagers = sqlContext.sql("SELECT name, age FROM people WHERE age >= 13 AND age <= 19")

// SQL查询的返回结果是一个DataFrame，且能够支持所有常见的RDD算子
// 查询结果中每行的字段可以按字段索引访问:
teenagers.map(t => "Name: " + t(0)).collect().foreach(println)

// 或者按字段名访问:
teenagers.map(t => "Name: " + t.getAs[String]("name")).collect().foreach(println)

// row.getValuesMap[T] 会一次性返回多列，并以Map[String, T]为返回结果类型
teenagers.map(_.getValuesMap[Any](List("name", "age"))).collect().foreach(println)
// 返回结果: Map("name" -> "Justin", "age" -> 19)
```

#### Programmatically Specifying the Schema

如果无法提前定义case class（例如，记录的结构编码为字符串或者将解析文本数据集，并且针对不同的用户对字段进行不同的投影），则可以通过三个步骤以编程方式创建DataFrame ：

1.  从原始RDD创建行的RDD； 
2. 在第1步中创建的RDD中，创建一个与StructType表示的模式匹配的行结构。 
3. 通过SparkSession提供的`createDataFrame`方法将模式应用于行的RDD。

```scala
import org.apache.spark.sql.types._

// Create an RDD
val peopleRDD = spark.sparkContext.textFile("examples/src/main/resources/people.txt")

// The schema is encoded in a string
val schemaString = "name age"

// Generate the schema based on the string of schema
val fields = schemaString.split(" ")
  .map(fieldName => StructField(fieldName, StringType, nullable = true))
val schema = StructType(fields)

// Convert records of the RDD (people) to Rows
val rowRDD = peopleRDD
  .map(_.split(","))
  .map(attributes => Row(attributes(0), attributes(1).trim))

// Apply the schema to the RDD
val peopleDF = spark.createDataFrame(rowRDD, schema)

// Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

// SQL can be run over a temporary view created using DataFrames
val results = spark.sql("SELECT name FROM people")

// The results of SQL queries are DataFrames and support all the normal RDD operations
// The columns of a row in the result can be accessed by field index or by field name
results.map(attributes => "Name: " + attributes(0)).show()
// +-------------+
// |        value|
// +-------------+
// |Name: Michael|
// |   Name: Andy|
// | Name: Justin|
// +-------------+
```

### 聚合

（todo 代码看不太懂）

## Data Sources

### 通用Load/Save函数

```scala
val usersDF = spark.read.load("examples/src/main/resources/users.parquet")
usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
```

### 手动指定选项

您还可以手动指定将要使用的数据源以及要传递给数据源的任何其他选项。数据源由它们的完全限定名称（即org.apache.spark.sql.parquet）指定，但是对于内置源，您还可以使用其短名称（json，parquet，jdbc，orc，libsvm，csv，text）。从任何数据源类型加载的DataFrame都可以使用此语法转换为其他类型。

```scala
val peopleDF = spark.read.format("json").load("examples/src/main/resources/people.json")
peopleDF.select("name", "age").write.format("parquet").save("namesAndAges.parquet")
```

### 直接对文件运行SQL

```scala
val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")
```

