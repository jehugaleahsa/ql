# Queries
A query is an expression that generates a collection. Once a [source](./in-memory-sources.md) is defined, it can be referenced in a query.

## From
A query begins by specifying where the data comes from, using the `from` keyword. The simpliest query immediately returns the values from the source using the `select` operation:
```
let result =
    from customers as c
    select c;
```

> **NOTE:** This query is for demonstration purposes only. It is equivalent to `let result = customers;`

The `as` keyword is used to provide an alias for items in the collection. An alias *is required*.

> By convention, aliases tend to be small, but they can be more descriptive, like `customer` in the previous example. The reason for shorter aliases is to reduce the horizontal space needed to define queries, similar to SQL.

After the `from` operation, items from the collection are referred to using the alias. That's why `c` appears in the `select` operation.

Multiple `from` operations can appear in the same query. Imagine a `customer` object had a list of its `order`s in an `orders` property:
```
from customers as c
from c.orders as o
select { customer: c, order: o }
```

This query will return every customer/order combination.

## Select
The `select` operation defines which properties are returned (or "projected") by a query.

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

If properties are copied from multiple entities, properties with overlapping names will comes from the latter entity:
```
from customers as c
select { ...c, ...o }; # id will come from o, not c
```

The selected entity can customize the property name by putting the name before a colon:
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

The nested entities can be selected, as well:
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

## Where
In order to filter records, a `where` operation can be performed. After the `where` keyword, a boolean expression must be provided. Only records satisfying the `where` expression will be included in the results.
```
let loyalCustomers =
    from customers as c
    where c.isLoyal
    select c;
```

Multiple `where` operations can appear in the same query. For example, two `where` operations are peformed in the next example to filter the customers and then the orders:
```
let loyalCustomersWithOpenOrders =
    from customers as c
    where c.isLoyal
    join orders as o on c.id == o.customerId
    where o.status != 'closed'
    select { customer: c };
```

Multiple boolean expressions can be combined using `and`, `or`, and `not`, with parentheses to control precedence. Furthermore, a query can filter the same record multiple times using multiple `where` operations. The following query:
```
from customers as c
where c.isLoyal and c.isCardHolder
select c;
```
is equivalent to:
```
from customers as c
where c.isLoyal
where c.isCardHolder
select c;
```

> **NOTE:** The same cannot be said for `or`.

## Join
Not all sources will store related information in nested properties, like in our previous example. Some sources, like relational database sources, will store customers and orders separately, using IDs to link related records.

For the next example, assume we have a `customer` source and an `order` source, but this time `order` has a `customerId` property for linking back to a customer. In that case, we can use a `join` operation to match customers and orders together using the `customerId` property:
```
from customers as c
join orders as o on c.id == o.customerId
select { customer: c, order: o };
```

A `join` operation will also return every customer/order combination; however, it does it by matching a condition following the `on` keyword. The condition can be much more complicated to allow for more complicated join scenarios.

> **`from` vs `join`?** If your data is already structured in such a way that related objects are in nested properties, `from` is a logical choice. Otherwise, a `join` may make more sense. A `from` with a filter (a.k.a, `where`) can have the same effect as a `join`. However, for a particular source implementation, using an explicit `join` can lead to more optimized queries.

When the `join` keyword appears by itself, a match must be found or the records are filtered out. For example, the query above would filter out customers that did not have any orders.

## Left join
What if we wanted to return all customers, regardless of whether they had any orders? In other words, if a customer had orders, we would still get back every customer/order combination; however, for customers missing orders, we still want to get back one record, where the order is just `null`.

The `left join` operation does just that:
```
from customers as c
left join orders as o on c.id == o.customerId
select { customer: c, order: o };
```

> **NOTE:** QL has no equivalent to `RIGHT OUTER JOIN` or `FULL OUTER JOIN` in SQL. See [below](#outer-joins) for more details.

## Outer joins
Above, we briefly mentioned simulating SQL's `RIGHT` and `FULL OUTER JOIN`s. Now that we have covered `where`, we can show a full example. 

A `RIGHT OUTER JOIN` is achieved by reversing the `from` and the `join` in this example:
```
from orders as o
left join customers as c on c.id == o.customerId
select { customer: c, order: o };
```

A `FULL OUTER JOIN` is more complicated:
```
let cs = 
    from customers as c
    left join orders as o on c.id == o.customerId
    select { customer: c, order: o };
let os =
    from orders as o
    left join customers as c on c.id == o.customerId
    select { customer: c, order: o };
let result = cs union os by customer.id, order.id;
```

The important take away is that the equivalent of a `FULL OUTER JOIN` requires looping over the collections twice, as well as a way of `union`-ing the results. Unlike SQL, QL has no guarantee that it can uniquely identify `customer` or `order` entities. Therefore, the `by` clause specifies which properties to uniquely identify records using.