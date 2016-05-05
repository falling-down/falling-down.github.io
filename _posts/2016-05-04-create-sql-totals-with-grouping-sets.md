---
layout: post
title: Create Totals in SQL with Grouping Sets
category: SQL
---

Have you ever had a GROUP BY query and wished you could simply have the totals as well to validate against a control? Well guess what, SQL does provide a way to include totals and even sub-totals to your aggregated results. The GROUPING SETS clause allows you to define 1 or more levels to aggregate on and it will insert those summaries into your returned results.

Below is a simple example.

```SQL
select
  territory
  , sum(sales) sales
from fact_sales
group by GROUPING SETS (
  (territory)
  ,()
)
```

and the results look like this

| TERRITORY | SALES |
|----|-----|
| North | 1,000,000 |
| South | 3,000,000 |
| East | 500,000 |
| West | 1,500,000 |
| (null) | 6,000,000 |

That last line with (null) for territory is our grand total calculated by the GROUPING SETS. Let's breakdown the syntax because it can be a little confusing at first glance.

First of all, the GROUPING SETS clause is part of the GROUP BY, so you still start off with `GROUP BY` immediately followed by the `GROUPING SETS`.

GROUPING SETS accepts a variable number of sets delineated by commas with each set defined within a pair of parentheses. The collection of sets together are also contained in a pair of parentheses. 

```SQL
group by            -- start GROUP BY clause
GROUPING SETS       -- start GROUPING SETS clause
(                   -- opening parenthesis for GROUPING SETS
  (territory)       -- first GROUPING SET
  ,()               -- second GROUPING SET
)                   -- closing parenthesis for GROUPING SETS
``` 

The grouping set `()` denotes that it should total across all the results i.e. a grand total.

If you had only specified the `(territory)` set then it would have been equivalent to a standard GROUP BY territory.

Things get a little more interesting when you have additional attributes to group on. Here is another example.

```SQL
select
  territory
  , product_family
  , sum(sales) sales
from fact_sales
group by GROUPING SETS (
  (territory, product_family)
  ,(territory)
  ,()
)
```

The result of this query will include the sales totals by territory and product_family, the grand total, and also the sub-totals for each territory.

| TERRITORY | PRODUCT_FAMILY | SALES |
|----|-----|------|
| North | Produce | 500,000 |
| North | Drinks | 500,000|
| South | Produce | 1,000,000 |
| South | Meat | 1,600,000 |
| South | Candy | 400,000 |
| East | Produce | 200,000 |
| East | Candy | 300,000 |
| West | Meat | 1,000,000 |
| West | Drinks | 500,000 |
| North | (null) | 1,000,000 |
| South | (null) |  3,000,000 |
| East | (null) |  500,000 |
| West | (null) |  1,500,000 |
| (null) | (null) |  6,000,000 |

You can use any mix of grouping sets that you need as long as each non-aggregated column is included in at least one grouping set. For example, omitting the `(territory, product_family)` grouping set from the above query would have resulted in an error because product_family is in the SELECT portion but not in the GROUP BY (which is what happens even when not using GROUPING SETS). 

If you do not want the totals to show as NULL, then you can use COALESCE to give a description.

```SQL
select
  coalesce(territory, 'Total') territory
  , sum(sales) sales
from fact_sales
group by GROUPING SETS (
  (territory)
  ,()
)
```

GROUPING SETS is useful. Unfortunately not every database supports the syntax. Most major databases do (Oracle, MS SQL Server, PostgresQL, etc.) Most notably, Amazon's Redshift does not support GROUPING SETS. However, I imagine that will change as the product matures.

The ROLLUP and CUBE clauses provide similar functionality and may be supported by your database of choice.
