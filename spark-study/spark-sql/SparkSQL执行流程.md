### SparkSQL执行流程
本文基于Spark2.4。

主要从源码角度分析SparkSQL的执行流程。先看下列语句：

```scala
var df = spark.sql("select * from user where age > 10")
df.head(10)
```
以上是在Spark-Shell交互命令行执行的一段SparkSQL代码，从一张hive的user表查询年龄大于10的用户数据。通过SparkSession的sql方法，传入sql语句，执行得到一个DataFrame对象，再通过show()方法，查看查询的结果数据。那Spark内部具体是怎么完成整个操作的呢，下面从Spark-SQL的源码来做一下分析。

SparkSQL的入口是SparkSession的SQL方法，方法源码如下：

```scala
def sql(sqlText: String): DataFrame = {
    Dataset.ofRows(self, sessionState.sqlParser.parsePlan(sqlText))
 }
```
可以看到内部调用Dataset的ofRows方法。首先SQL语句会被parsePlan方法解析生成一个LogicPlan对象，方法源码如下：

```scala
/** Creates LogicalPlan for a given SQL string. */
  override def parsePlan(sqlText: String): LogicalPlan = parse(sqlText) { parser =>
    astBuilder.visitSingleStatement(parser.singleStatement()) match {
      case plan: LogicalPlan => plan
      case _ =>
        val position = Origin(None, None)
        throw new ParseException(Option(sqlText), "Unsupported SQL statement", position, position)
    }
}
```

接着看一下ofRows方法：

```scala
def ofRows(sparkSession: SparkSession, logicalPlan: LogicalPlan): DataFrame = {
    val qe = sparkSession.sessionState.executePlan(logicalPlan)
    qe.assertAnalyzed()
    new Dataset[Row](sparkSession, qe, RowEncoder(qe.analyzed.schema))
  }
```

可以看到，在方法内，首先使用LogicPlan对象来构造生成QueryExecution对象qe,QueryExecution包括以下几个重要参数：

```scala
lazy val analyzed: LogicalPlan = {
    SparkSession.setActiveSession(sparkSession)
    sparkSession.sessionState.analyzer.executeAndCheck(logical)
  }

  lazy val withCachedData: LogicalPlan = {
    assertAnalyzed()
    assertSupported()
    sparkSession.sharedState.cacheManager.useCachedData(analyzed)
  }

  lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)

  lazy val sparkPlan: SparkPlan = {
    SparkSession.setActiveSession(sparkSession)
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
    //       but we will implement to choose the best plan.
    planner.plan(ReturnAnswer(optimizedPlan)).next()
  }

  // executedPlan should not be used to initialize any SparkPlan. It should be
  // only used for execution.
  lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)

  /** Internal version of the RDD. Avoids copies and has no schema */
  lazy val toRdd: RDD[InternalRow] = executedPlan.execute()
```
这几个属性都是lazy(延迟加载的)，分别代表SQL执行流程的几个关键步骤，经过Analyze生成analyzed、经过Optimize生成optimizedPlan、再通过plan方法将LogicPlan转化为physicalPlan对象SparkPlan，最后调用SparkPlan的execute方法，将physicalPlan转化为一个RDD，这样就完成了SQL到RDD的转化。
接着用sparkSession、qe和Row编码器一起构造得到一个Dataset对象。

到这里为止，SparkSession的sql()已经执行完毕了，方法执行最终得到一个Dataset对象，其中包含Q
QueryExecution这个重要对象。这一切都是在driver端执行，还没有触发计算任务，只有执行Dataset对象的action，如show、head或take等方法，才会真正触发计算任务，这就是第二阶段的工作。

我们接着看一下第二阶段工作的实现源码，以Dataset.head(10)方法为例,方法作用是取SQL执行结果的前10条记录。head方法内执行withAction方法，这是Dataset类内部最重要的一个方法，所有的action操作，最终都会调用这个方法来触发计算任务：

```scala
def head(n: Int): Array[T] = withAction("head", limit(n).queryExecution)(collectFromPlan)
```
withAction方法源码如下：

```scala
private def withAction[U](name: String, qe: QueryExecution)(action: SparkPlan => U) = {
    try {
      qe.executedPlan.foreach { plan =>
        plan.resetMetrics()
      }
      val start = System.nanoTime()
      val result = SQLExecution.withNewExecutionId(sparkSession, qe) {
        action(qe.executedPlan)
      }
      val end = System.nanoTime()
      sparkSession.listenerManager.onSuccess(name, qe, end - start)
      result
    } catch {
      case e: Exception =>
        sparkSession.listenerManager.onFailure(name, qe, e)
        throw e
    }
  }
```
其中最主要的一句代码是：

```scala
val result = SQLExecution.withNewExecutionId(sparkSession, qe) {
	action(qe.executedPlan)
}
```
这里传入QueryExecution的executedPlan(SparkPlan)，执行action函数得到result。action函数由withAction函数的调用方自己传入，在action方法中执行SparkPlan，聚合计算结果返回。以head方法为例，传入的action函数是collectFromPlan，而collectFromPlan方法中会执行SparkPlan的executeCollect方法：

```scala
 /**
   * Runs this query returning the result as an array.
   */
  def executeCollect(): Array[InternalRow] = {
    val byteArrayRdd = getByteArrayRdd()

    val results = ArrayBuffer[InternalRow]()
    byteArrayRdd.collect().foreach { countAndBytes =>
      decodeUnsafeRows(countAndBytes._2).foreach(results.+=)
    }
    results.toArray
  }
```
执行getByteArrayRdd方法得到一个RDD，接着执行rdd的collect()方法，这一步才触发了Spark的计算任务，collect方法是通过SparkContext的runJob方法来提交计算任务，方法源码：

```scala
def collect(): Array[T] = withScope {
    val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
    Array.concat(results: _*)
  }
```
sc是SparkContext对象，可以看到执行了runJon方法，runJob方法内部，最后则是通过调用DagScheduler的runJob方法和submitJob方法，sumbitJob方法把DAG和任务信息包装成JobWaiter发送到任务队列，等待提交到集群执行，JobWaiter会返回给runjob，再通过completionFuture机制在runJob方法中调用JobWaiter的get方法，异步的获取Spark计算任务执行结果：

submitJob方法：

```scala
  def submitJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): JobWaiter[U] = {
    // Check to make sure we are not launching a task on a partition that does not exist.
    val maxPartitions = rdd.partitions.length
    partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
      throw new IllegalArgumentException(
        "Attempting to access a non-existent partition: " + p + ". " +
          "Total number of partitions: " + maxPartitions)
    }

    val jobId = nextJobId.getAndIncrement()
    if (partitions.size == 0) {
      // Return immediately if the job is running 0 tasks
      return new JobWaiter[U](this, jobId, 0, resultHandler)
    }

    assert(partitions.size > 0)
    val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
    val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
    eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      SerializationUtils.clone(properties)))
    waiter
  }
```

runjob方法：

```scala
  def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
    val start = System.nanoTime
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
    ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
    waiter.completionFuture.value.get match {
      case scala.util.Success(_) =>
        logInfo("Job %d finished: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      case scala.util.Failure(exception) =>
        logInfo("Job %d failed: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
        val callerStackTrace = Thread.currentThread().getStackTrace.tail
        exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
        throw exception
    }
  }
```
这样就完成了一条SparkSQL的执行。而在执行过程中，SQL语句到RDD的转化，需要经过解析、分析、优化、逻辑计划转化为物理计划等操作，这是由Spark Catalyst模块来完成的，在后续文章中再做详细分析。