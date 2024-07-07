# Intoduction
**Q**uery **L**anguage is an attempt to informally define a grammar for a next-gen programming language. It can also behave as a SQL transpiler. The premise is that nearly all computations on a modern day computer can be described using a series of SQL-like queries that are composable. Operations which cannot be easily performed in QL are best left done in another programming language. The hope of this specification is reduce the overall number of situations where QL cannot be used, defining a general-purpose language that is suitable for the majority of software development.

Those familiar with SQL, functional programming, or vector programming (e.g., R) concepts should feel right at home. For those coming from a procedural programming background, it might take some getting used to.

# Table of contents
1. [Introduction](#intoduction)
2. [Table of contents](#table-of-contents)
3. [Quick Start](#quick-start)
3. [In-memory sources](./in-memory-sources.md#in-memory-sources)
    * [Empty collections and null](./in-memory-sources.md#empty-collections-and-null)
    * [Concatenating](./in-memory-sources.md#concatenating)
    * [Querying](./in-memory-sources.md#querying)
    * [Splicing](./in-memory-sources.md#splicing) 
    * [Mutable collections](./in-memory-sources.md#mutable-collections)
    * [Appending values](./in-memory-sources.md#appending-values)
    * [Updating values](./in-memory-sources.md#updating-values)
    * [Deleting values](./in-memory-sources.md#deleting-values)

# Quick Start
For this quick start, assume we have customers with orders, and those orders are made up of items.

```
# Retrieve all customers
from customers as c
select c;
```

```
# Retrieve a specific customer
from customers as c
where c.id == 123
select c;
```

```
# Count the number of orders per customer
from customers as c
join orders as o on c.id == o.customerId
group o by c as g
aggregate {
    customer: g.c,
    count: count(g)
};
```

```
# Delete customers without any orders
from customers as c
let hasOrders =
    from orders as o
    where o.customerId == c.id
    aggregate any(o)
where not hasOrders
delete c;
```

```
# Update orders with 20% coupon
from orders as o
where o.id == 123
update o { amount: o.amount * .8, discount: .2 };
```