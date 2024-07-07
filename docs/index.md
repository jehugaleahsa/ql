# Intoduction
**Q**uery **L**anguage is an attempt to informally define a grammar for a next-gen programming language. It can also behave as a SQL transpiler. The premise is that nearly all computations on a modern day computer can be described using a series of SQL-like queries that are composable. Operations which cannot be easily performed in QL are best left done in another programming language. The hope of this specification is reduce the overall number of situations where QL cannot be used, defining a general-purpose language that is suitable for the majority of software development.

Those familiar with SQL, functional programming, or vector programming (e.g., R) concepts should feel right at home. For those coming from a procedural programming background, it might take some getting used to.

# Table of contents
1. [Introduction](#intoduction)
2. [Table of contents](#table-of-contents)
3. [In-memory sources](./in-memory-sources.md#in-memory-sources)
    * [Empty collections and null](./in-memory-sources.md#empty-collections-and-null)
    * [Concatenating](./in-memory-sources.md#concatenating)
    * [Querying](./in-memory-sources.md#querying)
    * [Splicing](./in-memory-sources.md#splicing) 
    * [Mutable collections](./in-memory-sources.md#mutable-collections)
    * [Appending values](./in-memory-sources.md#appending-values)
    * [Updating values](./in-memory-sources.md#updating-values)
    * [Deleting values](./in-memory-sources.md#deleting-values)