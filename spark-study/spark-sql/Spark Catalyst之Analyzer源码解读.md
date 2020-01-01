###Spark Catalyst之Analyzer源码解读
Analyzer是Spark Catalyst模块内的重要组件，负责处理Parser解析SQL生成的Unresolved LogicPLan，它借助Catalog内的元数据(如HiveMeta)解析Unresolved LogicPLan生成resolved LogicPLan，元数据包括两块：表的Schema信息和基本函数信息。通过元数据来判断执行计划内的表/字段是否存在、确定字段的数据类型、sum/avg等会被解析成特定的聚合函数等等。

源码执行过程：
1、QueryExecution中初始化analyzer变量：

```scala
lazy val analyzed: LogicalPlan = {
    SparkSession.setActiveSession(sparkSession)
    sparkSession.sessionState.analyzer.executeAndCheck(logical)
} 
```
2、Analyzer类

executeAndCheck方法：

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
execute方法：

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

executeSameContext方法：

```scala
private def executeSameContext(plan: LogicalPlan): LogicalPlan = super.execute(plan)
```

3、RuleExecutor类：

execute方法：

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

 batches定义在Analyzer中：
 
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
 
 比如ResolveRelations这个rule会判断表是否存在：
 
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
