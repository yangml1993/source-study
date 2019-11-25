### SparkSQL优化器之Spark-Catalyst组件
Spark-Catalyst是SparkSQL的核心组件。它负责将SQL语句转换成物理执行计划。

SQL->AST->UnresolvedLogicPlan->LoginPlan->PhysicalPlan->Rdd

QueryExecution

1) SqlParser将SQL语句解析成一个逻辑执行计划(未解析)
antlr4

Parser

2) Analyzer利用HiveMeta中表/列等信息，对逻辑执行计划进行解析(如表/列是否存在等)
Analyzer
sparkSession.sessionState.analyzer.executeAndCheck(logical)

3)
Optimizer

PLanner

catalog