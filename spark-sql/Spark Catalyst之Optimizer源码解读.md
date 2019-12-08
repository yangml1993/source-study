###Spark Catalyst之Optimizer源码解读

Optimizer负责对LogicPlan进行优化。QueryExecution通过它将Analyzd LogicPlan转为Optimized LogicPlan：

```scala
lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)

```
