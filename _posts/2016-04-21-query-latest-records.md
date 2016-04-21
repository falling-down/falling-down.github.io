---
layout: post
title: Query the latest records in a data set
---

You have some data that tracks a sequence of events for a group of entities.
For example, a sequence of transactions for a set of bank accounts
or a list of orders for a group of customers.

Your manager is interested to see all the details on the latest order from all your customers.

Your orders data likely has a lot of info, but most importantly it will have
some reference to the customer, some unique way to identify individual orders,
and some way to determine the chronological order of the orders. For simplicity,
assume the order id is an incremental sequence (1, 2, 3, etc).

| ORDER_ID | CUSTOMER | DATE | AMOUNT | DESCRIPTION |
|----------
| 1 | BOB | 2015-05-02 | 105 | Learn SQL |
| 2 | CHRIS | 2015-05-02 | 55 | Video Editing |
| 3 | JILL | 2015-05-04 | 33 | Cooking |
| 4 | BOB | 2015-06-04 | 250 | Learn Java |
| 5 | BOB | 2015-06-04 | 89 | Learn CSS |
| 6 | JILL | 2015-06-22 | 50 | Start a home business |
| 7 | KIM | 2015-07-04 | 25 | Learn Psychology |
| 8 | CHRIS | 2015-07-06 | 45 | Travel Italy |
| 9 | JILL | 2015-07-12 | 65 | Build mobile apps |
| 10 | KIM | 2015-07-23 | 70 | Best Surfing Locations |

I typically see analysts new to SQL initially attempt to answer this question with a query like

```
select
  max(order_id)
, customer
, date
, amount
, description
from orders
group by
  customer
, date
, amount
, description
```

Fortunately, even new SQL users are quick to realize that this is not giving them
the answer they want. The MAX aggregate function simply shows the greatest value within the set of values
being grouped on.

So let's change this to at least tell us the latest order id for each customer.

```
select
  customer
, max(order_id)
from orders
group by
  customer
```

Our small sample set would then result in

| CUSTOMER | MAX(ORDER_ID) |
|----
| BOB | 5 |
| CHRIS | 8 |
| JILL | 9 |
| KIM | 10 |

A good first step, but our manager wants to know the details on these
latest orders too.

One solution that is a natural next step from the query we just wrote is to use
this query as way to limit another query back against
the orders table. There is a couple ways to do this. The most common form is
similar to this.

```
select
  order_id
, customer
, date
, amount
, description
from orders
inner join (
  select
    customer
  , max(order_id) max_order_id
  from orders
  group by
    customer  
) m
  on orders.order_id = m.max_order_id
```

This works, but it is not my preferred approach.

My preferred solution does not use the aggregate MAX function but instead
utilizes the analytic RANK function.

```
select
  order_id
, customer
, date
, amount
, description
, rank() over (partition by customer order by order_id desc) rank_order
from orders
```

The result will look like

| ORDER_ID | CUSTOMER | DATE | AMOUNT | DESCRIPTION | RANK_ORDER |
|----------
| 1 | BOB | 2015-05-02 | 105 | Learn SQL | 3 |
| 2 | CHRIS | 2015-05-02 | 55 | Video Editing | 2 |
| 3 | JILL | 2015-05-04 | 33 | Cooking | 3 |
| 4 | BOB | 2015-06-04 | 250 | Learn Java | 2 |
| 5 | BOB | 2015-06-04 | 89 | Learn CSS | 1 |
| 6 | JILL | 2015-06-22 | 50 | Start a home business | 2 |
| 7 | KIM | 2015-07-04 | 25 | Learn Psychology | 2 |
| 8 | CHRIS | 2015-07-06 | 45 | Travel Italy | 1 |
| 9 | JILL | 2015-07-12 | 65 | Build mobile apps | 1 |
| 10 | KIM | 2015-07-23 | 70 | Best Surfing Locations | 1 |

Sweet! Instead of using that ugly subquery I simply added a single line to a
basic SELECT against the orders table. We are not at the desired result yet,
but let's break down what is happening here.

The `over` clause identifies
this form of RANK as an analytic function. Next, the `partition by customer` piece instructs
RANK to rank order each subset of rows that have an identical customer value. If we had
omitted the `partition by` entirely, then it would treat the entire table as one subset. Finally, the `order by` is required for RANK and instructs
in what way the rows should be ranked. In our case, `order by order_id desc`,
the order_id value in descending order.

The result is now each of the rows we are interested in has a RANK_ORDER = 1.
So we just need to modify the query slightly to keep those and remove the others.

Unfortunately, you cannot simply add a WHERE clause on a RANK() function.
Analytic functions are not permitted in a WHERE clause. So we have to wrap our
query as a subquery.

```
select * from (
select
  order_id
, customer
, date
, amount
, description
, rank() over (partition by customer order by order_id desc) rank_order
from orders
) o
where
  o.rank_order = 1
```

So now we have are desired result. Apart from the little extra cruft needed to
filter the rows, this last query is more readable. It is also quick to change
if you need to answer a variety of similar questions.

Customers' first orders

```
rank() over (partition by customer order by order_id) rank_order
```

Customers' largest orders

```
rank() over (partition by customer order by amount desc) rank_order
```

Largest order by day

```
rank() over (partition by date order by amount desc) rank_order
```
