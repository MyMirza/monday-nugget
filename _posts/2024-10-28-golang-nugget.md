---
layout: post
title: "Golang Nugget - October 28, 2024"
date: 2024-10-28
categories: [golang]
excerpt_separator: <!--more-->
redirect_to: https://golangnugget.com/p/golang-grpc-testing-concurrency-debugging-tips-28-10-2024
---
Welcome to this week’s edition of **Golang Nugget**, your go-to source for the latest insights and techniques in the Go programming world! This week, we’re covering:

Testing Tools for gRPC: Discover tools like the “tests-coverage-tool” for gRPC services to ensure thorough testing, and see how Mutexes can help prevent race conditions in concurrent programming.

Goroutines vs. Threads: Understand why Goroutines outperform traditional threads, making them ideal for scalable applications. Simplify setup with `sync.Once` and learn tips to write cleaner Go code.

Debugging and Task Management: Dive into debugging Go core dumps with Delve, and try out Go-Taskflow for managing complex task dependencies.

Happy coding!
<!--more-->

{% include mailerlite_golang_nugget_postpage.html %}

### [Test coverage visualization for gRPC services](https://medium.com/@filonov.nikitkaa/test-coverage-visualization-for-grpc-services-8d5f2e69aff5?source=rss------golang-5)

The article introduces the `tests-coverage-tool`, a Golang-based utility for measuring requirements coverage in gRPC services, focusing on proto contracts rather than deep business logic. It highlights the importance of ensuring all gRPC service methods and fields are covered by automated tests, especially as services grow and evolve.

The tool automatically gathers coverage data, requiring only proper configuration, and is designed to work with gRPC services, though the concept can be adapted for other protocols. The tool's architecture includes two projects: the main tool and a reporting submodule. It uses a gRPC interceptor to collect coverage data during test execution, saving results in JSON format.

The tool's configuration is managed via a YAML file, and integration into tests is straightforward, requiring minimal code changes. The tool generates detailed HTML and JSON reports, providing insights into coverage percentages, covered methods, and fields, including deprecated ones. It aids QA engineers in identifying coverage gaps and planning test coverage for new functionalities.

While it automates coverage measurement, it doesn't assess business logic, which requires manual evaluation. The tool is available on GitHub and is recommended for projects using gRPC.

Here's a snippet of the interceptor code:

```go
func CoverageInterceptor() grpc.UnaryClientInterceptor {
    return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
        invokerErr := invoker(ctx, method, req, reply, cc, opts...)
        result, err := buildCoverageResult(method, req, reply)
        if err != nil {
            log.Printf("Error building coverage result: %v", err)
            return invokerErr
        }
        toolConfig, err := config.NewConfig()
        if err != nil {
            log.Printf("Error building config: %v", err)
            return invokerErr
        }
        filename := fmt.Sprintf("%s.json", uuid.New().String())
        resultsDir := toolConfig.GetResultsDir()
        if err = utils.SaveJSONFile(result, resultsDir, filename); err != nil {
            log.Printf("Error saving coverage result: %v", err)
        }
        return invokerErr
    }
}
```

[Read more...](https://medium.com/@filonov.nikitkaa/test-coverage-visualization-for-grpc-services-8d5f2e69aff5?source=rss------golang-5)

---

### [Race Condition: Goroutines](https://medium.com/@sandeepshabd/race-condition-goroutines-4de0da07e7e1?source=rss------golang-5)

The post describes a race condition scenario in a banking system simulation using goroutines in Go, where two users attempt to withdraw funds from the same bank account concurrently. This race condition arises because both goroutines check the account balance simultaneously and proceed with withdrawals without synchronization, potentially leading to an incorrect final balance.

For instance, if the initial balance is \$1000, Goroutine 1 might withdraw \$700, and Goroutine 2 might also withdraw \$500 based on the same initial balance, resulting in a negative balance of \$-200. To resolve this issue, a `sync.Mutex` is introduced to ensure mutual exclusion, allowing only one goroutine to access and modify the balance at a time, thus preventing the race condition.

The critical section in the `withdraw` method is locked using the mutex, ensuring that Goroutine 2 waits for Goroutine 1 to complete its operation before proceeding.

```go
type BankAccount struct {
    balance int
    mu sync.Mutex
}

func (a *BankAccount) withdraw(amount int) {
    a.mu.Lock() // Lock to prevent race condition
    defer a.mu.Unlock()
    if a.balance >= amount {
        time.Sleep(time.Millisecond * 100)
        a.balance -= amount
        fmt.Printf("Successfully withdrew $%d, remaining balance: $%d\n", amount, a.balance)
    } else {
        fmt.Printf("Failed to withdraw $%d, insufficient balance. Current balance: $%d\n", amount, a.balance)
    }
}
```

[Read more...](https://medium.com/@sandeepshabd/race-condition-goroutines-4de0da07e7e1?source=rss------golang-5)

---

### [Goroutines vs. Threads: Why Go’s Lightweight Concurrency Model Outperforms System Threads](https://abubakardev0.medium.com/goroutines-vs-threads-why-gos-lightweight-concurrency-model-outperforms-system-threads-025a5c877bc9?source=rss------golang-5)

The post discusses the advantages of using Goroutines in Go for handling concurrency over traditional system threads. Goroutines are lightweight, memory-efficient, and managed by Go's runtime, making them faster and more efficient than system threads, which suffer from high memory consumption, expensive management, and context switching overhead.

The provided code example demonstrates launching 10,000 concurrent tasks using Goroutines, showcasing their ability to handle large workloads without overwhelming system resources. By setting `GOMAXPROCS(1)`, the example highlights how Goroutines can efficiently multiplex tasks onto a single OS thread. The use of `sync.WaitGroup` ensures all Goroutines complete before the program exits.

The post concludes that Goroutines are ideal for building scalable, efficient systems, offering significant performance benefits for modern applications like web servers and microservices.

Here's a snippet of the code:

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func worker(id int) {
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    runtime.GOMAXPROCS(1)
    var wg sync.WaitGroup
    for i := 1; i <= 10000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            worker(id)
        }(i)
    }
    wg.Wait()
    fmt.Println("All workers completed")
}
```

[Read more...](https://abubakardev0.medium.com/goroutines-vs-threads-why-gos-lightweight-concurrency-model-outperforms-system-threads-025a5c877bc9?source=rss------golang-5)

---

### [A Useful Pattern for Nil Channel Values in Go](https://dolthub.com/blog/2024-10-25-go-nil-channels-pattern/)

DoltHub is developing Dolt, a SQL database with Git-like version control features, using Go. The blog post discusses Go's nil channel behavior, which blocks indefinitely on send and receive operations, potentially causing deadlocks. However, this behavior is useful for selectively disabling branches in a `select` statement, enabling idiomatic patterns like optional functionality and state machines.

For instance, a channel can be optionally nil to enforce timeouts or coordinate sending values to multiple channels without additional synchronization primitives. The post provides examples, such as a `Debounce` function that manages input/output timing and a `Batch` function that transforms a channel into batches of a maximum size. These patterns simplify code by avoiding multiple select statements for different channel states.

Here's a snippet illustrating optional functionality with a timeout:

```go
var timeout chan time.Time
if enforceTimeout {
    timeout = time.After(10 * time.Second)
}
select {
case recv := <-recvCh:
    return recv, nil
case <-timeout:
    return nil, errors.New("timed out")
}
```

[Read more...](https://dolthub.com/blog/2024-10-25-go-nil-channels-pattern/)

---

### [Making Initialization Easy with Golang’s sync.Once](https://medium.com/@andhikaprbw/making-initialization-easy-with-golangs-sync-once-b02a73666497?source=rss------golang-5)

If you've ever faced race conditions while trying to initialize shared resources in Go, `sync.Once` is a lifesaver. It ensures a block of code runs only once, even when called from multiple goroutines, by using a mutex for synchronization. This is particularly useful for singleton initialization, resource management, configuration loading, and setting up event handlers.

Here's a simple example:

```go
package main

import (
    "fmt"
    "sync"
)

var once sync.Once
var config *Configuration

type Configuration struct {
    Setting string
}

func initConfig() {
    config = &Configuration{Setting: "Initialized"}
    fmt.Println("Configuration initialized.")
}

func GetConfig() *Configuration {
    once.Do(initConfig) // Only runs initConfig once!
    return config
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            cfg := GetConfig() // This will only initialize once
            fmt.Println(cfg.Setting)
        }()
    }
    wg.Wait() // Wait for all goroutines to finish
}
```

In this code, even if five goroutines try to get the configuration simultaneously, `initConfig` runs only once. Using `sync.Once` simplifies your code, prevents race conditions, and improves performance, making it a valuable tool for Go developers.

[Read more...](https://medium.com/@andhikaprbw/making-initialization-easy-with-golangs-sync-once-b02a73666497?source=rss------golang-5)

---

### [Golang Concurrency](https://medium.com/@soulaimaneyh/golang-concurrency-d6786d66d134?source=rss------golang-5)

Concurrency in programming means running multiple tasks at the same time, which makes programs more efficient and responsive. In Golang, concurrency is managed using several key features. Goroutines are lightweight threads that allow multitasking. Channels are used for safe communication between these goroutines. The select statement helps manage multiple channels at once. The sync package provides tools like mutexes and wait groups for accessing shared memory safely. Lastly, the context package helps manage the lifecycle of goroutines, including cancellation and setting deadlines.

Here's a simple example of using goroutines and channels in Golang:

```go
package main

import (
    "fmt"
    "time"
)

func sayHello(done chan bool) {
    fmt.Println("Hello, World!")
    time.Sleep(1 * time.Second)
    done <- true
}

func main() {
    done := make(chan bool)
    go sayHello(done)
    <-done
    fmt.Println("Goroutine finished executing")
}
```

This code demonstrates a basic use of a goroutine and a channel to synchronize the completion of a task.

[Read more...](https://medium.com/@soulaimaneyh/golang-concurrency-d6786d66d134?source=rss------golang-5)

---

### [Writing gRPC automated tests in Go with Allure reporting](https://medium.com/@filonov.nikitkaa/writing-grpc-automated-tests-in-go-with-allure-reporting-c1c4a149d9d0?source=rss------golang-5)

The article provides a comprehensive guide on writing automated tests for a gRPC server using Go, and generating Allure reports. It begins by emphasizing the need for a basic understanding of RPC, gRPC, protobuf, Go syntax, and Docker. The author explains the necessity of creating a local gRPC server for testing, with setup instructions available on GitHub. The server can be run directly on the OS or via Docker.

The article details the contract structure for CRUD operations on articles, using protobuf syntax. It lists essential third-party libraries like grpc-go, allure-go, and gomega for testing and reporting. Configuration involves setting up YAML files for different environments, and a parser to read these configurations.

The article describes creating a gRPC client and methods for interacting with the server, including logging and error handling. Dependency injection is implemented using the dig library, and assertions are handled with gomega. Utilities for generating random article data and setting up test components are provided.

Finally, the article outlines writing and running tests, and generating Allure reports, with the complete source code available on GitHub.

Here's a snippet of the protobuf contract:

```protobuf
syntax = "proto3";
option go_package = "./;articlesservice";

service ArticlesService {
    rpc GetArticle (GetArticleRequest) returns (GetArticleResponse);
    rpc CreateArticle(CreateArticleRequest) returns (CreateArticleResponse);
    rpc UpdateArticle(UpdateArticleRequest) returns (UpdateArticleResponse);
    rpc DeleteArticle(DeleteArticleRequest) returns (DeleteArticleResponse);
}

message Article {
    string id = 1;
    string title = 2;
    string author = 3;
    string description = 4;
}
```

[Read more...](https://medium.com/@filonov.nikitkaa/writing-grpc-automated-tests-in-go-with-allure-reporting-c1c4a149d9d0?source=rss------golang-5)

---

### [Show HN: Go-taskflow, A taskflow-like Golang DAG Task Execution](https://github.com/noneback/go-taskflow)

Go-Taskflow is a static Directed Acyclic Graph (DAG) task computing framework for Go, inspired by taskflow-cpp, designed to manage complex dependencies in concurrent tasks using Go's native concurrency model. Key features include high extensibility, a user-friendly interface, and support for static, subflow, and conditional tasking, enhancing modularity and programmability. It also offers built-in visualization and profiling tools to aid in debugging and optimization.

Use cases include orchestrating data pipelines, workflow automation, and parallel task execution. The framework allows users to define tasks and their dependencies, execute them concurrently, and visualize or profile the task execution. The example code demonstrates creating tasks, arranging them in a DAG, and executing them using an executor. Future enhancements include task priority scheduling and taskflow loop support.

```go
executor := gotaskflow.NewExecutor(uint(runtime.NumCPU() - 1))
tf := gotaskflow.NewTaskFlow("G")
A, B, C := gotaskflow.NewTask("A", func() { fmt.Println("A") }), gotaskflow.NewTask("B", func() { fmt.Println("B") }), gotaskflow.NewTask("C", func() { fmt.Println("C") })
A.Precede(B)
C.Precede(B)
tf.Push(A, B, C)
executor.Run(tf).Wait()
```

[Read more...](https://github.com/noneback/go-taskflow)

---

### [An Ode to mockgen: Because Writing Unit Tests Can Be Less Painful](https://araujo88.medium.com/an-ode-to-mockgen-because-writing-unit-tests-can-be-less-painful-38c07fd54af2?source=rss------golang-5)

In Go programming, unit testing can be simplified using mockgen from the GoMock library, which automates the creation of mock objects. To use mockgen, first install it via Go’s package manager with `go get github.com/golang/mock/mockgen`, ensuring your GOPATH is set correctly.

Mockgen operates in two modes: reflect and source, with the source mode reading Go files to generate mock implementations. For example, given a Go file with a `UserProfile` interface, you can generate a mock by running `mockgen -source=path/to/package/user.go -destination=path/to/mocks/user_mock.go`. This command specifies the source file and the destination for the mock file.

In unit tests, import the mock package and use `gomock.Controller` to manage mock lifecycles, setting expectations with the `EXPECT()` method. This approach allows developers to focus on test logic rather than mock setup, enhancing testing efficiency.

[Read more...](https://araujo88.medium.com/an-ode-to-mockgen-because-writing-unit-tests-can-be-less-painful-38c07fd54af2?source=rss------golang-5)

---

### [Go: Optional Arguments in the age of Generics](https://medium.com/@johnsiilver/go-optional-arguments-in-the-age-of-generics-f8eba23103ad?source=rss-ebed66dcde0------2)

In Go, optional arguments for functions are typically handled using a combination of function types and variadic arguments, often seen with a "With" prefix. This pattern allows for clean and readable constructor calls. However, the introduction of generics has complicated this pattern, making it less readable due to the need to specify type parameters.

To address this, a new approach involves using a struct to hold options and applying them through reassignment rather than pointers, which avoids burdening the garbage collector. This method restores the simplicity of the original pattern while accommodating generics. For options requiring type parameters, a runtime check is used to ensure the correct function type is passed, which is a minor trade-off for maintaining readability.

Here's a simplified example of the pattern:

```go
type MyGeneric[T any] struct {
    options options
}

type options struct {
    log *slog.Logger
}

func (o options) defaults() options {
    o.log = slog.Default()
    return o
}

type Option func(o options) options

func WithLogger(l *slog.Logger) Option {
    return func(o options) options {
        o.log = l
        return o
    }
}

func New[T any](options ...Option) *MyGeneric[T] {
    opts := options{}
    g := &MyGeneric{
        options: opts.defaults(),
    }
    for _, o := range options {
        g.options = o(opts)
    }
    return g
}

g := New[int](WithLogger(customLogger))
```

This approach simplifies the use of generics with optional arguments, maintaining the clean syntax of the original pattern.

[Read more...](https://medium.com/@johnsiilver/go-optional-arguments-in-the-age-of-generics-f8eba23103ad?source=rss-ebed66dcde0------2)

---
### [Debug Go core dumps with delve: export byte slices](https://michael.stapelberg.ch/posts/2024-10-22-debug-go-core-dumps-delve-export-bytes/)

The article discusses core dump debugging in Go, particularly when bugs are hard to reproduce. It introduces using the delve debugger to analyze core dumps and save byte slices from memory to files for further analysis.

A simple example is provided using Go Protobuf, where delve is used to inspect byte slices and export them to files using `os.WriteFile`.

The article then shifts to debugging an RPC service using core dumps, explaining how to configure Go to generate core dumps with the `GOTRACEBACK=crash` environment variable and using `coredumpctl` to work with them. It also covers extending delve with a custom Starlark function to export byte slices from core dumps.

For net/http servers, it explains that panics are recovered by default, preventing core dumps, and suggests sending a SIGABRT signal to trigger a core dump.

The article concludes by highlighting the usefulness of core dump debugging in small environments and the potential need for centralized core dump collection in larger setups.

Here's a snippet for exporting byte slices using delve:

```go
# Starlark code to write byte slices to a file
def command_writebytestofile(args):
    var_name, filename = args.split(" ")
    s = eval(None, var_name).Variable
    mem = examine_memory(s.Base, s.Len).Mem
    write_file(filename, mem)
```

[Read more...](https://michael.stapelberg.ch/posts/2024-10-22-debug-go-core-dumps-delve-export-bytes/)