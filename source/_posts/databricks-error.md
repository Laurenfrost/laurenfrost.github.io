---
title: Azure Databricks 踩坑记录
date: 2021-01-12 16:35:23
tags: spark
category: 踩坑记录
---

开宗明义：傻逼 Databricks！真tmd傻逼！！！！！！！！

> 无数次的经验告诉我，instance 的内存大小非常重要！！！  
> instance 的数量只能让更多的 job 同时运行，也就是跑得更快。但真正决定程序能不能运行的还是 instance 的内存大小。  
> 如果你的 instance 跑不了 spark 划分的最大的一个 job，那整个程序就跑不了。  
> 问就是加钱。

## databricks-connect 问题
不要以为你的程序打包好之后，上传 Azure Databricks 运行也没问题，就可以高枕无忧。databricks-connect 总是能比你想象得更傻逼！

### Cluster Setup

> First you need to enable the feature on your Databricks cluster. Your cluster must be using Databricks Runtime 5.1 or higher. In the web UI edit your cluster and add this/these lines to the spark.conf:
>
> spark.databricks.service.server.enabled true
> If you are using Azure Databricks also add this line:
> 
> spark.databricks.service.port 8787
> (Note the single space between the setting name and value).
> 
> Restart your cluster.
[参考这里](https://datathirst.net/blog/2019/3/7/databricks-connect-finally)

效果大概是这样：

![databricks-cluster-config](./databricks-cluster-config.png)

*在这里你调整了 port 之后，别忘了你`databricks-connect configure`的时候，相应的 port 也需要调整，不然它连接不上。*

### 报错 `A master URL must be set`
虽然 Azure Databrick 在针对 Scala 的编程建议里写道:

> The code should use SparkContext.getOrCreate to obtain a Spark context; otherwise, runs of the job will fail.
> 也就是说，类似于写成这样：
> ```scala
> val spark = SparkSession.builder().getOrCreate()
> ```

它说得确实没错，因为当你打包 jar 上传之后，它已经在相应的 spark 环境里了。它是能直接 get 到 Databricks 提供给它的 context 的。但当你使用 databricks-connect 提供的那一套东西的时候，你就只能 "create" 一个 context。  

所以你要是想用它，你就必须指定 `master` 为 `local`，就像这样：

```scala
val spark = SparkSession.builder().master("local[*]").getOrCreate()
```

然后它才会尝试连接到 Azure Databrick 的 cluster。

### 报错 `Result for RPC Some lost`
完整的报错信息是：
```
Exception in thread "main" java.lang.IllegalArgumentException: requirement failed: Result for RPC Some(c74963ef-2411-4c05-9c06-bdd2faeba2ff) lost, please retry your request.
```

网络波动，重试一遍就好。


### 报错 `java class may not be present on the remote cluster`
完整的报错信息是：
```
Exception in thread "main" com.databricks.service.DependencyCheckWarning: The java class <something> may not be present on the remote cluster. It can be found in <something>/target/scala-2.11/classes. To resolve this, package the classes in <something>/target/scala-2.11/classes into a jar file and then call sc.addJar() on the package jar. You can disable this check by setting the SQL conf spark.databricks.service.client.checkDeps=false.
```

这个讲道理我也没有什么好办法。

+ 思路 1
    参考这个[StackOverflow的答案](https://stackoverflow.com/questions/60510382/databricks-connect-dependencycheckwarning-the-java-class-may-not-be-present-on)，虽然它看起来并不怎么靠谱。

    具体而言，就是遵从报错信息的指示，把 class 打包成 jar，然后用 `sc.addJar()` 加载这个 jar。

    ```scala
    spark.sparkContext.addJar("你的 jar 文件地址")
    ```

    我也不知道它究竟有没有成效，但至少报错变了。

+ 思路 2
    这个思路就是遵从报错信息的另一个指示，通过设置来禁用这个 check，不让他报错。

    虽然有点掩耳盗铃之嫌，不过我还是试了一下。毕竟“黑猫白猫，逮住老鼠就是好猫”。

    ```scala
    val spark = SparkSession
        .builder()
        .config("spark.databricks.service.client.checkDeps", "false")
        .master("local[*]")
        .getOrCreate()
    ```

    后来报错确实变了，变成了这样。这下是彻底找不到 class 了。

    ```
    Exception in thread "main" java.io.InvalidClassException: failed to read class descriptor
        at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1938)
        at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1830)
        at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2121)
        at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1647)
        at java.io.ObjectInputStream.readObject(ObjectInputStream.java:483)
        at java.io.ObjectInputStream.readObject(ObjectInputStream.java:441)
        at org.apache.spark.sql.util.ProtoSerializer.deserializeObject(ProtoSerializer.scala:4289)
        at org.apache.spark.sql.util.ProtoSerializer$$anonfun$deserializeContext$1.apply(ProtoSerializer.scala:259)
        at org.apache.spark.sql.util.ProtoSerializer$$anonfun$deserializeContext$1.apply(ProtoSerializer.scala:258)
        at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
        at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:234)
        at scala.collection.Iterator$class.foreach(Iterator.scala:891)
        at scala.collection.AbstractIterator.foreach(Iterator.scala:1334)
        at scala.collection.IterableLike$class.foreach(IterableLike.scala:72)
        at scala.collection.AbstractIterable.foreach(Iterable.scala:54)
        at scala.collection.TraversableLike$class.map(TraversableLike.scala:234)
        at scala.collection.AbstractTraversable.map(Traversable.scala:104)
        at org.apache.spark.sql.util.ProtoSerializer.deserializeContext(ProtoSerializer.scala:258)
        at org.apache.spark.sql.util.ProtoSerializer$$anonfun$deserializePlan$1.apply(ProtoSerializer.scala:2882)
        at org.apache.spark.sql.util.ProtoSerializer$$anonfun$deserializePlan$1.apply(ProtoSerializer.scala:2882)
        at scala.Option.foreach(Option.scala:257)
        at org.apache.spark.sql.util.ProtoSerializer.deserializePlan(ProtoSerializer.scala:2882)
        at com.databricks.service.SparkServiceRPCHandler.com$databricks$service$SparkServiceRPCHandler$$execute0(SparkServiceRPCHandler.scala:454)
        at com.databricks.service.SparkServiceRPCHandler$$anonfun$executeRPC0$1.apply(SparkServiceRPCHandler.scala:343)
        at com.databricks.service.SparkServiceRPCHandler$$anonfun$executeRPC0$1.apply(SparkServiceRPCHandler.scala:298)
        at scala.util.DynamicVariable.withValue(DynamicVariable.scala:58)
        at com.databricks.service.SparkServiceRPCHandler.executeRPC0(SparkServiceRPCHandler.scala:298)
        at com.databricks.service.SparkServiceRPCHandler$$anon$3.call(SparkServiceRPCHandler.scala:255)
        at com.databricks.service.SparkServiceRPCHandler$$anon$3.call(SparkServiceRPCHandler.scala:251)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at com.databricks.service.SparkServiceRPCHandler$$anonfun$executeRPC$1.apply(SparkServiceRPCHandler.scala:287)
        at com.databricks.service.SparkServiceRPCHandler$$anonfun$executeRPC$1.apply(SparkServiceRPCHandler.scala:267)
        at scala.util.DynamicVariable.withValue(DynamicVariable.scala:58)
        at com.databricks.service.SparkServiceRPCHandler.executeRPC(SparkServiceRPCHandler.scala:266)
        at com.databricks.service.SparkServiceRPCServlet.doPost(SparkServiceRPCServer.scala:108)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:707)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:790)
        at org.eclipse.jetty.servlet.ServletHolder.handle(ServletHolder.java:848)
        at org.eclipse.jetty.servlet.ServletHandler.doHandle(ServletHandler.java:584)
        at org.eclipse.jetty.servlet.ServletHandler.doScope(ServletHandler.java:514)
        at org.eclipse.jetty.server.handler.ScopedHandler.handle(ScopedHandler.java:141)
        at org.eclipse.jetty.server.handler.HandlerWrapper.handle(HandlerWrapper.java:134)
        at org.eclipse.jetty.server.Server.handle(Server.java:534)
        at org.eclipse.jetty.server.HttpChannel.handle(HttpChannel.java:320)
        at org.eclipse.jetty.server.HttpConnection.onFillable(HttpConnection.java:251)
        at org.eclipse.jetty.io.AbstractConnection$ReadCallback.succeeded(AbstractConnection.java:283)
        at org.eclipse.jetty.io.FillInterest.fillable(FillInterest.java:108)
        at org.eclipse.jetty.io.SelectChannelEndPoint$2.run(SelectChannelEndPoint.java:93)
        at org.eclipse.jetty.util.thread.strategy.ExecuteProduceConsume.executeProduceConsume(ExecuteProduceConsume.java:303)
        at org.eclipse.jetty.util.thread.strategy.ExecuteProduceConsume.produceConsume(ExecuteProduceConsume.java:148)
        at org.eclipse.jetty.util.thread.strategy.ExecuteProduceConsume.run(ExecuteProduceConsume.java:136)
        at org.eclipse.jetty.util.thread.QueuedThreadPool.runJob(QueuedThreadPool.java:671)
        at org.eclipse.jetty.util.thread.QueuedThreadPool$2.run(QueuedThreadPool.java:589)
        at java.lang.Thread.run(Thread.java:748)
    Caused by: java.lang.ClassNotFoundException: com.optimind.copernicus.spark.DataGetters$$anonfun$2
        at java.lang.ClassLoader.findClass(ClassLoader.java:523)
        at org.apache.spark.util.ParentClassLoader.findClass(ParentClassLoader.java:35)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
        at org.apache.spark.util.ParentClassLoader.loadClass(ParentClassLoader.java:40)
        at org.apache.spark.util.ChildFirstURLClassLoader.loadClass(ChildFirstURLClassLoader.java:48)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
        at java.lang.Class.forName0(Native Method)
        at java.lang.Class.forName(Class.java:348)
        at org.apache.spark.util.Utils$.classForName(Utils.scala:257)
        at org.apache.spark.sql.util.ProtoSerializer.org$apache$spark$sql$util$ProtoSerializer$$readResolveClassDescriptor(ProtoSerializer.scala:4298)
        at org.apache.spark.sql.util.ProtoSerializer$$anon$4.readClassDescriptor(ProtoSerializer.scala:4286)
        at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1936)
        ... 53 more
    ```

### 报错 `cannot assign instance of scala.None$ to field`
完整的报错信息是：
```
Exception in thread "main" org.apache.spark.SparkException: Job aborted due to stage failure: Exception while getting task result: java.io.IOException: java.lang.ClassCastException: cannot assign instance of scala.None$ to field org.apache.spark.util.AccumulatorMetadata.name of type scala.Option in instance of org.apache.spark.util.AccumulatorMetadata
```

我也不知道这段天书在说啥，毕竟我没有清楚地了解 spark 的内部工作机理。

只是个人猜测大概是本地的 spark 和 Azure Databricks 的 spark 之间没有协调好。或许我应该调整一下依赖？

---

重整旗鼓，让我们看看这个 `AccumulatorMetadata` 是个什么鸡巴？[spark 源码](https://fossies.org/linux/spark/core/src/main/scala/org/apache/spark/util/AccumulatorV2.scala)：

```scala
   18 package org.apache.spark.util
    ...
   30 private[spark] case class AccumulatorMetadata(
   31     id: Long,
   32     name: Option[String],
   33     countFailedValues: Boolean) extends Serializable
   34 
   35 
   36 /**
   37  * The base class for accumulators, that can accumulate inputs of type `IN`, and produce output of
   38  * type `OUT`.
   39  *
   40  * `OUT` should be a type that can be read atomically (e.g., Int, Long), or thread-safely
   41  * (e.g., synchronized collections) because it will be read from other threads.
   42  */
   43 abstract class AccumulatorV2[IN, OUT] extends Serializable {
   44   private[spark] var metadata: AccumulatorMetadata = _
   45   private[this] var atDriverSide = true
    ...
   81   /**
   82    * Returns the name of this accumulator, can only be called after registration.
   83    */
   84   final def name: Option[String] = {
   85     assertMetadataNotNull()
   86 
   87     if (atDriverSide) {
   88       metadata.name.orElse(AccumulatorContext.get(id).flatMap(_.metadata.name))
   89     } else {
   90       metadata.name
   91     }
   92   }
    ...
```
<s>所以这是个 spark-3.0 的 API，而我要的环境是 Databricks 5.5 LTS 对应的是 spark-2.4.3，这一定有问题。</s>

是我搞错了，通过查阅[文档](https://spark.apache.org/docs/2.4.3/api/scala/index.html#org.apache.spark.util.AccumulatorV2)和[源码](https://github.com/apache/spark/blob/v2.4.3/core/src/main/scala/org/apache/spark/util/AccumulatorV2.scala)，其实在 2.4.3 里是有这个接口的，基本就是上面这个代码，所以我的这个猜测是错的。(跪了)

---

When I package jar file, those dependences are also copied into the file. And some of them have conflict with runtime env in remote cluster, which leads to the error `cannot assign instance of scala.None$ to field`. So, configure build.sbt and prevent unnecessary libraries would help. (I hope so.)

### 报错 `Failed to add jar file`
完整的报错信息：
```
2021-01-19 08:18:06,056 ERROR [main] SparkContext: Failed to add ./jars/com.optimind.copernicus.spark.jar to Spark environment
java.lang.IllegalArgumentException: Error while instantiating 'org.apache.spark.sql.internal.SessionStateBuilder':
	at org.apache.spark.sql.SparkSession$.org$apache$spark$sql$SparkSession$$instantiateSessionState(SparkSession.scala:1178) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession$$anonfun$sessionState$2.apply(SparkSession.scala:170) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession$$anonfun$sessionState$2.apply(SparkSession.scala:169) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at scala.Option.getOrElse(Option.scala:121) ~[scala-library-2.11.12.jar:na]
	at org.apache.spark.sql.SparkSession.sessionState$lzycompute(SparkSession.scala:169) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession.sessionState(SparkSession.scala:166) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession.conf$lzycompute(SparkSession.scala:198) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession.conf(SparkSession.scala:198) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.service.SparkClient$.clientEnabled(SparkClient.scala:206) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.spark.util.SparkClientContext$.clientEnabled(SparkClientContext.scala:108) ~[spark-core_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.SparkContext.addJarFile$1(SparkContext.scala:1996) [spark-core_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.SparkContext.addJar(SparkContext.scala:2021) [spark-core_2.11-2.4.3.jar:2.4.3]
Caused by: java.util.concurrent.TimeoutException: null
	at org.spark_project.jetty.client.util.FutureResponseListener.get(FutureResponseListener.java:109) ~[spark-core_2.11-2.4.3.jar:2.4.3]
	at org.spark_project.jetty.client.HttpRequest.send(HttpRequest.java:676) ~[spark-core_2.11-2.4.3.jar:2.4.3]
	at com.databricks.service.DBAPIClient.get(DBAPIClient.scala:81) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.service.DBAPIClient.jsonGet(DBAPIClient.scala:108) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.service.SparkServiceDebugHelper$.validateSparkServiceToken(SparkServiceDebugHelper.scala:130) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.service.SparkClientManager$class.getForCurrentSession(SparkClient.scala:287) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.service.SparkClientManager$.getForCurrentSession(SparkClient.scala:366) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.service.SparkClient$.getServerHadoopConf(SparkClient.scala:245) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at com.databricks.spark.util.SparkClientContext$.getServerHadoopConf(SparkClientContext.scala:222) ~[spark-core_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.SparkContext$$anonfun$hadoopConfiguration$1.apply(SparkContext.scala:317) ~[spark-core_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.SparkContext$$anonfun$hadoopConfiguration$1.apply(SparkContext.scala:312) ~[spark-core_2.11-2.4.3.jar:2.4.3]
	at scala.util.DynamicVariable.withValue(DynamicVariable.scala:58) ~[scala-library-2.11.12.jar:na]
	at org.apache.spark.SparkContext.hadoopConfiguration(SparkContext.scala:311) [spark-core_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.internal.SharedState.<init>(SharedState.scala:67) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession$$anonfun$sharedState$1.apply(SparkSession.scala:145) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession$$anonfun$sharedState$1.apply(SparkSession.scala:145) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at scala.Option.getOrElse(Option.scala:121) ~[scala-library-2.11.12.jar:na]
	at org.apache.spark.sql.SparkSession.sharedState$lzycompute(SparkSession.scala:145) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession.sharedState(SparkSession.scala:144) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.internal.BaseSessionStateBuilder.build(BaseSessionStateBuilder.scala:291) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	at org.apache.spark.sql.SparkSession$.org$apache$spark$sql$SparkSession$$instantiateSessionState(SparkSession.scala:1175) ~[spark-sql_2.11-2.4.3.jar:2.4.3]
	... 13 common frames omitted
```


网络波动，重试一遍就好。
关闭 clash 的 mixin 功能。


### 报错 `cannot assign instance`
完整的报错信息：
```
Exception in thread "main" org.apache.spark.SparkException: Job aborted due to stage failure: Task 5 in stage 45.0 failed 4 times, most recent failure: Lost task 5.3 in stage 45.0 (TID 1846, 10.139.64.5, executor 0): java.lang.ClassCastException: cannot assign instance of org.slf4j.impl.Log4jLoggerAdapter to field ch.qos.logback.classic.Logger.parent of type ch.qos.logback.classic.Logger in instance of ch.qos.logback.classic.Logger
```

我猜大概是我 lib/ 目录下里有个 fat jar 里有 logger 的库。待我删了它。

```
21/01/19 09:16:11 WARN SparkEnv: Exception while deleting Spark temp dir: C:\Users\Bellf\AppData\Local\Temp\spark-ecd10e21-e079-41ac-b327-c88fe36b9c65\userFiles-59b6b2e4-3139-48e7-9836-00b083054018
java.io.IOException: Failed to delete: C:\Users\Bellf\AppData\Local\Temp\spark-ecd10e21-e079-41ac-b327-c88fe36b9c65\userFiles-59b6b2e4-3139-48e7-9836-00b083054018\slf4j-log4j12-1.7.16.jar
	at org.apache.spark.network.util.JavaUtils.deleteRecursivelyUsingJavaIO(JavaUtils.java:144)
	at org.apache.spark.network.util.JavaUtils.deleteRecursively(JavaUtils.java:118)
	at org.apache.spark.network.util.JavaUtils.deleteRecursivelyUsingJavaIO(JavaUtils.java:128)
	at org.apache.spark.network.util.JavaUtils.deleteRecursively(JavaUtils.java:118)
	at org.apache.spark.network.util.JavaUtils.deleteRecursively(JavaUtils.java:91)
	at org.apache.spark.util.Utils$.deleteRecursively(Utils.scala:1187)
	at org.apache.spark.SparkEnv.stop(SparkEnv.scala:159)
	at org.apache.spark.SparkContext$$anonfun$stop$11.apply$mcV$sp(SparkContext.scala:2134)
	at org.apache.spark.util.Utils$.tryLogNonFatalError(Utils.scala:1506)
	at org.apache.spark.SparkContext.stop(SparkContext.scala:2133)
	at org.apache.spark.SparkContext$$anonfun$2.apply$mcV$sp(SparkContext.scala:629)
	at org.apache.spark.util.SparkShutdownHook.run(ShutdownHookManager.scala:216)
	at org.apache.spark.util.SparkShutdownHookManager$$anonfun$runAll$1$$anonfun$apply$mcV$sp$1.apply$mcV$sp(ShutdownHookManager.scala:188)
	at org.apache.spark.util.SparkShutdownHookManager$$anonfun$runAll$1$$anonfun$apply$mcV$sp$1.apply(ShutdownHookManager.scala:188)
	at org.apache.spark.util.SparkShutdownHookManager$$anonfun$runAll$1$$anonfun$apply$mcV$sp$1.apply(ShutdownHookManager.scala:188)
	at org.apache.spark.util.Utils$.logUncaughtExceptions(Utils.scala:2121)
	at org.apache.spark.util.SparkShutdownHookManager$$anonfun$runAll$1.apply$mcV$sp(ShutdownHookManager.scala:188)
	at org.apache.spark.util.SparkShutdownHookManager$$anonfun$runAll$1.apply(ShutdownHookManager.scala:188)
	at org.apache.spark.util.SparkShutdownHookManager$$anonfun$runAll$1.apply(ShutdownHookManager.scala:188)
	at scala.util.Try$.apply(Try.scala:192)
	at org.apache.spark.util.SparkShutdownHookManager.runAll(ShutdownHookManager.scala:188)
	at org.apache.spark.util.SparkShutdownHookManager$$anon$2.run(ShutdownHookManager.scala:178)
	at org.apache.hadoop.util.ShutdownHookManager$1.run(ShutdownHookManager.java:54)
```

## Databricks Notebook 问题

### 报错 `not found value expr`
问题代码：
```scala
val nodes = spark.read.format("jdbc")......load()
nodes.withColumn("shapeWKT", expr("ST_GeomFromWKB(shape)")).show()
```

报错内容：
```log
command-633362346182315:33: error: not found: value expr
nodes.withColumn("shapeWKT", expr("ST_GeomFromWKB(shape)")).show()
```

解决办法：
```scala
import org.apache.spark.sql.functions._
```

参考资料：  
[https://stackoverflow.com/questions/36330024/sparksql-not-found-value-expr/36330128](https://stackoverflow.com/questions/36330024/sparksql-not-found-value-expr/36330128)

### 报错 `Undefined function: 'ST_GeomFromWKB'`
问题代码：
```scala
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._

val nodes = spark.read.format("jdbc")......load()
nodes.withColumn("shapeWKT", expr("ST_GeomFromWKB(shape)")).show()
```

报错内容：
```log
org.apache.spark.sql.AnalysisException: Undefined function: 'ST_GeomFromWKB'.
 This function is neither a registered temporary function nor a permanent function registered in the database 'default'.; line 1 pos 0
```

解决办法：  
在 cluster 的 advanced option 里，给 spark config 添加下列语句：
```
spark.kryo.registrator
org.datasyslab.geospark.serde.GeoSparkKryoRegistrator
```
然后 import 相关库：
```scala
import org.apache.spark.serializer.KryoSerializer
import org.datasyslab.geosparkviz.sql.utils.GeoSparkVizRegistrator
import org.datasyslab.geosparksql.utils.{Adapter, GeoSparkSQLRegistrator}
import org.datasyslab.geosparkviz.core.Serde.GeoSparkVizKryoRegistrator
GeoSparkSQLRegistrator.registerAll(spark)
GeoSparkVizRegistrator.registerAll(spark)
```

参考资料：  
[https://stackoverflow.com/questions/62830434/st-geomfromtext-function-using-spark-java](https://stackoverflow.com/questions/62830434/st-geomfromtext-function-using-spark-java)

### 报错 `Can not create the managed table`
简而言之就是不能 overwrite。

报错代码：
```scala
SomeData_df.write.mode("overwrite").saveAsTable("SomeData")
```

报错内容：
```log
org.apache.spark.sql.AnalysisException: Can not create the managed table('SomeData'). 
The associated location('dbfs:/user/hive/warehouse/somedata') already exists.;
```

解决办法：
明明是 overwrite 模式，但还是不行，真傻逼。

+ 思路1：直接删除  
  它不是已经存在了吗，删了就完事儿了。
  ```scala
  dbutils.fs.rm("dbfs:/user/hive/warehouse/SomeData", recurse=true)
  ```

+ 思路2：调整 Databricks 设置
  ```scala
  spark.conf.set("spark.sql.legacy.allowCreatingManagedTableUsingNonemptyLocation","true")
  ```

  但这个选项在 Spark 3.0.0 中被去掉了。如果用更高版本的 Databricks 集群的话就不行。会有这样的报错：
  ```log
  Caused by: org.apache.spark.sql.AnalysisException: The SQL config 'spark.sql.legacy.allowCreatingManagedTableUsingNonemptyLocation' was removed in the version 3.0.0.
  It was removed to prevent loosing of users data for non-default value.;
  ```
  要解决这个问题，必须写出完整的文件路径才行。

+ 思路3：直接 overwrite 文件的绝对路径。
  ```scala
  df.write \
    .option("path", "hdfs://cluster_name/path/to/my_db") \
    .mode("overwrite") \
    .saveAsTable("my_db.my_table")
  ```

+ 思路4：修改 cluster 配置  
  这个基本上是思路 2 的加强版。  
  在 cluster 的 advanced option 里，给 spark config 添加下列语句：
  ```
  spark.sql.legacy.allowCreatingManagedTableUsingNonemptyLocation true
  ```

参考资料：  
+ [https://docs.microsoft.com/en-us/azure/databricks/kb/jobs/spark-overwrite-cancel](https://docs.microsoft.com/en-us/azure/databricks/kb/jobs/spark-overwrite-cancel)
+ [https://stackoverflow.com/questions/55380427/azure-databricks-can-not-create-the-managed-table-the-associated-location-alre](https://stackoverflow.com/questions/55380427/azure-databricks-can-not-create-the-managed-table-the-associated-location-alre)


## Databricks Cluster 问题

### Java heap space 不够
`java.lang.OutOfMemoryError: Java heap space`
```
21/02/10 10:41:45 WARN TaskSetManager: Lost task 22.2 in stage 13.0 (TID 47942, 10.139.64.12, executor 19): java.lang.OutOfMemoryError: Java heap space
	at java.util.HashMap.resize(HashMap.java:704)
	at java.util.HashMap.putVal(HashMap.java:663)
	at java.util.HashMap.put(HashMap.java:612)
	at com.bmwcarit.barefoot.topology.Graph.construct(Graph.java:94)
	at com.bmwcarit.barefoot.roadmap.RoadMap.constructNetworkTopology(RoadMap.kt:147)
	at com.optimind.copernicus.spark.Main$$anonfun$4.apply(Main.scala:103)
	at com.optimind.copernicus.spark.Main$$anonfun$4.apply(Main.scala:86)
	at org.apache.spark.sql.execution.MapGroupsExec$$anonfun$10$$anonfun$apply$4.apply(objects.scala:365)
	at org.apache.spark.sql.execution.MapGroupsExec$$anonfun$10$$anonfun$apply$4.apply(objects.scala:364)
	at scala.collection.Iterator$$anon$12.nextCur(Iterator.scala:435)
	at scala.collection.Iterator$$anon$12.hasNext(Iterator.scala:441)
	at org.apache.spark.sql.catalyst.expressions.GeneratedClass$GeneratedIteratorForCodegenStage3.processNext(Unknown Source)
	at org.apache.spark.sql.execution.BufferedRowIterator.hasNext(BufferedRowIterator.java:43)
	at org.apache.spark.sql.execution.WholeStageCodegenExec$$anonfun$13$$anon$1.hasNext(WholeStageCodegenExec.scala:640)
	at org.apache.spark.sql.execution.datasources.FileFormatWriter$.org$apache$spark$sql$execution$datasources$FileFormatWriter$$executeTask(FileFormatWriter.scala:235)
	at org.apache.spark.sql.execution.datasources.FileFormatWriter$$anonfun$write$1.apply(FileFormatWriter.scala:173)
	at org.apache.spark.sql.execution.datasources.FileFormatWriter$$anonfun$write$1.apply(FileFormatWriter.scala:172)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:90)
	at org.apache.spark.scheduler.Task.doRunTask(Task.scala:140)
	at org.apache.spark.scheduler.Task.run(Task.scala:113)
	at org.apache.spark.executor.Executor$TaskRunner$$anonfun$17.apply(Executor.scala:606)
	at org.apache.spark.util.Utils$.tryWithSafeFinally(Utils.scala:1541)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:612)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
  ```

解决办法很简单，用更大的instance。

![你需要更大的instance](./more-mem-instance.png)

--- 分界线 ---

2021-08-24 更新：

不花钱怎么能跑程序？

无数次的经验告诉我，instance 的内存大小非常重要！！！

instance 的数量只能让更多的 job 同时运行，也就是跑得更快。但真正决定程序能不能运行的还是 instance 的内存大小。

如果你的 instance 跑不了 spark 划分的最大的一个 job，那整个程序就跑不了。

![更多更多内存](./more-and-more-mem-instance.png)
