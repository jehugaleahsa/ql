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
* `sum` - the sum of the values in the collection (numeric)
* `average` - the average of the values in the collection (numeric)
* `product` - the product of the values in the collection (numeric)
* `joined` - the values in the collection joined together into a string, separated by a separator

> **NOTE:** One curiosity of `aggregate` is that you can apply collection operations on an alias. In the other operations we explored previously, an alias refers to a single value in a collection. In `aggregate`, the same alias now refers to an entire collection! Any filtering applied to the collection up to that point is accounted for when aggregating the values.