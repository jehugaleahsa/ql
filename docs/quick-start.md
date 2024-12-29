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
    customer: g.key,
    count: count(g)
};
```

```
# Apply 20% coupon to an order
from orders as o
where o.id == 123
update o { ...o, amount: o.amount * .8 };
```