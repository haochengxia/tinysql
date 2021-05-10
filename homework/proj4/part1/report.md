# Proj 4 Optimization

> Date: 2021/04/17

## Part 1 搜索框架 System R 和 Cascades

### 1. 基础概念

[揭秘 TiDB 新优化器：Cascades Planner 原理解析](https://pingcap.com/blog-cn/tidb-cascades-planner/)笔记：

“*TiDB 当前的优化器将优化过程主要分为逻辑优化（Logical Optimize）和物理优化（Physical Optimize）两个阶段。逻辑优化是将一棵逻辑算子树（LogicalPlan Tree）进行逻辑等价的变化，最后的结果是一棵更优的逻辑算子树；而物理优化则是将一棵逻辑算子树转换成一棵物理算子树（PhysicalPlan Tree）。*”

#### 1.1 逻辑优化

TiDB目前逻辑优化框架存在的问题：
1. 由于要求优化一定要有收益，但是有些优化规则是只对特定场景有收益的，这类优化规则不适合加入；
   **e.g.**
2. 所有优化规则按照了固定了顺序去执行，这导致顺序安排对开发者知识要求较高；
   **e.g.**
3. 每个规则至多只会在被顺序遍历到的时候执行一次。
   **e.g.**

#### 1.2 物理优化

“*在物理优化中，优化器会结合数据的分布（统计信息）情况来对查询计划进行优化，物理优化是一个记忆化搜索的过程，搜索的目标是为逻辑执行计划寻找满足特定物理属性的物理执行计划，并在其中选择代价最低的作为搜索结果，因此也被称为基于代价的优化（Cost-Based Optimization，CBO），例如 DataSource 应该选择怎样的扫描路径（使用哪个索引），Join 应该选择怎样的执行方式（HashJoin、MergeJoin 或 IndexJoin）等。*”

### 2. 分析

`rule_predicate_push_down.go`文件主要实现的即为逻辑优化的谓词下推规则。

根据[TiDB 源码阅读系列文章（七）基于规则的优化](https://pingcap.com/blog-cn/tidb-source-code-reading-7/)，谓词下推操作，

```go
// PredicatePushDown implements LogicalPlan interface.
func (p *baseLogicalPlan) PredicatePushDown(predicates []expression.Expression) ([]expression.Expression, LogicalPlan) {
	if len(p.children) == 0 {
		return predicates, p.self
	}
	child := p.children[0]
	rest, newChild := child.PredicatePushDown(predicates)
	addSelection(p.self, newChild, rest, 0)
	return nil, p.self
}
```

“*`PredicatePushDown` 函数处理当前的查询计划 p，参数 predicates 表示要添加的过滤条件。函数返回值是无法下推的条件，以及生成的新 plan。这个函数会把能推的条件尽量往下推，推不下去的条件，做到一个 Selection 算子里面，然后连接在节点 p 上面，形成新的 plan。比如说现在有条件 a > 3 AND b = 5 AND c < d，其中 a > 3 和 b = 5 都推下去了，那剩下就接一个 c < d 的 Selection。*”

接下来分析该接口函数的具体行为：

```go
type baseLogicalPlan struct {
	basePlan

	taskMap  map[string]task
	self     LogicalPlan
	children []LogicalPlan
}
```

如果当前所有的子逻辑plan数为0，返回过滤条件和当前plan，否则拿出子plan继续进行谓词下推优化，并更新子plan。

需要对不同的算子实现谓词下推优化，而在这个project中需要实现的是对Aggregation 算子的谓词优化：

```go
// PredicatePushDown implements LogicalPlan PredicatePushDown interface.
func (la *LogicalAggregation) PredicatePushDown(predicates []expression.Expression) (ret []expression.Expression, retPlan LogicalPlan) {
	// TODO: Here you need to push the predicates across the aggregation.
	//       A simple example is that `select * from (select count(*) from t group by b) tmp_t where b > 1` is the same with
	//       `select * from (select count(*) from t where b > 1 group by b) tmp_t.
	
	return predicates, la
}
```

简单参照Selection 算子实现：
* Selection 算子：
   拿到条件，作为参数交给函数`splitSetGetVarFunc`，根据有没有取值/赋值操作区分能不能下推，把能下推的操作交给第一个子plan。

```go
// PredicatePushDown implements LogicalPlan PredicatePushDown interface.
func (p *LogicalSelection) PredicatePushDown(predicates []expression.Expression) ([]expression.Expression, LogicalPlan) {
	canBePushDown, canNotBePushDown := splitSetGetVarFunc(p.Conditions)
	retConditions, child := p.children[0].PredicatePushDown(append(canBePushDown, predicates...))
	retConditions = append(retConditions, canNotBePushDown...)
	if len(retConditions) > 0 {
		p.Conditions = expression.PropagateConstant(p.ctx, retConditions)
		// Return table dual when filter is constant false or null.
		dual := Conds2TableDual(p, p.Conditions)
		if dual != nil {
			return nil, dual
		}
		return nil, p
	}
	return nil, child
}
```
### 3. 实现

```go
func (la *LogicalAggregation) PredicatePushDown(predicates []expression.Expression) (ret []expression.Expression, retPlan LogicalPlan) {
	// TODO: Here you need to push the predicates across the aggregation.
	//       A simple example is that `select * from (select count(*) from t group by b) tmp_t where b > 1` is the same with
	//       `select * from (select count(*) from t where b > 1 group by b) tmp_t.
	// In the example, 'b > 1' is the predicate which can be pushed down, so the predicates including 'b > 1'
	var canBePushed []expression.Expression

	exprAgg := make([]expression.Expression, 0, len(la.AggFuncs))
	// Fill with expr in AggFuncs
	for _, func_expr := range la.AggFuncs {
		exprAgg = append(exprAgg, func_expr.Args[0])
	}

	groupByColumns := expression.NewSchema(la.GetGroupByCols()...)

	for _, cond := range predicates {
		switch cond.(type) {
		case *expression.Constant:
			// If condition in predicates is constant, it can be pushed down
			canBePushed = append(canBePushed, cond)
			ret = append(ret, cond)
		case *expression.ScalarFunction:
			// Extract the cols involved in the cond
			extractedCols := expression.ExtractColumns(cond)
			ok := true
			// Check this cond whether can be pushed down
			for _, col := range extractedCols {
				if !groupByColumns.Contains(col) {
					ok = false
					break
				}
			}
			if ok {
				newFunc := expression.ColumnSubstitute(cond, la.Schema(), exprAgg)
				canBePushed = append(canBePushed, newFunc)
			} else {
				ret = append(ret, cond)
			}
		default:
			ret = append(ret, cond)
		}
	}
	la.baseLogicalPlan.PredicatePushDown(canBePushed)
	return ret, la
}
```

### 4. 结果

go test . -check.f testTransformationRuleSuite

go test . -check.f TestPredicatePushDown


## Reference

1. [揭秘 TiDB 新优化器：Cascades Planner 原理解析](https://pingcap.com/blog-cn/tidb-cascades-planner/)
2. [TiDB 源码阅读系列文章（七）基于规则的优化](https://pingcap.com/blog-cn/tidb-source-code-reading-7/)

