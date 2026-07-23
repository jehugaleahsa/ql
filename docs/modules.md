# Modules
A *module* groups related definitions and controls which of them the outside world can see. Modules are how a QL program is organized into files and folders, and how one part of a program borrows names defined in another.

Every definition lives in some module. Within a module, names refer to one another directly; across modules, a name is reachable only if the module that owns it chooses to `export` it, and only if the code reaching for it `import`s it.

## Files and folders
A module is either a **file** or a **folder**:

* A `.ql` file is a module. Its definitions are that module's contents.
* A folder is a module whose contents are assembled from a *root file* plus any submodules the root file declares.

A folder module's root file is named `module.ql`. The one exception is the folder at the very root of the program, whose root file is the entry point: `main.ql` for an application, `library.ql` for a library. Which entry file is chosen depends on the project type, and the choice will be overridable (for example, by a command-line argument) once the tooling is specified.

Modules form a tree: the root module contains submodules, each of which may contain further submodules. A module's position in that tree is what the visibility and path rules below are expressed in terms of.

## Declaring submodules
A folder's children are **explicit** - the folder's root file declares each one with the `module` keyword. Nothing is discovered automatically:
```
module trig;
module algebra;
```

A declared name resolves to either a file or a folder:

* `module trig;` looks for `trig.ql` (a file module), or
* a `trig/` folder containing a `module.ql` root file (a folder module).

The duality is resolved per name, at the point of declaration.

Declaring a submodule is a separate act from importing from it. `module trig;` says *trig is part of my tree* - it brings the submodule into existence and compiles it. [Importing](#importing) is what pulls names *out* of it into the current scope. Keeping the two apart means each keyword has one job.

> **NOTE:** A submodule is private by default, like any other definition. Prefix the declaration with `export` to widen that - see below.

## Exporting
Visibility is **explicit**. A definition is private unless marked with `export`. "Private" means visible within its own module **and that module's descendant modules** - a submodule can see the definitions of its ancestors, but the outside world cannot.

```
export let sqrt = fn(x: f64): f64 => ...;   # public
let normalize = fn(x: f64): f64 => ...;     # private to this module and its descendants
```

`export` takes an optional scope in parentheses, widening or narrowing exactly how far a name reaches:

| Form | Visible to |
|---|---|
| *(no `export`)* | this module **and its descendant modules** - nothing outside |
| `export(self)` | the same - written explicitly |
| `export(super)` | the parent module (and its descendants) |
| `export(package)` | every module in the current package |
| `export(in <path>)` | a named ancestor module and everything under it |
| `export` | fully public - anyone who can reach the module |

```
export(super) let helper = fn() => ...;
export(package) let VERSION = "1.0.0";
```

`export` applies to submodule declarations too, controlling whether a child module is visible beyond its parent:
```
export module algebra;          # public submodule
export(package) module internal;
module scratch;                 # private submodule
```

> **NOTE:** `export(self)` is exactly equivalent to writing no `export` at all. It exists for symmetry, so the ladder reads complete; in practice a plain unmarked definition is the idiomatic way to say "private".

## Importing
A name from another module is brought into scope with `from ... import`, naming the source module first:
```
from math import sqrt;
```

The `from`-first ordering is deliberate: once the source is named, tooling knows the set of available names and can complete the import list for you - there is no need to write an empty placeholder and go back to fill it in.

Multiple names are grouped in braces. A single name needs no braces; braces are also the natural home for a multi-line list:
```
from math import { sqrt, pi };

from math import {
    sqrt,
    pi,
    tau,
};
```

To bring in the module *itself* as a namespace, import `self`. This can stand alone or sit alongside individual names:
```
from math import self;              # `math.sqrt`, `math.pi`, ...
from math import { self, sqrt };    # both the `math` namespace and `sqrt` directly
```

A name may be renamed on the way in with `as`:
```
from math import sqrt as root;
from math import self as m;
```

> **NOTE:** There is no bare `import math;` form. Importing the namespace is spelled `from math import self;`, so every import flows through the same `from ... import` construct.

## Re-exporting
A module can expose names it did not define - a *barrel* that forwards from elsewhere. The forwarding form mirrors `import`, replacing `import` with `export`:
```
from trig export { sin, cos, tan };
from algebra export solve;
```

`from ... export` forwards names *without* binding them into the local scope. That is the point: if a file wants to both *use* a name and *forward* it, it must write the `import` as well as the `export`. The friction is intentional - it keeps a barrel file either a pure set of forwards or a real module with its own code, never a muddle of the two. It takes the same scope knob as any other export:
```
from trig export(package) { sin, cos };
```

## Paths
A path names where a definition lives. Segments are separated with `.`, the same operator used everywhere else in QL:
```
from package.math.trig import sin;
```

A path may start from one of three roots, the same three words that appear as export scopes:

* `self` - the current module,
* `super` - the parent module,
* `package` - the root of the current package.

```
from super.utils import trim;
from package.math import sqrt;
```

One vocabulary describes *where a name lives* whether you are restricting its visibility (`export(super)`) or navigating to it (`super.utils`).

## Traits and the orphan rule
Trait implementations are not imported by name. Once a trait and a type are both in scope, the relevant [`impl`](./entities.md#traits) applies automatically - there is nothing to bring in explicitly.

This works because of the [orphan rule](./entities.md#where-an-impl-may-be-written): an `impl Trait for Type` is allowed only in the module that defines the trait or the module that defines the type. That guarantees at most one implementation of a given trait for a given type anywhere in a program, so a type's behavior never depends on which modules happen to be imported.
