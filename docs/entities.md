# Entities
Beyond [primitive types](./primitive-types.md), new data types can be declared with the `struct` keyword. These are called entities: they group values together, including both primitive types and other entities.

An entity is composed of named properties, and each property has a type. For example, a customer could be represented like this:
```
let Customer = struct {
    id: i32?,
    name: String,
    address: String
};
```

Now imagine we realize that addresses are more complicated. We can create an entity for addresses, like so:
```
let Address = struct {
    line1: String,
    line2: String?,
    municipality: String,
    region: String,
    postalCode: String
};
```

> **NOTE:** If a property's value can be absent, put a `?` after the type to make it optional. Equivalently, you can write `Optional<T>`.

Now, we can update our `Customer` entity to look like this:
```
let Customer = struct {
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
A *trait* describes behavior: a set of methods a type can provide. Traits are how QL shares behavior across types, taking the place of the inheritance found in object-oriented languages.

A trait is declared with `trait`, listing the methods it requires. It may also supply *default* methods, written in terms of the required ones:
```
let Comparable<T> = trait {
    let compareTo = fn(self: Self, other: T): i32;      # required
    let lessThan = fn(self: Self, other: T): bool {     # default - built on compareTo
        self.compareTo(other) < 0
    };
};
```

A type opts into a trait with an `implement` block, supplying the required methods. The defaults then come for free:
```
let Version = struct {
    major: i32,
    minor: i32
};

implement Comparable<Version> for Version {
    let compareTo = fn(self: Self, other: Version): i32 {
        # ...compare major, then minor
    };
};
```

`Version` now has both `compareTo` and `lessThan`, even though it only implemented `compareTo`. Implementing a small required core to unlock a larger set of operations is the central pattern of the trait system - the [collection traits](./collections.md#common-collection-traits) work the same way, where implementing `iterator()` unlocks the rest.

A type can implement as many traits as it likes, each in its own `implement` block:
```
let Customer = struct {
    id: i32?,
    name: String,
    address: Address
};

implement Comparable<Customer> for Customer { ... }
implement Serializable for Customer { ... }
```

> **NOTE:** QL has no classical inheritance. A type is never a subtype of another; it simply *implements* traits. Shared behavior comes from traits and their default methods, not from a class hierarchy - which sidesteps the fragile-base-class and diamond problems inheritance is prone to.

### Where an `implement` block may be written
So that trait resolution is never ambiguous, QL enforces one rule (Rust calls it the *orphan rule*): an `implement Trait for Type` is allowed only if **you define the trait, or you define the type**, in the same module. You can implement your own trait for someone else's type, or someone else's trait for your own type - but never a foreign trait for a foreign type.

This guarantees there is at most one implementation of a given trait for a given type anywhere in a program, so behavior never depends on which modules happen to be imported. When you need to bridge two things you don't own, define a small trait of your own and implement it for the foreign type, or wrap that type in one of yours.

### Composing traits
A trait can require other traits by intersecting them with `&`. An implementor must satisfy every part:
```
let Appendable<T> = trait
    & Iterable<T>
    & {
        let add = fn(self: Self, item: T);
        let isEmpty = fn(self: Self): bool { self.count() == 0 }   # default - count() comes from Iterable
    };
```

To `implement Appendable<T> for X`, the compiler checks that `X` also implements `Iterable<T>` - you cannot be `Appendable` without being `Iterable`. In return, `Appendable`'s own default methods, and any code bounded by `Appendable`, may call the required traits' methods, like `count()` above.

The parts are joined with `&`, read as "and": a value satisfying `A & B` implements both. A trait's own members are written as an anonymous `& { ... }` block, so a trait is uniformly an intersection of the traits it requires and the members it introduces. Leading `&` operators are allowed, so each part lines up on its own line; the same reads fine inline: `trait Iterable<T> & { let add = fn(self: Self, item: T); }`.

When a trait adds nothing of its own - it just bundles others under a name - drop the `trait` keyword and the block, and a plain `let` aliases the intersection:
```
let ReadWrite = Readable & Writable;
```
Any type that is both `Readable` and `Writable` is a `ReadWrite`, with no separate `implement` block required - the same way `let Meter = f32` aliases an existing type.

> **NOTE:** `&` intersects: a value must implement all parts. It is the composition counterpart to `|` (union), which will describe a value that is one of several types, once union types are added.

## Anonymous types
Within [queries](./queries.md), defining anonymous types is quite common. For example:
```
from customers as c
select {
    id: c.id,
    firstName: c.firstName
};
```

Here, we are creating an anonymous type from a `Customer` with only two properties, `id` and `firstName`. The types of `id` and `firstName` are inferred automatically, including whether each is optional. Being anonymous, these types cannot be constructed outside of a query by name.

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

## Aliases
A plain `let` can also give an existing type another name - a type alias:
```
let Meter = f32;
```

There is no separate keyword for this. `struct` and `trait` introduce *new* types, and an ordinary `let` binding aliases an existing one - consistent with the rest of the language, where everything is a `let`.