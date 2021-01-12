---
title: Azure Databricks 踩坑记录
date: 2021-01-12 16:35:23
tags: spark
category: 踩坑记录
---

开宗明义：傻逼 Databricks！真tmd傻逼！！！！！！！！

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

这个 `AccumulatorMetadata` 是个什么鸡巴？让我们看看 spark [源码](https://fossies.org/linux/spark/core/src/main/scala/org/apache/spark/util/AccumulatorV2.scala)：

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
所以这是个 spark-3.0 的 API，而我要的环境是 Databricks 5.5 LTS 对应的是 spark-2.4.3，这一定有问题。
