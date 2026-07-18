# Queries
A query is an expression that generates a collection. Once a source is defined, it can be referenced in a query. Queries are also used to source the information needed in insert, update, and delete operations.

## Terminating a query
Every query must end with a clause that *names its result*. There are two ways to do that:

* **Project** the result. `select`, `distinct`, `group by`, and `aggregate` each name the value(s) the query produces, so any of them can be the final clause.
* **Mutate** a target. `into`, `update`, `delete`, and `merge` consume the query to change data, so they, too, can be the final clause. (See [Modifying data](./modifying-data.md).)

Every other clause is a *modifier*. `from`, `join`, `where`, `let`, `skip`, `take`, `order by`, and `partition by` shape the pipeline, but each takes a predicate, key, or count rather than naming an output. A modifier can never be the last clause in a query.

This is why `select c` and `distinct c` can end a query but `where c.isLoyal` cannot: `select` and `distinct` are handed the alias `c`, so they say what comes out, while `where` is handed a boolean expression that only says which records to keep. It also means `distinct c` is complete on its own - writing `distinct c select c` would just re-project what `distinct` already projected. The distinction matters most once several aliases are in scope: after a join, a trailing `where` would be ambiguous about whether the query returns the customer or the order, whereas `select` and `distinct` are explicit.

> **NOTE:** This is also why `order by` cannot be the last clause (see [Ordering](#ordering)). It reorders records but names no result; the ordering exists to feed a downstream `skip`, `take`, or `select`.

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
    where openOrders.any()
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

### Optionals
QL does not allow inspecting members of a value that might be absent until you have confirmed it is present. For example, if a customer might be `None`, its orders cannot be inspected until the `Some` case is established. The `is` operator both tests the variant and binds its payload:
```
from customers as c?
where c is Some(c)
from c.orders as o
select o;
```

The alias `c` is followed by a `?`, declaring it optional (`Customer?`) - it is an error to treat a `Customer?` as a `Customer` without first handling the absent case. `where c is Some(c)` keeps only the rows where the value is present and rebinds `c` to the inner value - a plain `Customer` - by *shadowing* the optional. From that point on, `c` is the unwrapped customer. A bound name is usable immediately, so `where c is Some(c) and c.isLoyal` filters and narrows in one clause.

> **NOTE:** Reusing the name (`Some(c)`) shadows the optional with its unwrapped value; use a fresh name (`Some(customer)`) instead if you also need the original optional. Either way, narrowing is an ordinary new binding, not a flow-sensitive re-typing of an existing variable - there is no separate rule where the compiler "just knows" `c` is non-optional within some region.

Because a pattern in a `from` also filters, the same query reads even more directly by matching in the source itself:
```
from customers as Some(c)
from c.orders as o
select o;
```

Here only the present customers flow through, each already unwrapped to `c`.

Another option is to substitute a value for the absent case rather than filtering it out:
```
from customers as c?
from (c?.orders ?? []) as o
select o;
```

The `?.` operator turns a member access on `None` into `None`, and the `??` operator replaces `None` with the value on its right. So if `c` is `None`, `c?.orders` is `None` and `?? []` makes the whole expression an empty collection.

## Distinct
Unique values can be retrieved using a `distinct` operation. For example:
```
from customers as c
distinct c;
```

> **NOTE:** `distinct` can be the last operation in a query, no `select` required - it names its output (`c`) just as `select` would. See [Terminating a query](#terminating-a-query).

In the example above, what if we wanted to base uniqueness by first name only? In that case, the `on` keyword can be used:
```
from customers as c
distinct c on c.firstName;
```

> **NOTE:** Multiple properties can be passed to `on` using a tuple, such as `c.firstName, c.lastName`. However, it can also be passed an anonymous type, such as `{ firstName: c.firstName, lastName: c.lastName }`. Anonymous types used in the context of `on` automatically implement equality based on the fields provided. However, specifying a tuple works just as well in this situation and requires less typing and overhead.

> **NOTE:** It can be helpful to think of `distinct` as a special form of `where`. After the `where` all subsequent operations will only see the unique values.

What if we wanted to just return the unique first names? We would first need to map from a customer to their first names, then use `distinct`:
```
from customers as c
select c.firstName as name
distinct name;
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
What if we wanted to return all customers, regardless of whether they had any orders? In other words, if a customer had orders, we would still get back every customer/order combination; however, for customers missing orders, we still want to get back one record, where the order is `None`.

The `left join` operation does just that:
```
from customers as c
left join orders as o on c.id == o.customerId
select { customer: c, order: o };
```

An interesting aspect of left joins is that the joined collection item (`o` in this case) is implicitly optional afterward. So, if `orders` is `Order[]`, then `o` will be treated as `Order?`. Any operations on `o` for the rest of the query (after the `on` clause) must first establish the `Some` case (e.g. `where o is Some(order)`). Within the `on` clause, though, `o` can be treated as `Order`.

**NOTE:** The only caveat is if `orders` is `Order?[]` to begin with, in which case the alias must be declared `o?` and the `None` case handled within the `on` clause. This should rarely be the case, as `None` records are uncommon in practice.

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
    where r is Some(role)
    select role;
```

After the `on` clause, both `ur` or `gr` can potentially be `None`, but not both - try to understand why. The compiler doesn't know that only one can be `None`, though. In the example, we use the `??` operator to grab whichever value is present first and confirm presence using `where r is Some(role)`, but we also could have used the `!` suffix operator to express our intent, like so:
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
where usedUserRoles.any() or usedGroupRoles.any()
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
    from customers as c
    join orders as o on c.id == o.customerId
    group o by c;
```

In this example, we are grouping the orders by their customers. Since we put `o` after the `group` keyword, the values in the groups will be orders, and the `key` for each group will be the customer.

If we want to group by customer, but only using the customer's first name, we use `on`:
```
let ordersByCustomer =
    from customers as c
    join orders as o on c.id == o.customerId
    group o by c on c.firstName;
```

## Aggregation
In the previous examples, after a `group by`, we are no longer working with a one-dimensional collection. Instead, we have a list of groups, which are themselves a type of list. Often, after grouping data, we want to summarize each group, going back to a one-dimensional collection. To do this, we can use `aggregate`.

For example, here's how we can count the orders for each customer:
```
let customerOrderCounts =
    from customers as c
    join orders as o on c.id == o.customerId
    group o by c as g
    aggregate {
        customer: g.key,
        orderCount: g.count()
    };
```

First, notice we provide an alias for the group, `g`. We use this in the `aggregate` operation below.

Here, the `aggregate` operation takes the list of customer groups produced from a `group by` and combines each into a result, one for each group.

As you can see, if we want to access the customer information (what we grouped by), we use the `key` property on the group.

An `aggregate` operation can be used even without a `group by`. Consider:
```
from customers as c
aggregate c.count();
```

This query returns the number of customers. The result is a single integer value.

Here is a list of common aggregate methods, each called on the collection:

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

> This set is open, not fixed - each is an aggregator value passed to a universal `reduce`, with a sugar method for convenience. See [Aggregate functions](./aggregates.md) for how aggregators are defined, extended, and translated per backend.

> **NOTE:** One curiosity of `aggregate` is that you can apply collection operations on an alias. In the other operations we explored previously, an alias refers to a single value in a collection. In `aggregate`, the same alias now refers to an entire collection! Any filtering applied to the collection up to that point is accounted for when aggregating the values.

## Ordering
Values can be sorted using the `order by` operation.
```
from [3, 1, 2] as v
order by v
select v; # [1, 2, 3]
```

> **NOTE:** An `order by` operation cannot be the last operation in a query - it names no result (see [Terminating a query](#terminating-a-query)).

Sorting can refer to specific fields, as well:
```
from customers as c
order by c.firstName
select c;
```

## Skip
The `skip` operation skips either a specific number of values in a query or while a condition persists:
```
from 1..10 as v
skip 3
select v; # [4, 5, 6, 7, 8, 9]
```
or
```
from 1..10 as v
skip v < 4
select v; # [4, 5, 6, 7, 8, 9]
```

## Take
The `take` operation takes a specific number of values in a query or while a condition persists:
```
from 1..10 as v
take 3
select v; # [1, 2, 3]
```
or
```
from 1..10 as v
take v < 4
select v; # [1, 2, 3]
```

## Windows
A window computes a value *for each row* using a set of related rows around it, without collapsing rows the way a `group by` does. Where a `group by` turns many rows into one row per group, a window keeps every row and attaches a computed value to it.

Windowing starts with the `partition by` operation. Unlike `group by`, it does *not* collapse rows; instead, it gives each row access to its *partition* - the collection of rows sharing the same key - bound to an alias. Within the query you then have two things available at once:

* `o` - the current row (an `Order`), as in any other query.
* `p` - the partition: a *collection* of the orders that share this order's `customerId`.

```
from orders as o
partition by o.customerId as p
let share = o.amount / p.amount.sum()
select { ...o, share };
```

This computes each order's share of its customer's total. Both aliases are at work: `o.amount` is *this row's* amount (a scalar), while `p.amount.sum()` sums across the whole partition (`p.amount` projects the amount of every order in `p`, and `.sum()` reduces it). Because `p` is plainly a collection and `o` is plainly a row, the expression says what it means.

> **NOTE:** `partition by` *adds* the partition alias without collapsing rows or removing any existing alias. This is the crucial difference from `group by`, which replaces the current row with a group (see [Windows and scope](#windows-and-scope)).

### Frames
A whole-partition aggregate like `p.amount.sum()` uses every row in the partition and does not depend on order. More often you want a *frame*: a subset of the partition relative to the current row, such as "the last three orders" or "everything up to now." A frame is simply a [slice](./ranges.md#ranges-as-indexes) of the partition.

A frame is only meaningful when the partition has an order, so that "preceding," "following," and position are defined. You establish that order by sorting *before* partitioning. `partition by` is *stable*: it preserves the order the rows are already in, so partitioning an ordered stream yields ordered partitions.

```
from orders as o
order by o.date
partition by o.customerId as p
let runningTotal = p[..=0].amount.sum()       # start of partition through the current row
let movingAvg    = p[-2..=0].amount.average()  # the current row and the two before it
let rowNumber    = p[..=0].count()             # a running count
select { ...o, runningTotal, movingAvg, rowNumber };
```

The order matters: `order by o.date` sorts the orders, and `partition by` then splits that sorted stream into per-customer partitions, each of which is therefore in date order. Sorting *after* `partition by` would reorder the output rows but leave each partition's internal order untouched - so the sort a frame depends on must come first.

When a partition alias is indexed, the offsets are measured *relative to the current row*: `p[0]` is the current row (the same order as `o`), negative offsets are preceding rows, and positive offsets are following rows. Each frame is an ordinary slice:

* `p[..=0]` - from the start of the partition through the current row (a *running* frame).
* `p[-2..=0]` - the current row and the two preceding it (a trailing frame of three).
* `p[0..]` - the current row through the end of the partition.
* `p[..]` - the entire partition (the same as `p` with no index).

Because a frame is just a sliced collection, you aggregate it with the same methods as any other collection - `.sum()`, `.average()`, `.count()`, and so on. There is no special windowing operator: a "window" is simply a `let` that slices and aggregates the partition.

> **NOTE:** A running frame (`p[..=0]`) is exactly a [scan](./aggregates.md#reduce-and-scan) - the running result at each row - while a whole-partition frame (`p[..]`) is a reduce broadcast to every row. Windowing is `reduce` and `scan` in query form.

> **NOTE:** Ordering changes the *type* of a partition, not merely its contents. Partitioning an ordered stream yields an *ordered* partition, which supports relative frames (`p[..=0]`, `p[-1]`, and the rest); partitioning an unordered stream yields an *unordered* partition, which supports only order-independent aggregates like `p.amount.sum()` over the whole partition. Asking for a relative frame on an unordered partition is a compile-time error - there is no "previous row" to refer to. This mirrors the way an ordered sequence is a distinct, more capable type than an unordered one.

### Offsets
Indexing a partition with a *single* offset returns that one row, rather than a collection - which is how you look at a neighboring row (what SQL calls `lag` and `lead`):
```
from orders as o
order by o.date
partition by o.customerId as p
let previousAmount = p[-1]?.amount  # the prior order's amount
select { ...o, previousAmount };
```

At the edges of a partition a neighboring row may not exist - `p[-1]` is `None` at the first row - so the usual [optional rules](#optionals) apply. That is why `p[-1]?.amount` uses `?.`, and `previousAmount` is inferred as optional.

### Row frames vs. value frames
The frames above count *rows*: the offset `-2` means "two rows back," so `p[-2..=0]` is always exactly three rows. Sometimes you instead want a frame defined by the *ordering value* - for example, "every order within the previous week," which may match any number of rows (including none). These two interpretations are genuinely different, and conflating them is a common source of confusion in SQL (its `ROWS` vs. `RANGE`).

This is exactly the [position-vs-value distinction](./ranges.md#position-vs-value) that applies to any slice, so QL distinguishes the two the same way - by what the index is written against, with no extra keyword:

* Bare offsets slice by **position**. `p[-2..=0]` is the current row and the two rows before it.
* An index written in terms of the ordering expression slices by **value**. Assuming orders are ordered by an integer `day`, this frame covers every order in the current order's week, however many that is:

```
from orders as o
order by o.day
partition by o.customerId as p
let weeklyTotal = p[o.day - 6 ..= o.day].amount.sum()
select { ...o, weeklyTotal };
```

Here `o.day` is *this row's* day, so `p[o.day - 6 ..= o.day]` selects every order in the partition whose `day` falls between this order's day minus six and this order's day. Because a value frame selects by magnitude, all rows sharing an ordering value (peers) fall in the frame together - usually the reason to prefer a value frame over a row frame.

Row and value frames line up shape-for-shape, differing only in axis:

* `p[..=0]` (row) and `p[..= o.day]` (value) - a running frame.
* `p[-2..=0]` (row) and `p[o.day - 2 ..= o.day]` (value) - a trailing frame.
* `p[..]` - the whole partition, the same either way.

> **NOTE:** A single-offset lookup like `p[-1]` is inherently a *row* operation - it names one neighboring row by position - so it has no value form. A value "offset" could match zero or many rows, which is what the range forms above express instead.

### Windows and scope
`partition by` is unusual among query operations. Most operations either leave the available aliases untouched (like `where`), extend them (like `let` and `join`), or replace them (like `select`, `group by`, and `aggregate`). `partition by` *extends* the scope: it adds the partition alias (`p`) while keeping the current row (`o`) and every other alias in place.

This is what makes windowing feel different from `group by` despite the two doing similar work. A `group by` replaces the current row with a group, so prior aliases are reachable only through the group's `key` (or by iterating the group). `partition by` keeps the row and hands you the partition alongside it, so nothing goes out of scope. A window is then just an ordinary `let` that slices and aggregates that partition - there is no separate windowing construct to learn.
