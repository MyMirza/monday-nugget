---
layout: post
title: "Golang Nugget - November 18, 2024"
date: 2024-11-18
categories: [golang]
excerpt_separator: <!--more-->
---
Hey folks! Welcome to this week’s **Golang Nugget**, where I’ve rounded up some of the most interesting insights and tips from the Go community. I didn’t come up with these myself, but they’re too good not to share.

Here’s what’s on the menu this week:

- A fascinating dive into how Go’s scheduler handles goroutines and why you often see unexpected execution orders.
- A breakdown of why plain string concatenation can outperform fmt.Sprintf—and when you should care.
- A fresh take on the Functional Option Pattern that simplifies configuration and reduces boilerplate.
- GoooQo, a dynamic query language for Go, makes writing database queries feel much more intuitive.
- Practical ways to use functional programming patterns in Go for cleaner, more maintainable code.
- There’s also a deep dive into preventing state corruption during panics and tips on creating smaller, safer Docker images using multi-stage builds.

These nuggets of knowledge are straight from some brilliant minds in the Go community—take a look and level up your Go skills!
<!--more-->

{% include mailerlite_golang_nugget_postpage.html %}

### [Some details about Go Scheduler](https://medium.com/@e.kubyshin/some-details-about-go-scheduler-daaa31642868?source=rss------golang-5)

When you run concurrent code in Go with GOMAXPROCS(1), the scheduler follows a specific pattern that might seem counterintuitive at first. Here's what's actually happening:

```go
func main() {
    runtime.GOMAXPROCS(1)
    var wg sync.WaitGroup
    wg.Add(5)
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Println(i)
            wg.Done()
        }()
    }
    wg.Wait()
}
```

The key points about Go's scheduling behaviour:

1. When creating new goroutines with `go func()`, they're placed in a special `runnext` slot rather than the regular run queue.
2. The scheduler prioritises the `runnext` slot over the regular queue.
3. With GOMAXPROCS(1), there's only one processor (P) handling goroutines.
4. Each new goroutine overwrites the previous one in `runnext`.

That's why you'll consistently see the output `4 0 1 2 3` - the last goroutine (i=4) runs first, then the others follow in FIFO order from the regular queue.

This behaviour is by design to optimise communication patterns between goroutines, reducing scheduling latency when goroutines are ready to run.

Want to experiment? Try changing GOMAXPROCS to 2 or more and see how the output becomes non-deterministic.

[Read more...](https://medium.com/@e.kubyshin/some-details-about-go-scheduler-daaa31642868?source=rss------golang-5)

---

### [fmt.Sprintf vs String Concat](https://www.dolthub.com/blog/2024-11-08-sprintf-vs-concat/)

String concatenation in Go offers better performance compared to fmt.Sprintf. While fmt.Sprintf looks cleaner, it comes with some overhead that can impact performance, especially in high-throughput scenarios.

Here's what makes fmt.Sprintf slower:
- Uses interface{} type which adds 8 bytes of overhead.
- Requires allocation of formatting state objects and buffers.
- Involves complex code paths for handling various format types.
- Needs to parse format strings and handle multiple cases.

The string concatenation operator (+) is faster because:

```go
// Instead of
getFieldName := fmt.Sprintf("%s.%s", p.table, p.name)
// Use
getFieldName := p.table + "." + p.name
```

The + operator gets converted to runtime.stringconcat which:
- Pre-calculates total string length.
- Uses stack allocation for small strings (up to 32 bytes).
- Performs simple copy operations in a tight loop.
- Avoids complex parsing and type checking.

Real-world impact? Well, in Dolt's case, switching from fmt.Sprintf to string concatenation yielded:
- 3-5% improvement in query performance.
- 2% reduction in memory allocations.
- 6% fewer allocations per operation.

The performance gain might seem small, but it's quite significant for a mature system. The key takeaway is that while fmt.Sprintf is convenient for complex formatting, simple string concatenation is more efficient for basic string joining operations.

Remember though, these optimisations matter most in hot paths or high-performance scenarios. For regular application code where string operations aren't frequent, readability might be more important than these micro-optimisations.

[Read more...](https://www.dolthub.com/blog/2024-11-08-sprintf-vs-concat/)

---

### [Next-Gen Functional Options in Golang: Effortless Configuration, Zero Boilerplate.](https://medium.com/@dmkolesnikov/next-gen-functional-options-in-golang-effortless-configuration-zero-boilerplate-8bb47b1872fc?source=rss------golang-5)

The Functional Option Pattern in Go is a clever way to handle configuration, but it's always had some annoying quirks. Well, here's the good news: there's a brilliant solution that makes it much more manageable!

The pattern traditionally looks like this:

```go
func WithOptA(x string) Option {
  return func(c *Config) error {
    c.opt = x
    return nil
  }
}
```

But with the OPTS library, you can simplify it to:

```go
var WithOptA = opts.ForType[Config, string]()
```

Here's what makes this approach brilliant:

- Eliminates boilerplate code - just one line per option.
- Supports both simple setters and complex validation logic.
- Handles mandatory parameters elegantly through `opts.Required`.
- Works seamlessly with dependency chains.

For mandatory parameters, you've got two solid choices:
1. Single required param? Just put it in the constructor.
2. Multiple required params? Use a config struct with public fields for required and private for optional.

Quick tip: Use the Option Pattern when you need flexibility and complex configurations (think HTTP clients). Stick with simple structs for straightforward, unlikely-to-change configs.

The library makes the Functional Option Pattern actually practical for everyday use. It's type-safe, clean, and maintains all the pattern's benefits without the traditional headaches. Give it a shot - your future self will thank you!

Check out the library at [github.com/fogfish/opts](github.com/fogfish/opts).

[Read more...](https://medium.com/@dmkolesnikov/next-gen-functional-options-in-golang-effortless-configuration-zero-boilerplate-8bb47b1872fc?source=rss------golang-5)

---

### [GoooQo Is a Dynamic Query Language Implemented in Golang for CRUD](https://github.com/doytowin/goooqo)

GoooQo introduces a fresh approach to database operations through Object-Query Mapping (OQM), which is quite different from traditional ORM. The core idea is brilliantly simple - it maps object field combinations to query conditions, making dynamic queries much more manageable.

The framework revolves around three key object types:
- Entity Objects handle static CRUD mappings like table and column names.
- Query Objects manage dynamic parts like filters and sorting.
- View Objects deal with complex query components like joins and grouping.

Here's how you get started. First, set up your database connection:

```go
db, _ := sql.Open("sqlite3", "./test.db")
tm := rdb.NewTransactionManager(db)
```

Then define your entity and query structures:

```go
type UserEntity struct {
    Int64Id
    Name    *string `json:"name"`
    Score   *int    `json:"score"`
    Memo    *string `json:"memo"`
    Deleted *bool   `json:"deleted"`
}

type UserQuery struct {
    PageQuery
    ScoreLt  *int
    MemoLike *string
    // ... other query fields
}
```

The real magic happens in the query execution. You can build complex queries simply by setting object properties:

```go
userQuery := UserQuery{
    PageQuery: PageQuery{PageSize: P(20), Sort: P("id,desc")}, 
    MemoLike: P("Great")
}
users, err := userDataAccess.Query(ctx, userQuery)
```

Transaction management is straightforward too. You can either manually control transactions or use a callback approach:

```go
err := tm.SubmitTransaction(ctx, func(tc TransactionContext) error {
    // transaction logic here
    return nil
})
```

What makes this framework particularly clever is how it handles dynamic query conditions - instead of dealing with complex query builders, you just set properties on your query object, and the framework takes care of the rest.

[Read more...](https://github.com/doytowin/goooqo)

---

### [Functional programming in Go](https://bitfieldconsulting.com/posts/functional)

Functional programming in Go boils down to three powerful operations that can replace traditional loops and make your code more elegant: Map, Filter, and Reduce. Here's what makes them brilliant:

Map transforms each element in a slice using a function:

```go
s := []string{"a", "b", "c"}
result := Map(s, strings.ToUpper) // Returns [A B C]
```

Filter selects elements that match specific criteria:

```go
s := []int{1, 2, 3, 4}
result := Filter(s, IsEven[int]) // Returns [2 4]
```

Reduce combines all elements into a single value:

```go
s := []int{1, 2, 3, 4}
sum := Reduce(s, 0, func(cur, next int) int {
    return cur + next
}) // Returns 10
```

These operations leverage Go's first-class functions - meaning functions can be passed as arguments or returned as values. While you could write traditional loops, this functional approach often leads to more readable and maintainable code.

The beauty lies in their composability - you can chain these operations together to solve complex problems with minimal code. Plus, they're generic, so they work with any data type that fits the constraints.

Want to level up your Go code? Try refactoring your next loop-heavy function using these functional programming patterns. You might be surprised at how much cleaner it becomes!

[Read more...](https://bitfieldconsulting.com/posts/functional)

---

### [Understanding Value and Pointer Receivers in Go Interfaces](https://afdz.medium.com/understanding-value-and-pointer-receivers-in-go-interfaces-e97a824fdded?source=rss------golang-5)

You know, it's actually quite fascinating how Go handles methods differently based on whether you use value or pointer receivers.

When you're working with methods in Go, you've got two ways to attach them to types: value receivers and pointer receivers. The difference might seem subtle, but it's crucial for how your code behaves.

Here's what you need to know about value receivers:

```go
type Counter struct {
    value int
}

func (c Counter) Increment() {
    c.value++ // This only modifies a copy!
}
```

With value receivers, you're working with a copy of your data. Any changes you make won't affect the original value - it's like having a photocopy to scribble on.

Now, pointer receivers are different:

```go
func (c *Counter) Increment() {
    c.value++ // This modifies the original!
}
```

When it comes to interfaces, there's an interesting twist. If you implement methods with value receivers, both the type and its pointer will satisfy the interface. But with pointer receivers, only the pointer type will do.

Here's a practical example:

```go
type Animal interface {
    Sound() string
}

type Dog struct{}
func (d Dog) Sound() string { return "Woof!" }    // Both Dog and *Dog work

type Cat struct{}
func (c *Cat) Sound() string { return "Meow!" }   // Only *Cat works
```

So, when should you use which? Well, use pointer receivers when you need to modify the receiver or when you're dealing with large structs to avoid copying. Value receivers are brilliant for small, immutable types or when you want both value and pointer types to satisfy an interface.

Remember to keep your receiver types consistent across methods for the same type - it'll make your code much clearer and easier to maintain.

[Read more...](https://afdz.medium.com/understanding-value-and-pointer-receivers-in-go-interfaces-e97a824fdded?source=rss------golang-5)

---

### [Go Exceptions for the Unconvinced](https://kristoff.it/blog/go-exceptions-unconvinced/)

Let's dig into a tricky aspect of Go's panic/recover mechanism that can lead to some nasty state corruption issues. The core problem appears around how panics interact with state modifications, particularly in concurrent code.

Here's the issue: When you're updating multiple related fields and a panic occurs mid-way, you can end up with partially updated state. This is especially problematic when the panic gets recovered higher up in the call stack, leaving your application running with corrupted data.

Consider this problematic pattern:

```go
foo.mutex.lock()
defer foo.mutex.unlock()

foo.a = doA() // Might panic!
foo.b = doB() // Never executes if doA panics
```

If doA() panics, foo.b never gets updated, leaving the object in an inconsistent state. This becomes particularly dangerous in scenarios like HTTP servers, which recover from panics by default to keep the server running.

The solution? Well, you'll want to use a transaction-like pattern:

```go
foo.mutex.lock()
defer foo.mutex.unlock()

var a = doA()
var b = doB()

// Atomic update
foo.a = a
foo.b = b
```

By computing all values before updating any state, we ensure atomic updates and maintain data consistency even if panics occur. This approach mirrors how you'd handle exceptions in languages like Java.

The key takeaway is that Go's panic mechanism, combined with defer's stack unwinding behaviour, requires defensive programming when modifying state. You've got to be especially careful in long-lived applications where panics might be recovered, as corrupted state can persist and cause issues down the line.

[Read more...](https://kristoff.it/blog/go-exceptions-unconvinced/)

---

### [How to Build Smaller Container Images: Docker Multi-Stage Builds](https://labs.iximiuz.com/tutorials/docker-multi-stage-builds)

Container images often end up bloated with unnecessary build-time dependencies that shouldn't be in production. Multi-stage builds solve this common Docker challenge.

The core concept is separating build and runtime environments while keeping everything in a single Dockerfile. Here's how it works:

```dockerfile
# Build stage
FROM golang:1.23 AS build
WORKDIR /app
COPY . .
RUN go build -o binary

# Runtime stage
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app/binary /app/binary
ENTRYPOINT ["/app/binary"]
```

Each stage serves a specific purpose - the first stage handles compilation with all necessary build tools, while the second stage contains just what's needed to run the application. The magic happens with the `COPY --from=build` instruction, which pulls artifacts from the build stage into the lean runtime image.

You can define multiple stages in a single Dockerfile and reference them using the `AS` keyword. The stages are built in order, and Docker is smart enough to:
- Skip unused stages.
- Run independent stages in parallel.
- Build all dependent stages automatically.

This approach works brilliantly across different tech stacks. For Node.js apps, you might have npm install and build in the first stage, then copy just the built assets to a slim runtime image. For Java applications, you'd use a JDK image for building and a JRE image for running.

The result? Significantly smaller production images with fewer potential vulnerabilities. You get the best of both worlds - a full-featured build environment and a minimal runtime container.

[Read more...](https://labs.iximiuz.com/tutorials/docker-multi-stage-builds)