---
layout: post
title: "GoLang Nugget - October 21, 2024"
date: 2024-10-21
categories: [golang]
excerpt_separator: <!--more-->
---
Welcome to this week's edition of **GoLang Nugget**, your go-to source for the latest insights and tips in the Go programming world!

This week, we dive into the transformative power of Test-Driven Development (TDD) in Go, especially when paired with MongoDB. Discover how TDD can optimize your code, clarify requirements, and enhance productivity by catching bugs early.

We also explore the world of concurrency with a focus on wait groups, a handy tool for managing goroutines. Learn how to use wait groups effectively to synchronize tasks and improve your code's efficiency.

For those looking to refine their Go skills, we share insights from seasoned developers on habits to drop for cleaner, more robust code. Plus, we discuss dynamic worker scaling to handle traffic spikes efficiently, ensuring your applications remain responsive under load.

Benchmarking enthusiasts will appreciate our coverage of the revamped benchstat tool, which now offers more detailed analysis capabilities. And if you're curious about code generation, we provide a quick guide to getting started with Go code generators.

Concurrency remains a hot topic, with a practical guide on synchronization techniques and patterns to avoid common pitfalls like deadlocks and data races. We also highlight common mistakes to avoid when learning Go, ensuring you adopt idiomatic practices from the start.

For those interested in distributed systems, we delve into implementing a key/value database using the Raft consensus algorithm, showcasing how to maintain strong consistency across a cluster.

Finally, we cover the essentials of mocking in Go with gomock, a powerful tool for isolating dependencies in your tests, and explore the nuances of the `defer` statement in Go, including its performance implications.

Stay tuned for more insights and happy coding!
<!--more-->
### [Test-Driven Development (TDD) in Go with MongoDB: A Practical Guide](https://medium.com/@robertbenyamino/test-driven-development-tdd-in-go-with-mongodb-a-practical-guide-d3a3f4233ac1?source=rss------golang-5)

Imagine constantly battling bugs in your backend code, or your code works in development but breaks in production. Enter Test-Driven Development (TDD), a methodology that flips traditional coding on its head by writing tests before the actual code. Here's why TDD is a game-changer:

1. **Code Optimization**: Designing tests first means you plan your code's API upfront, leading to cleaner and more efficient code.
2. **Clearer Requirements**: Writing tests forces you to deeply understand what you're building, clarifying requirements and identifying gaps.
3. **Easier Feature Addition**: A robust test suite makes adding features less risky, as tests catch any breakages.
4. **Higher Test Coverage**: TDD naturally results in comprehensive test coverage, catching bugs early.
5. **Enhanced Productivity**: Though initially slower, TDD reduces debugging time and prevents regression bugs, speeding up development.

The TDD cycle is simple: **Red-Green-Refactor**. Write a failing test (Red), make it pass with minimal code (Green), then refactor for quality (Refactor).

For practical application, consider building a survey API with Go and MongoDB using TDD. Start by defining your data model, set up a test environment with Docker, and write tests for your repository. Implement the repository to pass these tests, and run them to ensure everything works.

Here's a snippet to kickstart your data model:

```go
package models

import (
    "time"
    "go.mongodb.org/mongo-driver/bson/primitive"
)

type Survey struct {
    ID          primitive.ObjectID `bson:"_id,omitempty" json:"id"`
    Title       string             `bson:"title" json:"title"`
    Description string             `bson:"description" json:"description"`
    StartDate   time.Time          `bson:"start_date" json:"start_date"`
    EndDate     time.Time          `bson:"end_date" json:"end_date"`
    IsPublished bool               `bson:"is_published" json:"is_published"`
    Year        int                `bson:"year" json:"year"`
}
```

Remember, TDD isn't about perfect code on the first try; it's about creating a safety net for experimentation and improvement. Give it a shot in your next project and see how it transforms your development process!

[Read more...](https://medium.com/@robertbenyamino/test-driven-development-tdd-in-go-with-mongodb-a-practical-guide-d3a3f4233ac1?source=rss------golang-5)
---

### [Gist of Go: Wait groups](https://antonz.org/go-concurrency/wait-groups/)

This chapter from a book on Go concurrency focuses on channels and wait groups as tools for managing goroutines. Channels are primarily used for data transfer, synchronization, and cancellation between goroutines. However, for synchronization, wait groups are often more suitable. A wait group allows you to wait for multiple goroutines to finish by using an internal counter that increments with `Add` and decrements with `Done`, blocking with `Wait` until the counter reaches zero. The chapter provides examples of using wait groups, including refactoring a `timeit` function to use a wait group instead of a done channel. It also discusses the importance of passing wait groups as pointers to avoid synchronization issues and suggests encapsulating synchronization logic to simplify client code. The chapter introduces a `ConcGroup` type for managing concurrent tasks and handling panics within goroutines. It emphasizes the need for panic recovery within the same goroutine and provides a solution for capturing panics in a concurrent group. The chapter concludes with a brief mention of upcoming topics on data races.

Here's a snippet of the refactored `timeit` function using a wait group:

```go
func timeit(nIter int, nWorkers int, fn func()) int {
	var wg sync.WaitGroup
	start := time.Now()

	wg.Add(nWorkers)
	for i := 0; i < nWorkers; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < nIter/nWorkers; j++ {
				fn()
			}
		}()
	}

	wg.Wait()
	return int(time.Since(start).Milliseconds())
}
```

[Read more...](https://antonz.org/go-concurrency/wait-groups/)

---

### [10 Things I Stopped Doing in GoLang Programming After Learning from the Pros](https://medium.com/go-rust-and-julia-programming/10-things-i-stopped-doing-in-golang-programming-after-learning-from-the-pros-e178088b4e7a?source=rss------golang-5)

Imagine leveling up your Go skills and ditching old habits. One major shift is moving away from using the empty interface (`interface{}`) for flexibility. While it seems convenient, it sacrifices type safety and readability. Instead, embrace generic functions and custom types to maintain clarity and ensure type safety. This approach not only makes your code more robust but also aligns with Go's strengths.

[Read more...](https://medium.com/go-rust-and-julia-programming/10-things-i-stopped-doing-in-golang-programming-after-learning-from-the-pros-e178088b4e7a?source=rss------golang-5)

---

### [Dynamic Worker Scaling in Go: Enhancing Performance and Efficiency](https://medium.com/@cembdci/dynamic-worker-scaling-in-go-enhancing-performance-and-efficiency-f57ad110babd?source=rss------golang-5)

In the bustling digital landscape, handling unpredictable traffic spikes is crucial for maintaining smooth application performance. For Go developers, mastering dynamic worker scaling is key. Go's concurrency model, with workers running in goroutines, allows efficient task management. The core idea is to dynamically adjust the number of workers based on demand, ensuring responsiveness without overloading resources. Here's a practical nugget: use channels for communication between workers and the main application, and implement a scaling mechanism that monitors load and adjusts worker count accordingly. This approach prevents bottlenecks and resource exhaustion, optimizing both performance and user satisfaction. Here's a snippet to illustrate dynamic scaling:

```go
func (s *Scaler) scale() {
    load := len(s.inCh)
    currentWorkers := s.workerManager.WorkerCount()
    if load > s.loadThreshold && currentWorkers < s.maxWorkers {
        log.Info().Msg("Scaling Up")
        newWorker := &Worker{
            Wg: s.workerManager.wg,
            Id: currentWorkers,
            ReqHandler: s.workerManager.reqHandler,
        }
        s.workerManager.AddWorker(newWorker)
    } else if float64(load) < float64(LoadThresholdScaleDownRatio)*float64(s.loadThreshold) && currentWorkers > s.minWorkers {
        log.Info().Msg("Scaling Down")
        s.workerManager.RemoveWorker()
    }
}
```

This snippet shows how to scale workers up or down based on the current load, ensuring efficient request handling.

[Read more...](https://medium.com/@cembdci/dynamic-worker-scaling-in-go-enhancing-performance-and-efficiency-f57ad110babd?source=rss------golang-5)

---

### [Leveraging benchstat Projections in Go Benchmark Analysis](https://bwplotka.dev/2024/go-microbenchmarks-benchstat/)

The article discusses Go's micro-benchmarking framework and highlights the benchstat tool, which allows for detailed comparisons of Go A/B benchmark results. In 2023, benchstat was revamped to include projections, filtering, and groupings, enhancing its capability to compare benchmarks across various dimensions. The traditional benchmarking flow involves running benchmarks on different code versions and using benchstat to analyze improvements or regressions. However, this method has drawbacks like difficulty in tracking changes and environmental inconsistencies. The article introduces a new flow enabled by benchstat's updates, allowing comparisons across sub-benchmarks or cases, which improves reproducibility and reliability. This method, while more complex and time-consuming, offers flexibility in analyzing different dimensions of benchmarks. The article suggests using a hybrid approach of both flows depending on the benchmarking goals and emphasizes the importance of following a specific naming format for sub-benchmarks to leverage benchstat's full potential. Here's a snippet of the benchmarking code:

```go
func BenchmarkEncode(b *testing.B) {
  for _, sampleCase := range sampleCases {
    b.Run(fmt.Sprintf("sample=%v", sampleCase.samples), func(b *testing.B) {
      for _, compr := range compressionCases {
        b.Run(fmt.Sprintf("compression=%v", compr.name()), func(b *testing.B) {
          for _, protoCase := range protoCases {
            b.Run(fmt.Sprintf("proto=%v", protoCase.name), func(b *testing.B) {
              for _, marshaller := range marshallers {
                b.Run(fmt.Sprintf("encoder=%v", marshaller.name()), func(b *testing.B) {
                  msg := protoCase.msgFromConfigFn(sampleCase.config)
                  b.ReportAllocs()
                  b.ResetTimer()
                  for i := 0; i < b.N; i++ {
                    out, err := marshaller.marshal(msg)
                    testutil.Ok(b, err)
                    out = compr.compress(out)
                    b.ReportMetric(float64(len(out)), "bytes/message")
                  }
                })
              }
            })
          }
        })
      }
    })
  }
}
```

[Read more...](https://bwplotka.dev/2024/go-microbenchmarks-benchstat/)

---

### [A taste of Go code generator magic: a quick guide to getting started](https://evilmartians.com/chronicles/a-taste-of-go-code-generator-magic-a-quick-guide-to-getting-started)

The article discusses the challenges of metaprogramming and code generation in Go, highlighting the lack of resources available for creating Go code generators. It introduces a simple program that generates wrapper functions for type methods, serving as a starting point for building custom Go code generators. Key tools include `golang.org/x/tools/go/packages` for accessing package metadata, `go/types` for type metadata, and `github.com/dave/jennifer/jen` for code generation. The author shares their motivation for creating a generator to automate repetitive boilerplate code, specifically for managing a global database connection in a daemon app. The process involves loading package metadata, generating functions with proxy calls, handling imports, and saving the generated code with a `_gen.go` suffix. A sample code snippet demonstrates using the `jen` package to generate a simple function:

```go
package main

import (
	"log"
	"github.com/dave/jennifer/jen"
)

func main() {
	f := jen.NewFile("main")
	f.PackageComment("Code generated by generator, DO NOT EDIT.")
	f.Func().Id("Hello").Block(
		jen.Qual("fmt", "Println").Call(jen.Lit("Hello from generated code!")),
	)
	if err := f.Save("main_gen.go"); err != nil {
		log.Fatal(err)
	}
}
```

The article encourages exploring the `jen` and `go/types` packages for further capabilities in Go code generation.

[Read more...](https://evilmartians.com/chronicles/a-taste-of-go-code-generator-magic-a-quick-guide-to-getting-started)

---

### [Practical Concurrency Guide in Go](https://github.com/luk4z7/go-concurrency-guide)

The Go Concurrency Guide provides a comprehensive overview of concurrency patterns and synchronization techniques in Go, drawing from resources like "Go Concurrency in Go" and "Go Programming Language." Key topics include race conditions, data races, and memory access synchronization using primitives like Mutex, WaitGroup, RWMutex, Cond, and Pool. It addresses concurrency issues such as deadlocks, livelocks, and starvation, and explores the use of channels for communication between goroutines. The guide also covers concurrency patterns like confinement, cancellation, OR channels, error handling, pipelines, fan-in and fan-out, and the use of context for managing goroutine lifecycles. Additionally, it discusses Go's scheduler runtime, which uses a work-stealing strategy to efficiently manage goroutines across OS threads. The guide emphasizes best practices, such as using channels for communication rather than shared memory, and provides code snippets to illustrate concepts. Here's a simple example of using a Mutex for synchronization:

```go
type Counter struct {
    mu sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}
```

This guide is a valuable resource for understanding and implementing concurrency in Go, ensuring efficient and safe concurrent programming.

[Read more...](https://github.com/luk4z7/go-concurrency-guide)

---

### [10 Common Mistakes To Avoid When Learning GoLang Programming](https://levelup.gitconnected.com/10-common-mistakes-to-avoid-when-learning-golang-programming-a04d34bac069?source=rss------golang-5)

Beginners often mistakenly apply concepts from other languages like Python or Java to Go, which has its own idiomatic style emphasizing simplicity and readability. In Go, it's important to avoid unnecessary complexity, such as using `fmt.Sprintf` inside `fmt.Println`. Instead, directly using `fmt.Printf` is more idiomatic. Here's a more idiomatic version of the code:

```go
package main

import "fmt"

func main() {
    for i := 0; i < 10; i++ {
        if i%2 == 0 {
            fmt.Printf("Number %d is even\n", i)
        }
    }
}
```

[Read more...](https://levelup.gitconnected.com/10-common-mistakes-to-avoid-when-learning-golang-programming-a04d34bac069?source=rss------golang-5)

---

### [Building a Custom grep-Like Pattern Matcher in Go](https://medium.com/@AnnanyaPandey/building-a-custom-grep-like-pattern-matcher-in-go-40825fb9c98d?source=rss------golang-5)

The content describes building a custom pattern matcher in Go, similar to the grep command, to understand the mechanics of pattern matching. Pattern matching is crucial for searching and extracting information from text based on specific criteria, with applications in data validation, syntax parsing, and text processing. The matcher supports regular expressions, including special characters like `*, +, ?, []`, and `^`, and handles anchoring patterns at the start (`^`) or end (`$`) of a line. The core function, `MatchLine`, checks if any part of a line matches a given pattern, handling escape sequences (`\d, \w`), alternations (e.g., (`cat|dog`)), character classes (`[abc], [^abc]`), and quantifiers (`+, ?`). The implementation provides insights into regular expression engines, offering a flexible approach to pattern matching beyond Go's built-in regexp package. Here's a snippet of the `MatchLine` function:

```go
func MatchLine(line []byte, pattern string) (bool, error) {
    if utf8.RuneCountInString(pattern) == 0 {
        return false, fmt.Errorf("unsupported pattern: %q", pattern)
    }
    if pattern[0] == '^' && pattern[len(pattern)-1] == '$' {
        return matchExact(line, pattern[1:len(pattern)-1]), nil
    }
    if pattern[0] == '^' {
        return matchFromPosition(line, pattern[1:]), nil
    }
    if pattern[len(pattern)-1] == '$' {
        return matchFromEnd(line, pattern[0:len(pattern)-1]), nil
    }
    for i := 0; i < len(line); i++ {
        if strings.HasPrefix(pattern, "[^") && !matchFromPosition(line[i:], pattern) {
            return false, nil
        }
        if matchFromPosition(line[i:], pattern) {
            return true, nil
        }
    }
    return false, nil
}
```

[Read more...](https://medium.com/@AnnanyaPandey/building-a-custom-grep-like-pattern-matcher-in-go-40825fb9c98d?source=rss------golang-5)

---

### [Mastering Mocking in Go: Comprehensive Guide to gomock with Practical Examples](https://towardsdev.com/mastering-mocking-in-go-comprehensive-guide-to-gomock-with-practical-examples-e12c1773842f?source=rss------golang-5)

Imagine diving into Go unit testing and discovering the gomock library from go.uber.org/mock/gomock. This tool is a game-changer for creating mock objects, which are essentially stand-ins for real dependencies in your code. Why bother with mocks? They let you test your code in isolation, free from the unpredictability of external services or databases. gomock shines by auto-generating these mock objects from interfaces, saving you from writing tedious mock code manually.

To get started, install gomock and its mockgen utility with:
```bash
go get -u go.uber.org/mock/gomock
go install github.com/golang/mock/mockgen@v1.6.0
```

Generate mocks using:
```bash
mockgen -source=internal/repository.go -destination=mocks/repository_mock.go -package=mocks
```

In your tests, create a controller with `gomock.NewController(t)` to manage mock lifecycles. Use `EXPECT()` to define expected method calls and their return values. For example, simulate a successful user retrieval:
```go
mockRepo.EXPECT().
    GetUserByID("1").
    Return(&User{ID: "1", Name: "Alex"}, nil)
```

gomock also supports verifying call order with `gomock.InOrder` and handling asynchronous code using `sync.WaitGroup`. For more complex scenarios, use `Do()` to execute custom logic during method calls.

Integrate with real databases using testcontainer-go for a full-fledged testing setup. And don't forget table-driven tests to efficiently cover multiple scenarios with minimal code changes.

In essence, gomock is your go-to for making Go unit tests robust and reliable by isolating dependencies and ensuring your code behaves as expected.

[Read more...](https://towardsdev.com/mastering-mocking-in-go-comprehensive-guide-to-gomock-with-practical-examples-e12c1773842f?source=rss------golang-5)

---

### [Golang Defer: From Basic To Traps](https://victoriametrics.com/blog/defer-in-go/)

In Go, the `defer` statement is used to delay function execution until the surrounding function finishes. There are three types of `defer` in Go 1.22: open-coded, heap-allocated, and stack-allocated, each with different performance implications. `Defer` is useful for cleanup tasks like closing files or database connections. Deferred functions execute in a last-in-first-out order, and only those in the current function are executed unless a panic occurs, which triggers all deferred functions in the current goroutine. The `recover` function can handle panics within deferred functions. A common mistake is using `recover` incorrectly or trying to catch a panic from another goroutine. `Defer` captures values at the time of scheduling, which can lead to unexpected results if variables change later. This can be fixed using closures or pointers. Errors from deferred functions can be handled by using named return values. `Defer` objects are either heap or stack-allocated, with heap allocation being less efficient. Go optimizes `defer` with open-coded defers, which improve performance but have limitations. Open-coded defers are not applied if there are more than 8 `defer` statements or if the function has heap-allocated defers. The article is written by Phuong Le, a software engineer at VictoriaMetrics, and invites readers to reach out with questions or check out VictoriaMetrics for monitoring services.

[Read more...](https://victoriametrics.com/blog/defer-in-go/)

---

### [Go 1.23: Interactive release notes](https://antonz.org/go-1-23/)

Go 1.23 introduces several new features and improvements, focusing heavily on iterators, which now provide a standard way to work with sequences of values. Iterators can be used in for-range loops, and new iterator types like Seq and Seq2 are introduced for concise definitions. The release also enhances timer behavior, addressing garbage collection and stop/reset issues, ensuring timers are eligible for garbage collection immediately and fixing the Reset method. The unique package offers value canonicalization, reducing memory usage by deduplicating values. The http package sees improvements in cookie handling, including parsing and managing cookies with new attributes. The os package introduces the CopyFS function for recursive file and directory copying. Additionally, the slices package gains a Repeat function, and atomic operations now support bitwise AND/OR. Tooling updates include telemetry collection, new Go command flags, and improved runtime error messages. These changes aim to enhance performance, memory efficiency, and developer experience. Here's a snippet demonstrating the new iterator usage:

```go
// go 1.23
for key, val := range m.Range {
	fmt.Println(key, val)
}

// Example of using Seq type
func Reversed[V any](s []V) iter.Seq[V] {
	return func(yield func(V) bool) {
		for i := len(s) - 1; i >= 0; i-- {
			if !yield(s[i]) {
				return
			}
		}
	}
}
```

[Read more...](https://antonz.org/go-1-23/)

---

### [Implementing Raft: Part 4 - Key/Value Database](https://eli.thegreenplace.net/2024/implementing-raft-part-4-keyvalue-database/)

In Part 4 of the series on the Raft distributed consensus algorithm, the focus is on implementing a replicated key/value database (KV DB) with strong consistency using the Raft module. The KV DB functions as a state machine supporting operations like PUT, GET, and CAS (compare-and-swap), ensuring linearizable and serializable semantics. The system architecture includes a cluster of KV DB services, each containing a Raft Consensus Module and a data store, with communication via RPCs and a REST API for client interaction. The KV service processes commands through the Raft log to maintain consistency, even for read-only operations like GET, preventing stale reads. The KV client library facilitates interaction with the service, handling leader election and retries. The implementation prioritizes consistency over availability, adhering to the CAP theorem. Future work will address challenges like handling retries to avoid non-linearizable behavior, as discussed in the Raft paper. Here's a snippet illustrating the command structure:

```go
type Command struct {
  Kind CommandKind
  Key, Value string
  CompareValue string
  ResultValue string
  ResultFound bool
  Id int
}

type CommandKind int

const (
  CommandInvalid CommandKind = iota
  CommandGet
  CommandPut
  CommandCAS
)
```

[Read more...](https://eli.thegreenplace.net/2024/implementing-raft-part-4-keyvalue-database/)

---

### [Mastering Go Performance: 8 Essential Tips to Boost Your Code Efficiency](https://blog.stackademic.com/mastering-go-performance-8-essential-tips-to-boost-your-code-efficiency-4619bf723e90?source=rss------golang-5)

The post emphasizes the importance of performance optimization in Go programming, highlighting that efficient coding extends beyond syntax knowledge to include practical experience, creativity, and continuous learning. The author, with over five years of experience, shares eight key tips for writing high-performance Go code, focusing on the effective use of goroutines for concurrent programming. A critical point is to avoid excessive goroutine spawning, which can lead to resource exhaustion and degraded performance. Instead, the author suggests using a worker pool to manage the number of concurrent goroutines efficiently. An example implementation of a worker pool is provided, demonstrating how to limit concurrent goroutines and process jobs effectively.

```go
func workerPool(numWorkers int, jobs <-chan int, results chan<- int) {
    for i := 0; i < numWorkers; i++ {
        go worker(jobs, results)
    }
}

func worker(jobs <-chan int, results chan<- int) {
    for job := range jobs {
        results <- process(job)
    }
}

func process(job int) int {
    time.Sleep(time.Millisecond * ...)
    return job // Example processing
}
```

[Read more...](https://blog.stackademic.com/mastering-go-performance-8-essential-tips-to-boost-your-code-efficiency-4619bf723e90?source=rss------golang-5)