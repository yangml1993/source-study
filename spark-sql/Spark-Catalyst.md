### SparkSQL优化器之Spark-Catalyst
Spark-Catalyst是SparkSQL的核心组件。它负责将SQL语句转换成物理执行计划。
SQL->AST->UnresolvedLogicPlan->LoginPlan->PhysicalPlan->Rdd

antlr4

Parser

Analyzer

Optimizer

PLanner