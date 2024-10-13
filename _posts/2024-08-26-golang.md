---
layout: post
title: "GoLang Nugget - August 26, 2024"
date: 2024-08-26
categories: [golang]
excerpt_separator: <!--more-->
---
Welcome to this week's edition of **GoLang Nugget**, your go-to source for the latest in the Go programming world!

This week, we dive into the exciting release of Go 1.23, which brings new features like iterator functions for "for-range" loops and preview support for generic type aliases. We also explore the enhancements in Go's standard library and tool improvements, including Go telemetry and new command conveniences.

Understanding channels is crucial for effective concurrency in Go. We break down three ways to think about channels, highlighting their role in Go's concurrency model and how they interact with other primitives.

Performance is key, and we look at how sentinel errors and `errors.Is()` can slow down your code. Learn about more efficient error handling strategies to keep your Go applications running smoothly.

The introduction of range over function types in Go 1.23 simplifies iterating over custom containers, making it easier to write generic functions. We also touch on the improvements in secure randomness with Go 1.22, enhancing security by default.

Lastly, we explore writing generic collection types in Go, sharing insights and tips for implementing sortable sets using generics.

Stay tuned for more insights and updates in the world of Go!
<!--more-->
### [Go 1.23 is released](https://go.dev/blog/go1.23)

Go 1.23 has been released with several enhancements and new features. Key language changes include support for iterator functions in "for-range" loops and preview support for generic type aliases. Tool improvements feature Go telemetry for usage statistics, new go command conveniences like `go env -changed` and `go mod tidy -diff`, and updates to `go vet`. The standard library sees the addition of three new packages: iter, structs, and unique, along with improvements to time.Timer and time.Ticker. Experimental support for OpenBSD on 64-bit RISC-V and performance enhancements with profile-guided optimization are also included. Users are encouraged to read the release notes for detailed information and to report any issues. To upgrade, use `go get toolchain@go1.23.0` or `go get go@1.23.0`.

[Read more...](https://go.dev/blog/go1.23)

---

### [Three Ways To Think About Channels](https://dolthub.com/blog/2024-06-21-channel-three-ways/)

Understanding Golang channels involves recognizing them as locked, buffered queues with an API and implementation akin to a queue. Channels are part of a broader concurrency ecosystem that includes error groups, goroutines, and other primitives. The Go runtime manages channels efficiently, but they can add overhead compared to mutexes or atomics. Channels operate by parking and signaling senders and receivers to avoid unnecessary spinning. Key operations include sending (`ch <- w`), receiving (`w, ok := <-ch`), and using `select` statements to handle blocking scenarios. Channels are integral to Go's concurrency model, often used with goroutines, timeouts, tickers, wait groups, and error groups to manage concurrent tasks and graceful shutdowns. The Go runtime's efficient context switching and embedded channel support enable high-performance concurrent programming. Channels abstract complexity, making it easier to write concurrent code, but understanding their interaction with other concurrency primitives is crucial for effective use.

[Read more...](https://dolthub.com/blog/2024-06-21-channel-three-ways/)

---

### [Sentinel errors and errors.Is() slow your code down by 500%](https://dolthub.com/blog/2024-05-31-benchmarking-go-error-handling/)

The blog post benchmarks various error handling strategies in Go, revealing significant performance differences. The fastest method is using a boolean to indicate the presence of a value, while the slowest is using panic. Sentinel errors checked with `errors.Is()` are notably slower, especially when errors are wrapped. The post advises checking for non-nil errors before using `errors.Is()` to mitigate performance hits. It also critiques the use of sentinel errors for control flow, suggesting boolean returns as a more efficient alternative. The author emphasizes measuring performance impacts in real-world scenarios and avoiding sentinel errors for both performance and design reasons.

```go
func BenchmarkNotFoundBool(b *testing.B) {
	var bs boolStore
	for i := 0; i < b.N; i++ {
		val, found, err := bs.GetValue(i < 0)
		if err != nil {
			b.Fatal(err)
		} else if found {
			b.Fatal("expected not found")
		}
		if val != nil {
			b.Fatal("expected nil")
		}
	}
}
```

[Read more...](https://dolthub.com/blog/2024-05-31-benchmarking-go-error-handling/)

---

### [Range Over Function Types](https://go.dev/blog/range-functions)

The Go 1.23 release introduces a new feature: range over function types, which simplifies iterating over user-defined containers like sets. Previously, Go lacked a standardized way to loop through custom containers, leading to inconsistent methods. The new feature allows for using `for/range` with functions that take a single argument, a "yield" function, to iterate over elements. This standardizes iteration, making it easier to write generic functions. The blog explains two types of iterators: push (standard) and pull. Push iterators call a yield function for each element, while pull iterators return the next element on each call. The new `iter` package in the standard library provides types and functions to support these iterators. Here's an example of using the new feature:

```go
// Set holds a set of elements.
type Set[E comparable] struct {
    m map[E]struct{}
}

// All is an iterator over the elements of s.
func (s *Set[E]) All() iter.Seq[E] {
    return func(yield func(E) bool) {
        for v := range s.m {
            if !yield(v) {
                return
            }
        }
    }
}

// PrintAllElements prints all elements in the set.
func PrintAllElements[E comparable](s *Set[E]) {
    for v := range s.All() {
        fmt.Println(v)
    }
}
```

This feature enhances Go's ecosystem by providing a consistent and efficient way to iterate over various container types.

[Read more...](https://go.dev/blog/range-functions)

---

### [Secure Randomness in Go 1.22](https://go.dev/blog/chacha8rand)

Go 1.22 introduces improvements in secure randomness, enhancing the `math/rand` package by using a cryptographic random number source. This update reduces the risk when developers mistakenly use `math/rand` instead of `crypto/rand`. The new generator, ChaCha8Rand, is based on the ChaCha8 stream cipher, offering better security and unpredictability compared to the old linear-feedback shift register. ChaCha8Rand uses a 32-byte seed, generates 64-byte blocks of random data, and includes a mechanism for forward secrecy. Although slightly slower, the trade-off is worth it for the added security. This update makes Go programs more secure by default, even if developers accidentally use `math/rand` for cryptographic purposes. Here's a quick code snippet to show how the new generator works:

```go
package main

import (
    "math/rand/v2"
    "fmt"
)

func main() {
    r := rand.ChaCha8()
    fmt.Println(r.Uint64())
}
```

This snippet uses the new `ChaCha8Rand` generator to produce a secure random number. In a nutshell, Go 1.22 makes randomness more secure and reduces the risk of common mistakes, all while maintaining good performance.

[Read more...](https://go.dev/blog/chacha8rand)

---

### [Writing generic collection types in Go: the missing documentation](https://dolthub.com/blog/2024-07-01-golang-generic-collections/)

Go generics, introduced in Go 1.18, have not been widely used in Dolt's codebase despite its extensive use of Go. The author faced challenges implementing a generic collection, specifically a sortable Set, and documented the trial-and-error process. Initial attempts using two generic type parameters and self-referential type definitions without the `comparable` constraint failed due to compilation errors. The breakthrough came with understanding that type constraints can be declared as interfaces, combining custom constraints with built-in ones like `comparable`. This allowed the creation of a fully functional generic collection without type assertions. Key takeaways include avoiding multiple type parameters for single-type collections, ensuring type constraints include `comparable` for map keys and comparisons, and using type sets for constraints. The author emphasizes the importance of reading the language specification for deeper understanding. Here’s a snippet of the final solution:

```go
type Sortable[T comparable] interface {
	Less(member T) bool
}

type SortableSet[T interface{Sortable[T]; comparable}] interface {
	Add(member T)
	Size() int
	Contains(member T) bool
	Sorted() []T
}

type MapSet[T interface{Sortable[T]; comparable}] struct {
	members map[T]struct{}
}

func NewMapSet[T interface{Sortable[T]; comparable}]() SortableSet[T] {
	return MapSet[T]{
		members: make(map[T]struct{}),
	}
}
```

[Read more...](https://dolthub.com/blog/2024-07-01-golang-generic-collections/)

---

### [Go range iterators demystified](https://dolthub.com/blog/2024-07-12-golang-range-iters-demystified/)

Go 1.23 introduces a new feature where you can use the `range` keyword to iterate over custom collection types, not just slices and maps. This is done using special iterator functions. Range iterators are functions that let you use `range` with custom collections, coming in three flavors: `func(func() bool)`, `func(func(K) bool)`, and `func(func(K, V) bool)`. Here's a basic example:

```go
func iter1(yield func(i int) bool) {
    for i := range 3 {
        if !yield(i) {
            return
        }
    }
}

func testFuncRange1() {
    for i := range iter1 {
        fmt.Println("iter1", i)
    }
}
```

This prints:
```
iter1 0
iter1 1
iter1 2
```

The yield function runs the loop body, and if you don't check its return value, your program might panic if there's a `break`. You can define methods on your custom types to use `range`, such as iterating over prime numbers in a slice. Error handling during iteration can be managed using a two-parameter iterator. Sentinel errors can be handled more gracefully by wrapping traditional iterators. Pull and Pull2 functions convert range iterators back into traditional iterators for compatibility. In essence, Go 1.23's range iterators make it super flexible to iterate over any collection type, not just slices and maps, opening up a lot of possibilities for custom data structures.

[Read more...](https://dolthub.com/blog/2024-07-12-golang-range-iters-demystified)