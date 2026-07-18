# Queries
A query is an expression that generates a collection. Once a source is defined, it can be referenced in a query. Queries are also used to source the information needed in insert, update, and delete operations.

## From
A query often begins by specifying where the data comes from, using the `from` keyword. The simplest query immediately returns the values from the source using the `select` operation:
```
let result =
    from customers as c
    select c;
```

> **NOTE:** This query is for demonstration purposes only. It is equivalent to `let result = customers;`

The `as` keyword is used to provide an alias for items in the collection. An alias *is required*.

> By convention, aliases tend to be small, but they can be more descriptive, like `customer` in the previous example. The reason for shorter aliases is to reduce the horizontal space needed to define queries, similar to SQL. Choose a naming convention that works best for you.

After the `from` operation, items from the collection are referred to using the alias. That's why `c` appears in the `select` operation.

Multiple `from` operations can appear in the same query. Imagine a `customer` object had a list of its orders in an `orders` property:
```
from customers as c
from c.orders as o
select { customer: c, order: o }
```

This query will return every customer/order combination, wrapped in objects with a `customer` and `order` property.

## Select
The `select` operation defines which values are returned (or "projected") by a query.

An entire entity can be selected:
```
from customers as c
select c;
```

Or a copy of an entity can be selected:
```
from customers as c
select { ...c };
```

If properties are copied from multiple entities, properties with overlapping names will come from the latter entity. Assume for the next example that both customers and orders have an `id` property:
```
from customers as c
from c.orders as o
select { ...c, ...o }; # id will come from o, not c
```

The property names of selected entities can be customized by putting the name and colon before the value:
```
from customers as c
select { customer: c };
```

Multiple entities can be combined:
```
from customers as c
from c.orders as o
select { customer: c, order: o };
```

or just their properties:
```
from customers as c
from c.orders as o
select { customerId: c.id, orderId: o.id };
```

Nested entities can be selected, as well:
```
from customers as c
select {
    customer: c,
    orders: c.orders,
    extra: {
        hello: "world"
    }
};
```

Whenever you see properties wrapped in curly braces (i.e., `{}`), an anonymous type is being created. In certain circumstances, when the type can be inferred, an existing type will be used instead. To explicitly control which type is used, the name of the type can be specified prior to the curly braces:
```
from customers as c
select Customer { ...c };
```

A `select` does not need to be the last operation in a query, in which case the projected value must be given an alias:
```
from customers as c
select { customerName: c.name } as name
from c.orders as o
select { customerName: name, total: o.totalAmount };
```

### Arithmetic expressions and other computations
Within a query, expressions can include arithmetic operations or method calls, such as:
```
from customers as c
select { 
    score: (100 * c.orders.count()) as f64 + (c.creditRating * 0.05) 
};
```

### Let
Almost anywhere within a query, expressions can be broken out into smaller chunks by giving them an alias:
```
from customers as c
let orderScore = 100.0 * c.orders.count() as f64
let creditScore = c.creditRating * 0.05
let totalScore = orderScore + creditScore
select { score: totalScore };
``` 

This can help to keep a line from getting too long, allow for reuse, and make complex operations easier to understand.

> **NOTE:** A `let` can even appear prior to the first `from` clause, allowing reusable queries and values to be defined.

## Where
In order to filter records, a `where` operation can be performed. After the `where` keyword, a boolean expression must be provided. Only records satisfying the `where` expression will be included in the results.
```
let loyalCustomers =
    from customers as c
    where c.isLoyal
    select c;
```

Multiple `where` operations can appear in the same query. For example, two `where` operations are performed in the next example to filter the customers and then their orders:
```
let loyalCustomersWithOpenOrders =
    from customers as c
    where c.isLoyal
    let openOrders = 
        from c.orders as o
        where o.status != 'closed'
        select o
    where any(openOrders)
    select { customer: c };
```

Multiple boolean expressions can be combined using `and`, `or`, and `not`, with parentheses to control precedence. Furthermore, a query can filter the same record multiple times using multiple `where` operations. The following two queries are equivalent:
```
from customers as c
where c.isLoyal and c.isCardHolder
select c;
```
and:
```
from customers as c
where c.isLoyal
where c.isCardHolder
select c;
```

> **NOTE:** The same cannot be said for `or`.

### Null safety
QL does not allow inspecting members when a value can be `null`. For example, if a customer can be null, inspecting its orders cannot be performed until after checking for null. Consider:
```
from customers as c?
where c != null
from c.orders as o
select o;
```

First, note that the alias `c` is followed by a `?`. This indicates to QL that you are aware `c` can be `null` - it's considered an error if the `?` is missing. We then check for `null` in the `where`. Since the `where` filters out `null` customers, all subsequent operations in the query are okay assuming `c` cannot be `null`.

Another option is to substitute `null` with an alternative value, as such:
```
from customers as c?
from (c?.orders ?? []) as o
select o;
```

The `?.` operator causes any operations on a `null` to be replaced with a `null`, and the `??` operator says to replace `null` with the value on the right. Essentially, if `c` is `null`, the whole expression evaluates to an empty collection.

## Distinct
Unique values can be retrieved using a `distinct` operation. For example:
```
from customers as c
distinct c;
```

> **NOTE:** `distinct` can be the last operation in a query, no `select` required!

In the example above, what if we wanted to base uniqueness by first name only? In that case, the `on` keyword can be used:
```
from customers as c
distinct c on c.firstName;
```

> **NOTE:** Multiple properties can be passed to `on` using a tuple, such as `c.firstName, c.lastName`. However, it can also be passed an anonymous type, such as `{ firstName: c.firstName, lastName: c.lastName }`. Anonymous types used in the context of `on` automatically implement equality based on the fields provided. However, specifying a tuple works just as well in this situation and requires less typing and overhead.

> **NOTE:** It can be helpful to think of `distinct` as a special form of `where`. After the `where` all subsequent operations will only see the unique values.

What if we wanted to just return the unique first names? We would first need to map from a customer to their first names, then use `distinct`:
```
from customer as c
select c.firstName as fn
distinct fn;
```

## Join
Not all sources will store related information in nested properties, like orders under customers (e.g., `c.orders`) in our previous examples. Some sources, like a relational database, will store customers and orders separately, using IDs to link related records.

For the next example, assume we have a `customer` source and an `order` source, but this time `order` has a `customerId` property for linking back to a customer. In that case, we can use a `join` operation to match customers and orders together using the `customerId` property:
```
from customers as c
join orders as o on c.id == o.customerId
select { customer: c, order: o };
```

A `join` operation will return every customer/order combination; it does it by matching the condition following the `on` clause. The condition can be much more complicated to allow for more complicated join scenarios.

> **`from` vs `join`?** If your data is already structured in such a way that related objects are in nested properties, `from` is a logical choice. Otherwise, a `join` may make more sense. A `from` with a filter (a.k.a, `where`) can have the same effect as a `join`. However, for a particular source implementation, using an explicit `join` can lead to more optimized queries.

### How joins work
It can be helpful to have a mental model for how joins work. One way is to think of them operating like so:
```
from customers as c                # all customers
from orders as o                   # all orders
where c.id == o.customerId
select { customer: c, order: o };
```

Here we are basically iterating over *every* order for *every* customer, matching them by ID. In reality, joins can perform much more efficiently, using lookups and other mechanisms. It's worth learning join syntax in order to benefit from these performance improvements, but also to communicate more explicitly the relationship between entities.

Also note that each time a `join` is performed, a new alias is created. In our examples, a new alias for `o` is created. After that point, both `c` and `o` can be used in subsequent operations. It's common for there to be many joins in a query, with just as many aliases.

## Left join
What if we wanted to return all customers, regardless of whether they had any orders? In other words, if a customer had orders, we would still get back every customer/order combination; however, for customers missing orders, we still want to get back one record, where the order is just `null`.

The `left join` operation does just that:
```
from customers as c
left join orders as o on c.id == o.customerId
select { customer: c, order: o };
```

An interesting aspect of left joins is that the joined collection item (`o` in this case) is implicitly nullable afterward. So, if `orders` is `Order[]`, then `o` will be treated as `Order?`. Any operations on `o` for the rest of the query (after the `on` clause) must check for `null`. Within the `on` clause, though, `o` can be treated `Order`.

**NOTE:** The only caveat is if `orders` is `Order?[]` to begin with, in which case the alias must be declared `o?` and `null` handled within the `on` clause. This should rarely be the case, as `null` records are uncommon in practice.

> **NOTE:** A right join can be simulated by reversing the order collections are joined.

## Full join
The last type of join is a `full join`, which will join two related objects if they satisfy the condition, but will return every object from either collection even if no match is found. To demonstrate, we need two collections that can share information but don't form a parent/child relationship. Let's imagine a Role can be assigned to a User or a Group of Users, and we want to find all Roles that are assigned to at least one. Here's how we could do that.
```
let userRoles =
    from userRoles as ur
    join roles as r on ur.roleId == r.id
    select r;
let groupRoles =
    from groupRoles as gr
    join roles as r on gr.roleId == r.id
    select r;
let assignedRoles =
    from userRoles as ur
    full join groupRoles as gr on ur.id == gr.id
    let r = ur ?? gr
    where r != null
    select r;
```

After the `on` clause, both `ur` or `gr` can potentially be `null`, but not both - try to understand why. The compiler doesn't know that only one can be `null`, though. In the example, we use the `??` operator to grab whichever value is non-null first and check for `null` using `where r != null`, but we also could have used the `!` suffix operator to express our intent, like so:
```
# ... same as before
let assignedRoles =
    from userRoles as ur
    full join groupRoles as gr on ur.id == gr.id
    select (ur ?? gr)!;
```

This example is just for demonstration purposes. A much easier solution is as follows:
```
from roles as r
let usedUserRoles =
    from userRoles as ur
    where ur.roleId == r.id
    select ur
let usedGroupRoles =
    from groupRoles as gr
    where gr.roleId == r.id
    select gr
where any(usedUserRoles) or any(usedGroupRoles)
select r;
```

## Grouping
Groups can be formed using the `group by` keywords. When grouping, you must identify what entities are being grouped and what the key is (unique value to group them by). For example:
```
let nameGroups = 
    from customers as c
    group c by c.firstName;
```

Here we are grouping all customers by their `firstName` value. If we had customers like:

* Sue Smith
* Tom Harris
* Sue Williams
* Bob Thomas

We'd end up with a group for "Sue", "Tom", and "Bob" with two customers under "Sue":

* Sue - [Sue Smith, Sue Williams]
* Tom - [Tom Harris]
* Bob - [Bob Thomas]

The result of a `group by` operation is a list of groups. Each group has a `key` property. Whatever expression appears after the `by` is used to populate the `key` value for each group.

In the example above, we are grouping the entire customer (`c`) by its `firstName` attribute. The group for "Sue" will have two items (customers) and the `key` will equal `"Sue"`.

Things can get more interesting when multiple entities are joined together. For example:
```
let ordersByCustomer = 
    from customer as c
    join order as o on c.id == o.customerId
    group o by c;
```

In this example, we are grouping the orders by their customers. Since we put `o` after the `group` keyword, the values in the groups will be orders, and the `key` for each group will be the customer.

If we want to group by customer, but only using the customer's first name, we use `on`:
```
let ordersByCustomer =
    from customer as c
    join order as o on c.id == o.customerId
    group o by c on c.firstName;
```

## Aggregation
In the previous examples, after a `group by`, we are no longer working with a one-dimensional collection. Instead, we have a list of groups, which are themselves a type of list. Often, after grouping data, we want to summarize each group, going back to a one-dimensional collection. To do this, we can use `aggregate`.

For example, here's how we can count the orders for each customer:
```
let customerOrderCounts =
    from customer as c
    join order as o on c.id = o.customerId
    group o by c as g
    aggregate {
        customer: g.key,
        orderCount: count(g)
    };
```

First, notice we provide an alias for the group, `g`. We use this in the `aggregate` operation below.

Here, the `aggregate` operation takes the list of customer groups produced from a `group by` and combines each into a result, one for each group.

As you can see, if we want to access the customer information (what we grouped by), we use the `key` property on the group.

An `aggregate` operation can be used even without a `group by`. Consider:
```
from customers as c
aggregate count(c);
```

This query returns the number of customers. The result is a single integer value.

Here is a list of common aggregate functions:

* `count` - count the number of items in a collection
* `any` - true if the collection is non-empty
* `none` - true if the collection is empty
* `first` - the first value in a collection
* `last` - the last value in a collection
* `max` - the largest value in a collection (comparable)
* `min` - the smallest value in a collection (comparable)
* `sum` - the sum of the values in the collection (numeric)
* `average` - the average of the values in the collection (numeric)
* `product` - the product of the values in the collection (numeric)
* `joined` - the values in the collection joined together into a string, separated by a separator

> **NOTE:** One curiosity of `aggregate` is that you can apply collection operations on an alias. In the other operations we explored previously, an alias refers to a single value in a collection. In `aggregate`, the same alias now refers to an entire collection! Any filtering applied to the collection up to that point is accounted for when aggregating the values.

## Ordering
Values can be sorted using the `order by` operation.
```
from [3, 1, 2] as v
order by v
select v; # [1, 2, 3]
```

> **NOTE:** An `order by` operation cannot be the last operation in a query!

Sorting can refer to specific fields, as well:
```
from customers as c
order by c.firstName
select c;
```

## Skip
The `skip` operation skips either a specific number of values in a query or while a condition persists:
```
from 1..10 as v:
skip 3
select v; # [4, 5, 6, 7, 8, 9]
```
or
```
from 1..10 as v:
skip v < 4
select v; # [4, 5, 6, 7, 8, 9]
```

## Take
The `take` operation takes a specific number of values in a query or while a condition persists:
```
from 1..10 as v:
take 3
select v; # [1, 2, 3]
```
or
```
from 1..10 as v:
take v < 4
select v; # [1, 2, 3]
```

## Windows
A `window` computes a value *for each row* using a set of related rows around it - called a *frame* - without collapsing rows the way a `group by` does. Where a `group by` turns many rows into one row per group, a window keeps every row and simply attaches a computed value to it.

Because of this, a window fits naturally into the same slot as a `let`: it *adds* a value to each row while leaving every prior alias in scope. Contrast this with `group by`, which replaces the scope entirely (see the [note below](#windows-and-scope)).

A window is built from up to three parts:

* `partition by` - splits rows into independent partitions. This is like `group by`, except the rows are *not* collapsed; the window simply restarts at each partition boundary.
* `order by` - orders the rows within each partition. This is required for anything running, ranking, or offset-based.
* a *frame* - the subset of the partition that a given row can see, written as a slice relative to the current row.

The frame reuses the same slice syntax used elsewhere (see [Splicing](./in-memory-sources.md#splicing)). The current row is offset `0`, preceding rows are negative, and following rows are positive. Open-ended ranges reach to the start or end of the partition:

* `[..=0]` - from the start of the partition through and including the current row (a *running* frame)
* `[-2..=0]` - the current row and the two preceding it (a trailing frame of three)
* `[0..]` - the current row through the end of the partition
* `[..]` - the entire partition

Here we compute a running total and a three-row moving average of order amounts for each customer:
```
from orders as o
partition by o.customerId
order by o.date
let runningTotal = sum(o.amount) over [..=0]
let movingAvg = average(o.amount) over [-2..=0]
select { ...o, runningTotal, movingAvg };
```

Because the current row is `0`, and `..=` includes its endpoint, `[..=0]` covers every row from the start of the partition through the current one - a running total. Note that all of `o`'s properties remain available; the window only *adds* `runningTotal` and `movingAvg`.

A frame covering the whole partition (`[..]`) reduces the partition to a single value and shares it back across every row. This is handy for computing each row's share of a total:
```
from orders as o
partition by o.customerId
let share = o.amount / (sum(o.amount) over [..])
select { ...o, share };
```

> **NOTE:** A whole-partition frame does not require an `order by`, since the frame is the same regardless of order. A running or trailing frame does require one, so the "preceding" and "following" rows are well-defined.

Ranking is expressed as a running count. Since `count()` over a running frame increases by one for each row, it produces a row number within the partition:
```
from orders as o
partition by o.customerId
order by o.date
let rowNumber = count() over [..=0]
select { ...o, rowNumber };
```

### Offsets
A frame that selects a *single* row returns that row's value directly, rather than a collection. This is how you look at a neighboring row (what SQL calls `lag` and `lead`):
```
from orders as o
partition by o.customerId
order by o.date
let previousAmount = o.amount over [-1]  # the prior order's amount, or null at the partition's first row
select { ...o, previousAmount };
```

At the edges of a partition, a neighboring row may not exist. In that case the value is `null`, and the usual [null-safety rules](#null-safety) apply - so `previousAmount` above is inferred as nullable, and must be checked or defaulted (e.g., with `??`) before use.

### Row frames vs. value frames
The frames shown so far count *rows*: the offset `-2` means "two rows back," so `[-2..=0]` is always exactly three rows. Sometimes you instead want a frame defined by the *ordering value* - for example, "every order within the previous week," which may match any number of rows (including none). These two interpretations are genuinely different, and conflating them is a common source of confusion in SQL (its `ROWS` vs. `RANGE`).

QL distinguishes the two by what the frame is written against, with no extra keyword:

* A frame of bare offsets counts **rows**. `[-2..=0]` is the current row and the two rows before it.
* A frame written in terms of the ordering expression counts by **value**. Assuming orders are ordered by an integer `day`, the following frame covers every order in the current order's week, however many that is:

```
from orders as o
partition by o.customerId
order by o.day
let weeklyTotal = sum(o.amount) over [o.day - 6 ..= o.day]
select { ...o, weeklyTotal };
```

The rule for reading a value frame: the expressions *inside* the frame brackets are evaluated for the current row - they define the frame - while the aggregate's argument (`o.amount` above) ranges over the rows the frame selects. So `[o.day - 6 ..= o.day]` means "every order in this partition whose `day` falls between this order's `day` minus six and this order's `day`."

Because a value frame selects by magnitude, all rows sharing an ordering value (peers) naturally fall in the frame together - which is usually the reason to prefer a value frame over a row frame in the first place.

Row and value frames line up shape-for-shape, differing only in axis:

* `[..=0]` (row) and `[..= o.day]` (value) - a running frame, from the start of the partition through the current row.
* `[-2..=0]` (row) and `[o.day - 2 ..= o.day]` (value) - a trailing frame.
* `[..]` - the whole partition, which needs no anchor and so reads the same either way.

> **NOTE:** Single-offset lookups like `over [-1]` are inherently a *row* operation - they name one neighboring row - so they have no value-frame form. A value "offset" could match zero or many rows, which is what the range forms above express instead.

See [Ranges](./ranges.md#position-vs-value) for the general position-vs-value distinction that underlies this.

### Windows and scope
A `window` is unusual among query operations. Most operations either leave the set of available aliases untouched (like `where`), extend it (like `let` and `join`), or replace it (like `select`, `group by`, and `aggregate`). A window *extends* the scope - it adds a value while keeping every existing alias - even though, internally, it performs an aggregation over the frame.

This is what makes a window feel different from a `group by` despite doing similar work under the hood. A `group by` replaces the current row with a group, so prior aliases are only reachable through the group's `key` (or by iterating the group). A window keeps the row, so nothing goes out of scope; the frame it aggregates over is transient and never becomes an alias of its own. In short, a window is like a fused `group by`/`aggregate` that preserves rows instead of collapsing them.
