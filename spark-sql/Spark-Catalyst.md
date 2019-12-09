### SparkSQL优化器之Spark-Catalyst组件
Spark-Catalyst是SparkSQL的核心组件。它负责将SQL语句转换成物理执行计划。

SQL->AST->UnresolvedLogicPlan->LoginPlan->PhysicalPlan->Rdd

其中QueryExecution是Catalyst模块内最重要的一个类。它是DataSet/DataFrame的组成部分，，QueryExecution保存每一步生成的执行计划，analyze后的plan存在analyzed变量内，optimize后的plan存在optimized变量中等等，所以开发者也可以通过QueryExecution清楚的了解Spark SQL的每一步转化操作



1) SqlParser将SQL语句解析成一个Unresolved LogicPlan
ParserDriver的parse方法，通过antlr4生成语法树


2) Analyzer对Unresolved LogicPlan做分析得到Analyzd LogicPlan
它借助Catalog内的元数据(如HiveMeta)解析Unresolved LogicPLan生成resolved LogicPLan，元数据包括两块：表的Schema信息和基本函数信息。通过元数据来判断执行计划内的表/字段是否存在、确定字段的数据类型、sum/avg等会被解析成特定的聚合函数等等。它也是QueryExecution的一个变量，源码定义如下：

```scala
lazy val analyzed: LogicalPlan = {
    SparkSession.setActiveSession(sparkSession)
    sparkSession.sessionState.analyzer.executeAndCheck(logical)
  }
```
可以看到通过SessionState中的其中executeAndCheck方法就是对第一步parser解析得到的Unresolved LogicPlan做analyze，过程如下：

a)executeAndCheck方法中调用execute方法

```scala
  def executeAndCheck(plan: LogicalPlan): LogicalPlan = AnalysisHelper.markInAnalyzer {
    val analyzed = execute(plan)
    try {
      checkAnalysis(analyzed)
      analyzed
    } catch {
      case e: AnalysisException =>
        val ae = new AnalysisException(e.message, e.line, e.startPosition, Option(analyzed))
        ae.setStackTrace(e.getStackTrace)
        throw ae
    }
  }
```
b) execute方法中再调用executeSameContext方法

```scala
override def execute(plan: LogicalPlan): LogicalPlan = {
    AnalysisContext.reset()
    try {
      executeSameContext(plan)
    } finally {
      AnalysisContext.reset()
    }
  }
```

c)executeSameContext方法调用父类RuleExecutor类的execute方法

```scala
private def executeSameContext(plan: LogicalPlan): LogicalPlan = super.execute(plan)
```

d)RuleEecutor的execute方法循环rule batch来完成analyzed的check操作

```scala
 batches.foreach {
 	while (continue) {
        curPlan = batch.rules.foldLeft(curPlan) {
          case (plan, rule) =>
            val startTime = System.nanoTime()
            val result = rule(plan)
        }
    }
 }
```
rule batch定义在Analyzer类中，是Spark定义好的各种analyzed规则：

 ```scala
 lazy val batches: Seq[Batch] = Seq(
    Batch("Hints", fixedPoint,
      new ResolveHints.ResolveBroadcastHints(conf),
      ResolveHints.ResolveCoalesceHints,
      ResolveHints.RemoveAllHints),
    Batch("Simple Sanity Check", Once,
      LookupFunctions),
    Batch("Substitution", fixedPoint,
      CTESubstitution,
      WindowsSubstitution,
      EliminateUnions,
      new SubstituteUnresolvedOrdinals(conf)),
    Batch("Resolution", fixedPoint,
      ResolveTableValuedFunctions ::
      ResolveRelations ::
      ResolveReferences ::
      ResolveCreateNamedStruct ::
      ResolveDeserializer ::
      ResolveNewInstance ::
      ResolveUpCast ::
      ResolveGroupingAnalytics ::
      ResolvePivot ::
      ResolveOrdinalInOrderByAndGroupBy ::
      ResolveAggAliasInGroupBy ::
      ResolveMissingReferences ::
      ExtractGenerator ::
      ResolveGenerate ::
      ResolveFunctions ::
      ResolveAliases ::
      ResolveSubquery ::
      ...
  )
 ```
  比如ResolveRelations这个rule会判断表是否存在，rule实现如下：
 
 ```scala
 object ResolveRelations extends Rule[LogicalPlan] {
 	def resolveRelation(plan: LogicalPlan): LogicalPlan = plan match {
 		case u: UnresolvedRelation if !isRunningDirectlyOnFiles(u.tableIdentifier) =>
        val defaultDatabase = AnalysisContext.get.defaultDatabase
        val foundRelation = lookupTableFromCatalog(u, defaultDatabase)
        ...
 	}
 }
 ```
每个rule都有自己的实现方法，使用每个rule定义好的方法，就完成了对LogicPlan的Analyzed操作。最终通过checkAnalysis对Analyzed的结果做处理，checkAnalysis内定义了各种校验失败的处理方式和日志输出，比如表不存在：

```scala
 case u: UnresolvedRelation =>
        u.failAnalysis(s"Table or view not found: ${u.tableIdentifier}")
```
如果校验全部通过，checkAnalysis将plan的Analyzed标志设为true，表示Analyzed正确完成。

3)Optimizer对Unresolved LogicPlan优化得到optimized LogicPlan
 sparkSession.sessionState.optimizer.execute(withCachedData)

4)SparkPlanner将逻辑执行计划转换成物理执行计划





