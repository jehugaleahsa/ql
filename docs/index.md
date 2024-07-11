# Intoduction
**Q**uery **L**anguage is an attempt to informally define a grammar for a next-gen programming language. It can also behave as a SQL transpiler. The premise is that nearly all computations on a modern day computer can be described using a series of SQL-like queries that are composable. The hope of this specification is to define a general-purpose language that is suitable for the daily programming activities.

Those familiar with SQL, functional programming, or vector programming (e.g., R) concepts should feel right at home. For those coming from a procedural programming background, it might take some getting used to.

# Table of contents
1. [Introduction](#intoduction)
2. [Table of contents](#table-of-contents)
3. [Quick Start](#quick-start)
4. [In-memory sources](./in-memory-sources.md#in-memory-sources)
    * [Empty collections and null](./in-memory-sources.md#empty-collections-and-null)
    * [Concatenating](./in-memory-sources.md#concatenating)
    * [Querying](./in-memory-sources.md#querying)
    * [Splicing](./in-memory-sources.md#splicing) 
    * [Mutable collections](./in-memory-sources.md#mutable-collections)
    * [Appending values](./in-memory-sources.md#appending-values)
    * [Updating values](./in-memory-sources.md#updating-values)
    * [Deleting values](./in-memory-sources.md#deleting-values)
5. [Primitive Types](./primitive-types.md)
    * [Literals](./primitive-types.md#literals)
        * [Underscore separators](./primitive-types.md#underscore-separators)
    * [Text and strings](./primitive-types.md#text-and-strings)
        * [Escape sequences](./primitive-types.md#escape-sequences)
        * [Raw strings](primitive-types.md#raw-strings)
    * [Conversion](./primitive-types.md#conversion)
        * [Overflow/Underflow](./primitive-types.md#overflowunderflow)
        * [Unicode checks](./primitive-types.md#unicode-checks)
6. [Vectors](./vectors.md)
    * [Dimensions and broadcasting](./vectors.md#dimensions-and-broadcasting)
    * [Null](./vectors.md#null)
        * [Lifted arithmetic operations](./vectors.md#lifted-arithmetic-operations)
7. [Queries](./queries.md)
    * [From](./queries.md#from)
    * [Select](./queries.md#select)
    * [Where](./queries.md#where)
    * [Join](./queries.md#join)
    * [Left join](./queries.md#left-join)
    * [Outer joins](./queries.md#outer-joins)
8. [Combining Collections](./combining-collections.md)

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
group by c as g
aggregate {
    customer: g.c,
    count: count(g)
};
```

```
# Apply 20% coupon to an order
from orders as o
where o.id == 123
update o { amount: o.amount * .8 };
```