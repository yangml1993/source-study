### SparkSQL优化器之Spark-Catalyst组件
Spark-Catalyst是SparkSQL的核心组件。它负责将SQL语句转换成物理执行计划。

SQL->AST->UnresolvedLogicPlan->LoginPlan->PhysicalPlan->Rdd

QueryExecution

1) SqlParser将SQL语句解析成一个Unresolved LogicPlan
ParserDriver的parse方法，通过antlr4生成语法树


2) Analyzer对Unresolved LogicPlan做分析得到Analyzd LogicPlan
Analyzer利用HiveMeta中表/列等信息，对逻辑执行计划进行解析(如表/列是否存在等)
sparkSession.sessionState.analyzer.executeAndCheck(logical)


3)Optimizer对Unresolved LogicPlan优化得到optimized LogicPlan
 sparkSession.sessionState.optimizer.execute(withCachedData)

4)SparkPlanner将逻辑执行计划转换成物理执行计划





