# Entities
Beyond [primitive types](./primitive-types.md), new types can be declared, called entities. They allow grouping values together, including both primitive types and other entities.

An entity is composed of named properties, and each property has a type. For example, a customer could be represented like this:
```
let Customer = type {
    id: i32,
    name: String,
    address: String
};
```

Now imagine we realize that addresses are more complicated. We can create an entity for addresses, like so:
```
let Address = type {
    line1: String,
    line2: String?,
    municipality: String,
    region: String,
    postalCode: String
};
```

> **NOTE:** If a property's value can be `null`, put a `?` after the type to indicate it. Alternatively, you can say `Optional<T>`.

Now, we can update our `Customer` entity to look like this:
```
let Customer = type {
    id: i32?,
    name: String,
    address: Address
};
```

> **NOTE:** The point here is that entities can contain other entities.

An instance of a `Customer` can then be created like this:
```
let customer: Customer = {
    id: 1,
    name: "Big Mart, Inc.",
    address: {
        line1: "Big Mart Ln",
        municipality: "Greenborough",
        region: "WI",
        postalCode: "12345-123"
    }
};
```

Alternatively, the customer could be initialized in steps:
```
let address: Address = {
    line1: "Big Mart Ln",
    municipality: "Greenborough",
    region: "WI",
    postalCode: "12345-123"
};
let customer: Customer = {
    id: 1,
    name: "Big Mart, inc.",
    address: address
};
```

## Traits
Entity declarations using the `type` keyword can also be referred to as "traits". In order to share properties and methods across related entities, traits can be combined.

For example, let's define a common trait for all entities mapping to a database, giving them an `id` property:
```
let DBModel = type {
    id: i32?
};
```

Now we can redefine `Customer` to be a type of `DBModel`:
```
let Customer = type DBModel {
    name: String,
    address: Address
};
```

Zero, one, or more traits can be listed after the `type` keyword, separated by commas.

In the example above, since `DBModel` already defines an `id` property, it does not need to be repeated in `Customer`.

> **NOTE:** QL doesn't support the classical notion of inheritance, as found in object-oriented programming languages. Many of the more useful features are supported, however.

## Anonymous types
Within [queries](./queries.md), defining anonymous types is quite common. For example:
```
from customer as c
select {
    id: c.id,
    firstName: c.firstName
};
```

Here, we are creating an anonymous type from a `Customer` with only two properties, `id` and `firstName`. The types of `id` and `firstName` are inferred automatically, including whether a `null` is possible. Being anonymous, these types cannot be constructed outside of a query by name.

## Copying
In many operations, it's common to treat entities as immutable. Instead of directly updating a property, you can make a copy of an object, changing the value of individual properties instead:
```
let customer: Customer = {
    id: 123,
    firstName: "Bob",
    lastName: "Smith",
    age: 23
};
let updated: Customer = {
    ...customer,
    age: 24
};
```

After this operation, `customer` and `updated` will have all the same values, except `updated`'s `age` will be `24` instead of `23`. This same syntax is used as a short-hand inside [update operations](./in-memory-sources.md#updating-values).

For nested properties, copying can be a little more verbose:
```
let customer: Customer = {
    id: 123,
    firstName: "Bob",
    lastName: "Smith",
    address: {
        line1: "123 Palm Beach",
        state: "CA",
        zip: "55555"
    }
};
let updated: Customer = {
    ...customer,
    address: {
        ...customer.address,
        zip: "56789"
    }
};
```

> **NOTE:** It's important not to mix mutable and immutable code, especially when dealing with nested objects and collections. When copying objects, a shallow copy is performed by default. This means modification to a shared object may impact every object that shares it. It's usually better to start with a design that assumes immutability and only move toward mutability once it's been shown to improve performance.