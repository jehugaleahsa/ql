# Queries
A query is an expression that generates a collection. Once a [source](./in-memory-sources.md) is defined, it can be referenced in a query.

## From
A query begins by specifying where the data comes from, using the `from` keyword. The simplest query immediately returns the values from the source using the `select` operation:
```
let result =
    from customers as c
    select c;
```

> **NOTE:** This query is for demonstration purposes only. It is equivalent to `let result = customers;`

The `as` keyword is used to provide an alias for items in the collection. An alias *is required*.

> By convention, aliases tend to be small, but they can be more descriptive, like `customer` in the previous example. The reason for shorter aliases is to reduce the horizontal space needed to define queries, similar to SQL. Chose a naming convention that works best for you.

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

If properties are copied from multiple entities, properties with overlapping names will come from the latter entity. Assume in the next example that both customers and orders have an `id` property:
```
from customers as c
from c.orders as o
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

### Arithmetic expressions and other computations
A projected property can also be an expression involving arithmetic operations or method calls, such as:
```
from customers as c
select { 
    score: (100 * c.orders.count()) as f64 + (c.creditRating * 0.05) 
};
```

### Let
Anywhere within a query, expressions can be broken out into smaller chunks by giving them an alias:
```
from customers as c
let orderScore = 100f64 * c.orders.count() as f64
let creditScore = c.creditRating * 0.05
let totalScore = orderScore + creditScore
select { score: totalScore };
``` 

This can help to keep a line from getting too long, allow for reuse, and make complex operations easier to understand.

## Distinct
Unique values can be selected using the `distinct` operation. For example:
```
from customers as c
distinct c.firstName as fn
select fn;
```

> **NOTE:** By itself, `distinct` cannot project values and so cannot be the last operation in a query. Therefore, we needed to provide an alias of `fn` in this example.

In the example above, what if we wanted to return the full customer, only returning customers with unique first names instead? In that case, the `using` keyword can be used:
```
from customers as c
distinct c using(c.firstName)
select c;
```

## Where
In order to filter records, a `where` operation can be performed. After the `where` keyword, a boolean expression must be provided. Only records satisfying the `where` expression will be included in the results.
```
let loyalCustomers =
    from customers as c
    where c.isLoyal
    select c;
```

Multiple `where` operations can appear in the same query. For example, two `where` operations are performed in the next example to filter the customers and then the orders:
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
Not all sources will store related information in nested properties, like in our previous examples. Some sources, like relational database sources, will store customers and orders separately, using IDs to link related records.

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

An interesting aspect of left joins is that the joined collection item (`o` in this case) is implicitly nullable. So, if `orders` is `Order[]`, then `o` must be `Order?`. Any operations on `o` for the rest of the query (after the `on` clause) must check for `null`. 

> **NOTE:** A right join can be simulated by reversing the order collections are joined.

## Full join
The last type of join is a `full join`, which will join two related objects if they satisfy the condition, but will return every object from either collection even if no match is found. To demonstrate, we need two collections that can share information but don't form a parent/child relationship. Let's imagine a Role can be assigned to a User or a Group of Users, and we want to find all Roles that are assigned to at least one. Here's how we could do that.
```
let userRoles =
    from users as u
    join userRoles as ur on u.id == ur.userId
    join roles as r on ur.roleId == r.id
    select r;
let groupRoles =
    from groups as g
    join groupRoles as gr on g.id == gr.groupId
    join role as r on gr.roleId == r.id
    select;
let assignedRoles =
    from userRoles as ur
    full join groupRoles as gr on ur.id == gr.id
    let r = ur ?? gr
    where r != null
    select r;
```

After the `on` clause, both `ur` or `gr` can potentially be `null`, but not both. Try to understand why. The compiler doesn't know this, though. In the example, we use the `??` operator to grab whichever value is non-null first and check for `null` using `where r != null`, but we also could have used the `!` suffix operator to express our intent, like so:
```
let assignedRoles =
    from userRoles as ur
    full join groupRoles as gr on ur.id == gr.id
    select (ur ?? gr)!;
```

This example is just for demonstration purposes. A much easier solution is as follows:
```
from roles as r
let hasUserRoles =
    from userRoles as ur
    where ur.roleId == r.id
    aggregate any(ur)
let hasGroupRoles =
    from groupRoles as gr
    where gr.roleId == r.id
    aggregate any(gr)
where hasUserRoles or hasGroupRoles
select r;
```

## Grouping
Groups can be formed using the `group by` keywords. When grouping, you must identify what entities are being grouped and what the key (unique value) is to group them by. For example:
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

The result of a `group by` operation is a list of `Grouping<K, V>` objects, where `K` is the type of the key (what you're grouping by) and where `V` is what each group contains. The type of `K` and `V` is inferred automatically from the value that appears after the `group` and `by` keywords, respectively.

A `Grouping` is a type of vector with an additional `key` property. Whatever expression appears after the `by` is used to populate the `key` value for each group.

In the example above, we are grouping the entire customer (`c`) by its `firstName` attribute. The group for "Sue" will have two items (customers) and the `key` will equal `"Sue"`.

Things can get more interesting when multiple entities are joined together. For example:
```
let ordersByCustomer = 
    from customer as c
    join order as o on c.id = o.customerId
    group o by c;
```

In this example, we are grouping the orders by their customers. Since we put `o` after the `group` keyword, the values in the `Grouping` will be orders.

### Aggregation
Here's how we can count the orders for each customer:
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

Here, the `aggregate` operation takes the list of `Grouping<Customer>` objects produced from a `group by` and combines each into a result, one for each group.

As you can see, if we want to access the customer information (what we grouped by), we use the `key` property on the group.

An `aggregate` operation can be used even without a `group by`. Consider:
```
from customers as c
aggregate count(c);
```

This query returns the number of customers. The result is a single integer value.

### Keys
Be sure the value(s) you group by have a notion of "equality". For example, a database library may ensure that only a single instance of a record exists in memory for each row in the database, using the primary key under the hood to uniquely identify each record. In that case, "equality" will be based on identity (the location of the record in memory).

Other types, such as primitive values and strings, have a default meaning for "equals" which probably match you're expectations. If you need more control over the way grouping occurs, you have a few choices:
```
let ordersByCustomerId =
    from customer as c
    join order as o on c.id = o.customerId
    group o by c.id as g
    aggregate {
        customerId: g.key
        orderCount: count(g)
    };
```

Notice we lose access to the customer using this approach. If you need access to the full customer, you can use the `using` keyword, like so:
```
let ordersByCustomerId =
    from customer as c
    join order as o on c.id = o.customerId
    group o by c using (id) as g
    aggregate {
        customer: g.key
        orderCount: count(g)
    };
```

So, even if your record type doesn't support "equality" by default, you an control how grouping behaves.

> Notice that you cannot qualify the `id` property by `c`, like `c.id`. You can only access properties of the object you are grouping by.