---
background: "#white"
theme: seriph
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## TiDB Decorrelate

  Decorrelate is a SQL optimization technique that rewrites correlated subqueries into joins, improving query performance.

drawings:
  persist: false
defaults:
  foo: true
transition: slide-left
title: TiDB Analyze
colorSchema: light
mdc: true
---

# Decorrelate

---

## ExtractCorColumns
1. Extract the correlate columns https://github.com/pingcap/tidb/blob/24123512f14876973475667d99ab26b4c83d0104/pkg/planner/util/coreusage/correlated_misc.go#L46
    1. The interface: https://github.com/pingcap/tidb/blob/24123512f14876973475667d99ab26b4c83d0104/pkg/planner/core/base/plan_base.go#L268
    2. Each operator has its own implementation, but primarily, it checks each expression from the operator.
    3. The ExtractCorColumns function

```go
func ExtractCorColumns(expr Expression) (cols []*CorrelatedColumn) {
	switch v := expr.(type) {
	case *CorrelatedColumn:
		return []*CorrelatedColumn{v}
	case *ScalarFunction:
		for _, arg := range v.GetArgs() {
			cols = append(cols, ExtractCorColumns(arg)...)
		}
	}
	return
}
```

---

## With a Selection

```sql
-- Create orders table
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    total DECIMAL(10,2)
);

-- Insert sample data
INSERT INTO orders (order_id, customer_id, total) VALUES
(1, 100, 150.00),
(2, 100, 250.00),
(3, 100, 100.00),
(4, 101, 300.00),
(5, 101, 450.00),
(6, 102, 75.00);

-- Original correlated query
SELECT * FROM orders o
WHERE o.total > (
    SELECT AVG(total)
    FROM orders o2
    WHERE o2.customer_id = o.customer_id
);
```

---

## With a Selection - With decorrelate

```sql
+-----------------------------+-------+---------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|id                           |estRows|task     |access object|operator info                                                                                                                                                                                                                                            |
+-----------------------------+-------+---------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|Projection_10                |3.84   |root     |             |test.orders.order_id, test.orders.customer_id, test.orders.total                                                                                                                                                                                         |
|└─Selection_11               |3.84   |root     |             |gt(test.orders.total, Column#7)                                                                                                                                                                                                                          |
|  └─HashAgg_12               |4.80   |root     |             |group by:test.orders.order_id, funcs:firstrow(test.orders.order_id)->test.orders.order_id, funcs:firstrow(test.orders.customer_id)->test.orders.customer_id, funcs:firstrow(test.orders.total)->test.orders.total, funcs:avg(test.orders.total)->Column#7|
|    └─HashJoin_14            |7.49   |root     |             |left outer join, left side:TableReader_17, equal:[eq(test.orders.customer_id, test.orders.customer_id)]                                                                                                                                                  |
|      ├─TableReader_20(Build)|5.99   |root     |             |data:Selection_19                                                                                                                                                                                                                                        |
|      │ └─Selection_19       |5.99   |cop[tikv]|             |not(isnull(test.orders.customer_id))                                                                                                                                                                                                                     |
|      │   └─TableFullScan_18 |6.00   |cop[tikv]|table:o2     |keep order:false, stats:pseudo                                                                                                                                                                                                                           |
|      └─TableReader_17(Probe)|6.00   |root     |             |data:TableFullScan_16                                                                                                                                                                                                                                    |
|        └─TableFullScan_16   |6.00   |cop[tikv]|table:o      |keep order:false, stats:pseudo                                                                                                                                                                                                                           |
+-----------------------------+-------+---------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

---

## With a Selection - With decorrelate

```sql
+-----------------------------+-------+---------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|id                           |estRows|task     |access object|operator info                                                                                                                                                                                                                                            |
+-----------------------------+-------+---------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|Projection_10                |3.84   |root     |             |test.orders.order_id, test.orders.customer_id, test.orders.total                                                                                                                                                                                         |
|└─Selection_11               |3.84   |root     |             |gt(test.orders.total, Column#7)                                                                                                                                                                                                                          |
|  └─StreamAgg_13             |4.80   |root     |             |group by:test.orders.order_id, funcs:firstrow(test.orders.order_id)->test.orders.order_id, funcs:firstrow(test.orders.customer_id)->test.orders.customer_id, funcs:firstrow(test.orders.total)->test.orders.total, funcs:avg(test.orders.total)->Column#7|
|    └─Apply_22               |6.00   |root     |             |CARTESIAN left outer join, left side:TableReader_24                                                                                                                                                                                                      |
|      ├─TableReader_24(Build)|6.00   |root     |             |data:TableFullScan_23                                                                                                                                                                                                                                    |
|      │ └─TableFullScan_23   |6.00   |cop[tikv]|table:o      |keep order:true, stats:pseudo                                                                                                                                                                                                                            |
|      └─TableReader_20(Probe)|0.04   |root     |             |data:Selection_19                                                                                                                                                                                                                                        |
|        └─Selection_19       |0.04   |cop[tikv]|             |eq(test.orders.customer_id, test.orders.customer_id)                                                                                                                                                                                                     |
|          └─TableFullScan_18 |36.00  |cop[tikv]|table:o2     |keep order:false, stats:pseudo                                                                                                                                                                                                                           |
+-----------------------------+-------+---------+-------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```


---

## With a MaxOneRow

With a `MaxOneRow`, https://github.com/pingcap/tidb/blob/37a2bf4e891e2bee294d6503acc793b4803cc727/pkg/planner/core/rule_decorrelate.go#L252

For ScalarSubquery, it always produce only one row: https://github.com/pingcap/tidb/blob/37a2bf4e891e2bee294d6503acc793b4803cc727/pkg/planner/core/expression_rewriter.go#L1284

```sql
-- Create orders table
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    total DECIMAL(10,2)
);

-- Insert sample data
INSERT INTO orders (order_id, customer_id, total) VALUES
(1, 100, 150.00),
(2, 100, 250.00),
(3, 100, 100.00),
(4, 101, 300.00),
(5, 101, 450.00),
(6, 102, 75.00);

-- Original correlated query
SELECT o1.order_id,
       (SELECT MAX(o2.total)
        FROM orders o2
        WHERE o2.customer_id = o1.customer_id
        HAVING COUNT(*) = 1) as single_order_total
FROM orders o1;
```

---

## With a MaxOneRow - With decorrelate

```sql
+----------------------------+--------+---------+-------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|id                          |estRows |task     |access object|operator info                                                                                                                                                         |
+----------------------------+--------+---------+-------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|HashJoin_15                 |10000.00|root     |             |left outer join, left side:TableReader_18, equal:[eq(test.orders.customer_id, test.orders.customer_id)]                                                               |
|├─Selection_19(Build)       |6393.60 |root     |             |eq(Column#11, 1)                                                                                                                                                      |
|│ └─HashAgg_25              |7992.00 |root     |             |group by:test.orders.customer_id, funcs:max(Column#13)->Column#10, funcs:count(Column#14)->Column#11, funcs:firstrow(test.orders.customer_id)->test.orders.customer_id|
|│   └─TableReader_26        |7992.00 |root     |             |data:HashAgg_20                                                                                                                                                       |
|│     └─HashAgg_20          |7992.00 |cop[tikv]|             |group by:test.orders.customer_id, funcs:max(test.orders.total)->Column#13, funcs:count(1)->Column#14                                                                  |
|│       └─Selection_24      |9990.00 |cop[tikv]|             |not(isnull(test.orders.customer_id))                                                                                                                                  |
|│         └─TableFullScan_23|10000.00|cop[tikv]|table:o2     |keep order:false, stats:pseudo                                                                                                                                        |
|└─TableReader_18(Probe)     |10000.00|root     |             |data:TableFullScan_17                                                                                                                                                 |
|  └─TableFullScan_17        |10000.00|cop[tikv]|table:o1     |keep order:false, stats:pseudo                                                                                                                                        |
+----------------------------+--------+---------+-------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

---

## With a MaxOneRow - With decorrelate

```sql
+--------------------------------+------------+---------+-------------+------------------------------------------------------------------+
|id                              |estRows     |task     |access object|operator info                                                     |
+--------------------------------+------------+---------+-------------+------------------------------------------------------------------+
|Projection_12                   |10000.00    |root     |             |test.orders.order_id, Column#10                                   |
|└─Apply_14                      |10000.00    |root     |             |CARTESIAN left outer join, left side:TableReader_16               |
|  ├─TableReader_16(Build)       |10000.00    |root     |             |data:TableFullScan_15                                             |
|  │ └─TableFullScan_15          |10000.00    |cop[tikv]|table:o1     |keep order:false, stats:pseudo                                    |
|  └─MaxOneRow_17(Probe)         |10000.00    |root     |             |                                                                  |
|    └─Selection_18              |8000.00     |root     |             |eq(Column#11, 1)                                                  |
|      └─StreamAgg_33            |10000.00    |root     |             |funcs:max(Column#15)->Column#10, funcs:count(Column#16)->Column#11|
|        └─TableReader_34        |10000.00    |root     |             |data:StreamAgg_22                                                 |
|          └─StreamAgg_22        |10000.00    |cop[tikv]|             |funcs:max(test.orders.total)->Column#15, funcs:count(1)->Column#16|
|            └─Selection_32      |100000.00   |cop[tikv]|             |eq(test.orders.customer_id, test.orders.customer_id)              |
|              └─TableFullScan_31|100000000.00|cop[tikv]|table:o2     |keep order:false, stats:pseudo                                    |
+--------------------------------+------------+---------+-------------+------------------------------------------------------------------+
```
