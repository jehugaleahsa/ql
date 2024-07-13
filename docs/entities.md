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

> **NOTE:** If a property's value can be `null`, put a `?` after the type to indicate it.

Now, we can update our `Customer` entity to look like this:
```
let Customer = type {
    id: i32,
    name: String,
    address: Address
};
```

> **NOTE:** The point is that entities can contain other entities.

An instance of a `Customer` can then be created like this:
```
let customer: Customer = {
    id: 1,
    name: "Big Mart, inc.",
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
Entity declarations using the `type` keyword can also be referred to as "traits". In order to share properties across related entities, traits can be combined.

For example, let's define a common trait for all entities mapping to a database, giving them an `id` property:
```
let DBModel = type {
    id: i32
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