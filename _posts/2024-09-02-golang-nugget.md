---
layout: post
title: "Golang Nugget - September 2, 2024"
date: 2024-09-02
categories: [golang]
excerpt_separator: <!--more-->
redirect_to: https://golangnugget.com/p/golang-grpc-vs-http-static-binaries-debugging-tools-02-09-2024
---
Welcome to this week's edition of **Golang Nugget**, your go-to source for the latest insights and updates in the Go programming world!

This week, we dive into the performance showdown between gRPC+Protobuf and HTTP+JSON, revealing some surprising results that challenge common assumptions about speed and efficiency.

We also explore the intricacies of building static binaries with Go on Linux, offering practical tips to ensure your binaries are truly static, even when incorporating C code.

For those tackling complex debugging challenges, we introduce a powerful tool for hash-based bisect debugging, which simplifies pinpointing bugs in large codebases.

The new `unique` package in Go 1.23 is also in the spotlight, offering a robust solution for deduplicating comparable values efficiently.

If you're building REST APIs, our guide on using Ent and `net/http` in Go will help you get started quickly and effectively.

We also cover the importance of graceful shutdowns in Go applications, especially in Kubernetes environments, to ensure data integrity and resource management.

Lastly, we discuss a clever technique for faking method calls within structs, simplifying testing without cluttering your codebase.

Enjoy the read and happy coding!
<!--more-->
### [Performance Benchmarking: gRPC+Protobuf vs. HTTP+JSON](https://packagemain.tech/p/protobuf-grpc-vs-json-http)

Here's a concise overview you can share over coffee:

1. **Context**: JSON over HTTP is a common choice for service communication, but gRPC with Protocol Buffers is gaining popularity in microservices for its efficiency.
2. **Experiment**: A benchmark was conducted comparing gRPC and HTTP (both HTTP/1 and HTTP/2) in Go, focusing purely on data transport and serialization without additional operations.
3. **Setup**:
   - **gRPC Service**: Defined a `CreateUser` procedure using Protocol Buffers.
   - **HTTP Service**: Replicated the same `CreateUser` functionality using JSON.
4. **Benchmarking**: Benchmarks were run on a local machine using Go’s testing package.
5. **Results**: Surprisingly, HTTP/1 was the fastest, followed by gRPC, with HTTP/2 being the slowest. This contradicts the common belief that gRPC is always faster.
6. **Conclusion**: Despite the results, gRPC with Protocol Buffers remains a solid choice for structured inter-service communication.

Here's a crucial code snippet for the gRPC service setup:

```go
// gRPC service definition
syntax = "proto3";

option go_package = "grpc/gen";

service Users {
  rpc CreateUser(User) returns (CreateUserResponse) {}
}

message User {
  string id = 1;
  string email = 2;
  string name = 3;
}

message CreateUserResponse {
  string message = 1;
  uint64 code = 2;
  User user = 3;
}

// Generate Go code
protoc -I./grpc --go_out=. --go-grpc_out=. users.proto
```

And for the HTTP service:

```go
// HTTP handler
func CreateUser(w http.ResponseWriter, r *http.Request) {
  defer r.Body.Close()

  var user User
  json.NewDecoder(r.Body).Decode(&user)

  w.Header().Set("Content-Type", "application/json")

  json.NewEncoder(w).Encode(Response{
    Code:    201,
    Message: "ok",
    User:    &user,
  })
}
```

So, next time someone tells you gRPC is always faster, you’ve got some fresh insights to share!

[Read more...](https://packagemain.tech/p/protobuf-grpc-vs-json-http)
{% include mailerlite_golang_nugget_postpage.html %}
---

### [Building static binaries with Go on Linux](https://eli.thegreenplace.net/2024/building-static-binaries-with-go-on-linux/)

Go can produce statically-linked binaries, but this isn't always the default and may require extra steps, especially on Unix systems. For a simple "hello world" program, Go produces a statically-linked binary by default. However, when using certain functionalities like DNS lookups or user/group ID lookups, Go relies on the system's libc, resulting in dynamically-linked binaries. You can force static linking by using build tags like `netgo` or `osusergo`, or by disabling cgo with `CGO_ENABLED=0`. When incorporating C code via cgo, Go defaults to dynamic linking due to libc dependencies. To achieve static linking with C code, you can use musl or Zig as the C compiler. Musl can be used by setting the `CC` environment variable and specific linker flags, while Zig simplifies the process and supports cross-compilation. Despite these methods, a proposal for a `-static` flag in `go build` could streamline static linking in the future. Here's a simple example:

```go
package main

import "fmt"

func main() {
  fmt.Println("hello world")
}
```

Build with static linking:
```sh
$ CGO_ENABLED=0 go build -o helloworld
$ ldd ./helloworld
  not a dynamic executable
```

[Read more...](https://eli.thegreenplace.net/2024/building-static-binaries-with-go-on-linux/)

---

### [Hash-Based Bisect Debugging in Compilers and Runtimes](http://research.swtch.com/bisect)

The post discusses a powerful debugging tool called "bisect" that uses binary search techniques to pinpoint the exact line of code or call stack causing a bug in an unfamiliar codebase. It starts by explaining traditional binary search methods for debugging data and version history, then introduces the concept of bisecting program locations. The tool automates the process of narrowing down the failure to specific lines of code or call stacks, making debugging large, complex programs more manageable. The post provides examples of using bisect for function optimization, SSA rewrite selection, and library changes, highlighting its effectiveness in identifying issues caused by new implementations. It also covers the implementation details of the bisect-reduce algorithm, including list-based, counter-based, and hash-based approaches. The tool is particularly useful for debugging flaky tests and runtime changes, and the post concludes by encouraging readers to try bisect for their own debugging needs. Here's a snippet of the code for the bisect-reduce algorithm:

```go
func BisectReduce(targets []string) []string {
    return bisect(targets, []string{})
}

func bisect(targets []string, forced []string) []string {
    if len(targets) == 0 || buggy(forced) {
        return []string{}
    }
    if len(targets) == 1 {
        return []string{targets[0]}
    }

    m := len(targets)/2
    left, right := targets[:m], targets[m:]
    leftReduced := bisect(left, slices.Concat(right, forced))
    rightReduced := bisect(right, slices.Concat(leftReduced, forced))
    return slices.Concat(leftReduced, rightReduced)
}
```

[Read more...](http://research.swtch.com/bisect)

---

### [New unique package](https://go.dev/blog/unique)

The Go 1.23 standard library introduces the new `unique` package, designed for canonicalizing comparable values, effectively deduplicating them to a single unique copy, a process known as interning. The package provides a `Make` function that works with any comparable type and returns a `Handle[T]`, which ensures efficient pointer comparisons and automatic cleanup of unused entries. This is an improvement over simpler interning implementations that only work with strings and lack concurrency safety. The `unique` package is already utilized in the `net/netip` package to optimize memory usage and comparison efficiency for IP addresses. Additionally, the package leverages weak pointers, now supported by Go's garbage collector, to manage memory efficiently. Here's a snippet demonstrating the basic usage of `unique.Make`:

```go
var z6noz = unique.Make(addrDetail{isV6: true})

func (ip Addr) WithZone(zone string) Addr {
    if !ip.Is6() {
        return ip
    }
    if zone == "" {
        ip.z = z6noz
        return ip
    }
    ip.z = unique.Make(addrDetail{isV6: true, zoneV6: zone})
    return ip
}
```

[Read more...](https://go.dev/blog/unique)

---

### [Building a REST API in Go with Ent and net/http](https://golang.ch/building-a-rest-api-in-go-with-ent-and-net-http/)

Imagine we're sipping coffee and I hit you with this:

### Key Points to Build a REST API in Go with Ent

1. **Install Ent**: First, you need to install the Ent framework.
   ```bash
   go get entgo.io/ent/cmd/ent
   ```

2. **Define Schema**: Create a `Contact` schema with fields for `name` and `email`.
   ```go
   package schema

   import (
       "entgo.io/ent"
       "entgo.io/ent/schema/field"
   )

   type Contact struct {
       ent.Schema
   }

   func (Contact) Fields() []ent.Field {
       return []ent.Field{
           field.String("name").NotEmpty(),
           field.String("email").NotEmpty().Unique(),
       }
   }
   ```

3. **Generate Code**: Run the code generation to create Go code for your schema.
   ```bash
   go generate ./ent
   ```

4. **Set Up HTTP Server**: Create an HTTP server with an endpoint to handle POST requests for creating new contacts.
   ```go
   package main

   import (
       "context"
       "encoding/json"
       "log"
       "net/http"

       "entgo.io/ent/dialect/sql/schema"
       "myapp/ent"
       _ "github.com/mattn/go-sqlite3"
   )

   type CreateContactRequest struct {
       Name  string `json:"name"`
       Email string `json:"email"`
   }

   func main() {
       client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
       if err != nil {
           log.Fatalf("failed opening connection to sqlite: %v", err)
       }
       defer client.Close()

       if err := client.Schema.Create(context.Background(), schema.WithAtlas(true)); err != nil {
           log.Fatalf("failed creating schema resources: %v", err)
       }

       http.HandleFunc("/contacts", func(w http.ResponseWriter, r *http.Request) {
           if r.Method != http.MethodPost {
               http.Error(w, "Invalid request method", http.StatusMethodNotAllowed)
               return
           }

           var req CreateContactRequest
           if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
               http.Error(w, err.Error(), http.StatusBadRequest)
               return
           }

           contact, err := client.Contact.Create().
               SetName(req.Name).
               SetEmail(req.Email).
               Save(context.Background())
           if err != nil {
               http.Error(w, err.Error(), http.StatusInternalServerError)
               return
           }

           json.NewEncoder(w).Encode(contact)
       })

       log.Println("Server is running on http://localhost:8080")
       log.Fatal(http.ListenAndServe(":8080", nil))
   }
   ```

5. **Run the Server**: Start the server.
   ```bash
   go run main.go
   ```

6. **Test the Endpoint**: Use `curl` to create a new contact.
   ```bash
   curl -X POST -H "Content-Type: application/json" -d '{"name":"John Doe","email":"john.doe@example.com"}' http://localhost:8080/contacts
   ```

And boom, you've got a REST API in Go using Ent and `net/http`!

[Read more... ](https://golang.ch/building-a-rest-api-in-go-with-ent-and-net-http/)

---

### [Terminating Elegantly: A Guide to Graceful Shutdowns](https://packagemain.tech/p/graceful-shutdowns-k8s-go)

Yanking your computer's power cord can cause data loss and system instability, similar to a hard shutdown in software. A better approach is a graceful shutdown, which allows services to complete tasks and save data before shutting down. This guide explains how to implement graceful shutdowns in Go applications on Kubernetes using Unix signals like SIGTERM. We create a simple Go service with a Redis backend to test this. Initially, the service loses data during a Kubernetes rolling update. By handling SIGTERM with Go's signal package and using a sync.WaitGroup, we ensure all requests are processed before shutdown. Here's the improved code:

```go
package main

import (
  "context"
  "log"
  "net/http"
  "os"
  "os/signal"
  "sync"
  "syscall"
  "time"

  "github.com/go-redis/redis"
)

var wg sync.WaitGroup

func main() {
  ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM)
  defer stop()

  redisdb := redis.NewClient(&redis.Options{
    Addr: os.Getenv("REDIS_ADDR"),
  })

  server := http.Server{
    Addr: ":8080",
  }

  http.HandleFunc("/incr", func(w http.ResponseWriter, r *http.Request) {
    wg.Add(1)
    go processRequest(redisdb)
    w.WriteHeader(http.StatusOK)
  })

  go server.ListenAndServe()

  <-ctx.Done()

  if err := server.Shutdown(context.Background()); err != nil {
    log.Fatalf("could not shutdown: %v\n", err)
  }

  wg.Wait()
  redisdb.Close()
  os.Exit(0)
}

func processRequest(redisdb *redis.Client) {
  defer wg.Done()
  time.Sleep(5 * time.Second)
  redisdb.Incr("counter")
}
```

This ensures all requests are handled before shutdown, maintaining data integrity. Kubernetes handles pod termination by sending SIGTERM, allowing our application to shut down gracefully. For complex shutdowns, a timeout can be set. Graceful shutdowns are crucial for data integrity and resource management, especially in containerized environments like Kubernetes. For more details, check our Github repository.

[Read more... ](https://packagemain.tech/p/graceful-shutdowns-k8s-go)

---

### [Go: Faking Method Calls Within a Struct](https://medium.com/@johnsiilver/go-faking-method-calls-within-a-struct-2bf3e5af9e29?source=rss-ebed66dcde0------2)

In a previous article, we discussed using functional State Machines to simplify testing by breaking down complex call chains into manageable states. This follow-up addresses testing helper methods within structs without redundant logic testing. Instead of overusing interfaces, which can be sub-optimal for state changes, we can attach a field with the same function signature as the helper function to the struct. This allows injecting test code easily. For example:

```go
type MyStruct struct {
  fakeRun func(ctx context.Context, arg string) error
}

func (m *MyStruct) PublicFuncThatCallsRunSomething(ctx context.Context, arg string) error {
  result := m.runSomething(ctx, arg)
  // Rest of the code
}

func (m *MyStruct) runSomething(ctx context.Context, arg string) error {
  if m.fakeRun != nil {
    return m.fakeRun(ctx, arg)
  }
  // Regular code
}
```

This method allows testing the helper function separately and injecting fakes for tests. If internal state mutation is needed, pass the struct to the fake function. For verifying call arguments or ensuring a call happened, you can manipulate test variables or use a struct to record results. This approach simplifies testing without cluttering the code. For more on leveraging Go for DevOps, check out my book.

[Read more... ](https://medium.com/@johnsiilver/go-faking-method-calls-within-a-struct-2bf3e5af9e29?source=rss-ebed66dcde0------2)