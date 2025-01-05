# Motivation
For some practitioners, simple scripts are all the more we need from our programming languages. There are many professions where manipulating data is the primary goal. Languages and platforms like R, Python, MATLAB, SAS and SPSS dominate these fields, while general-purpose developers may never work with any of these. Python is a relative newcomer in this arena, in fact. It turns out most of these data munging, statistical, and analysis jobs also involve knowing quite a bit of SQL. Over time, we become "used to" SQL's oddities and may even come to like it for all its eccentricities. What if we could provide a modern, more expressive syntax that works as well on a relational database as it does on a CSV file?

For more general-purpose developers, we're trained to think in terms of branching (`if/else`) and loops (`while`). We learn to sub-divide our work into functions and other organization mechanisms (modules, packages, classes, etc.). Overall, though, we write code as if we own the entirety of the underlying machine, where execution goes from the start of our main function until completion.

Much later on we *might* we get introduced to asynchronous programming, concurrency, remote execution, distributed systems, and other complex topics. Asynchronous programming has become more popular since the 2000's, showing up in JavaScript most predominantly. It turns out JavaScript is a gentle introduction to concurrency, as JavaScript is a single-threaded environment. Delving into concurrency in other languages, such as Java or C#, suddenly multiple threads can be running at the same time, and suddenly programmers must concern themselves with things like data races, mutexes, and so on. Despite there being many techniques for managing this complexity, it is not a topic I have seen covered in great detail or to my satisfaction.

Some languages like C#, JavaScript, and Rust have introduced `async/await` in order to write asynchronous code in a fashion familiar to synchronous programming cronies, with varying levels of success. While these language constructs can greatly improve clarity, there's a common trend for developers to misuse or even abuse them. A common example is performing `await` inside a loop, when an action could have been created for each record and ran in parallel:
```JavaScript
// Bad
const results = [];
for (const url of urls) {
    const result = await fetch(url);
    results.push(result);
}

// Good
const results2 = await Promise.all(urls.map(u => fetch(u)));
```

It turns out, to reap many of the rewards of asynchronous coding, operations are best thought of as dependency graphs, where the results of each operation can feed into one more downstream dependencies. Wherever two operations can run in parallel, they probably should. This explains why `Promise`-like classes take on their particular form. Despite all the advancements, most us of are still writing code in a synchronous style.

## Data distribution and SQL
For decades, we've been slowly approaching a point where a single machine can no longer process a request. Not only is the scale of data increasing, but the data itself is geographically diverse. When we need to combine data from different sources, we're left writing completely different code for each platform. We're left trying to figure out where and how to combine the data so that it can be processed together. Even relational databases are hitting their limits.

Speaking of relational databases, SQL has been around for a long time. While there are many dialects, it's mostly standard. Even when a non-relational database crops up - perhaps touting it can store tables in a distributed fashion - it eventually ends up supporting *some* dialect of SQL.

SQL has many problems, though. There's just something about the syntax that's counter-intuitive. It feels like you only make breakthroughs when you learn some "trick".

The language was also not designed with auto-completion and other modern-day programming concepts in mind. I often find myself writing `SELECT * FROM abc` upfront, only to go back and replace the `*` with actual column names. Just by listing the `FROM`, suddenly tooling can help me to see what columns are available.

There are plenty of weird edge cases, as well. Why does `SELECT COUNT(*) from abc` return a single integer value, but `SELECT xyz FROM abc` returns multiple rows? Why are `COUNT`, `MAX`, `MIN`, and a handful of other function magical in this regard? Other functions seem to operate on individual values, not the entire table... just strange. I think it's these subtleties that make SQL hard to learn and reason about.

## Functional programming
These days, a lot of us are introduced to at least some functional programming concepts early on. This was not the case several decades ago, where functional programming was a niche topic left for math nerds. Even today, there are very few functional programming languages in mainstream use. Probably Haskell and Scala are the two most used, and Scala is just a general-purpose programming language with lots of functional programming concepts in-grained.

The more popular programming language have adopted the most useful parts: filter/map/reduce, immutability, lambdas, and closures. Other FP topics, like currying, algebraic types, etc. are left at the curb. Of all these, filter/map/reduce has seen the widest adoption.

In Java, there's the Streams API. In C#, there's LINQ. In Python, there's comprehensions and itertools. In JavaScript, there's several libraries, like lodash. Rust has iterators. In all these libraries, you start with a collection (or combination of collections), and massage the data from one form to another. To those unfamiliar, these filter/map/reduce operations seem foreign. For those familiar, writing the code any other way, a.k.a., with loops and branches, seems like unbearable torture. Converting a long-winded loop into a cute stream of operations is almost an art form.

In most of these languages, filter/map/reduce is achieved through simple method chaining. It's easy for these expressions to become an unreadable mess - a foot gun for enthusiastic, ambitious developers. This "art form" becomes a justification for unspeakable horrors. Method chaining is the extent of filter/map/reduce in most languages. Occasionally, the language will also support built-in tuple syntax, some sort of anonymous types, and pattern matching, in which case the verbosity can be kept to a minimum. In other languages, verbose `Pair` and `Tuple` classes start sprouting up everywhere.