---
layout: post
title: "GoLang Nugget - October 7, 2024"
date: 2024-10-07
categories: [golang]
excerpt_separator: <!--more-->
---
Welcome to this week's edition of **GoLang Nugget**, your go-to source for the latest insights and tips in the Go programming world.

This week, we dive into the essentials of writing effective unit tests in Go, ensuring your code is robust and bug-free. We also explore the gaps in Go's standard library and how third-party tools can enhance your development experience.

For those looking to optimize Docker images, we provide a step-by-step guide to slimming down your Go app images, making them lean and efficient for production.

We also take a closer look at the Go compiler's register allocation process, offering insights into its optimizations and challenges.

Additionally, discover how Go tests have evolved over the years, with a focus on testing CLI tools using the `testscript` package.

In the realm of distributed systems, we discuss strategies for handling transactions across microservices, emphasizing the benefits of eventual consistency.

Lastly, learn about the FAIR library, designed to ensure fair resource allocation in multi-tenant environments, making it a valuable tool for distributed systems.

Stay tuned for more nuggets of wisdom in the world of Go!
<!--more-->
### [Writing Effective Unit Tests in Golang: A Practical Guide](https://medium.com/@abhijith0807/writing-effective-unit-tests-in-golang-a-practical-guide-0d24444769f6?source=rss------golang-5)

Imagine you're constructing a house and need to ensure each brick is solid before stacking them. That's the role of unit testing in your code. In Golang, the built-in `testing` package simplifies writing these tests. You can also leverage mocking frameworks like `gomock`, `testify`, and `bounegru/monkey` to simulate dependencies and ensure your code functions in isolation. This practice catches bugs early, enhances code quality, and makes your software easier to maintain. Here's a quick example to get you started:

```go
import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Expected 5, got %d", result)
    }
}
```

This snippet tests a simple `Add` function to ensure it returns the correct sum.

[Read more...](https://medium.com/@abhijith0807/writing-effective-unit-tests-in-golang-a-practical-guide-0d24444769f6?source=rss------golang-5)
{% include golang_nugget_mailerlite.html %}
---

### [Golang: Some batteries not included (2021)](https://yolken.net/blog/golang-batteries-not-included)

Here's the essential guide to using Go (Golang) and its standard library, along with key third-party libraries to fill in the gaps:

1. **Flags**: Go’s built-in `flag` package is basic. For enhanced flag handling, consider:
    - **cobra**: Popular and robust.
    - **segmentio/cli**: Simple and easy for basic CLIs.

2. **Logging**: The standard `log` package lacks features like log levels and structured output. Alternatives include:
    - **logrus**: Supports log levels and structured formats.
    - **segmentio/events**: Great for structured logging in backend systems.

3. **Testing**: Go’s testing lacks assert functions. Use:
    - **testify**: Provides `assert` and `require` functions for cleaner tests.

4. **YAML Handling**: Go doesn’t support YAML natively. Use:
    - **go-yaml**: Standard for YAML parsing.
    - **ghodss/yaml**: Wraps `go-yaml` and supports JSON-compatible tags.

5. **Asset Embedding**: For embedding static assets in binaries, use:
    - **go-bindata**: Embeds assets into Go files for easy inclusion in binaries.

Here's a quick code snippet for using `logrus`:

```go
import (
    log "github.com/sirupsen/logrus"
)

func main() {
    log.Info("This is an info message")
    log.Debug("This is a debug message")
}
```

Remember, Go’s standard library is solid, but sometimes you need these third-party tools to make your life easier.

[Read more...](https://yolken.net/blog/golang-batteries-not-included)

---

### [Dockerizing Golang Apps: A Step-by-Step Guide to Reducing Docker Image Size](https://medium.com/code-beyond/dockerizing-golang-apps-a-step-by-step-guide-to-reducing-docker-image-size-306898e7359e?source=rss------golang-5)

In the realm of Docker image optimization for Go applications, transitioning from bulky to sleek is all about strategic choices. Start with the full `golang:1.22.5` image, which is hefty due to its comprehensive toolset. Transition to `golang:1.22.5-alpine` for a lighter footprint, but the real magic happens with multi-stage builds. First, compile your Go app in an Alpine environment, then transfer only the binary to a minimal base image, like `alpine:latest`, shedding unnecessary weight. For ultimate efficiency, use the `scratch` base image, which is completely empty, and compile your Go binary with `CGO_ENABLED=0` and `-ldflags="-s -w"` to strip debugging info. This approach slashes your Docker image size from 1.25 GB to a mere 12 MB, making it nimble and production-ready. Here's a crucial snippet for the final optimization step:

```dockerfile
FROM golang:1.22.5-alpine as builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ENV CGO_ENABLED=0
RUN go build -ldflags="-s -w" -o main .

FROM scratch
WORKDIR /app
COPY --from=builder /app/main .
CMD ["./main"]
```

This method not only accelerates deployment but also minimizes the attack surface, though it requires careful handling due to the lack of debugging tools in the `scratch` image.

[Read more...](https://medium.com/code-beyond/dockerizing-golang-apps-a-step-by-step-guide-to-reducing-docker-image-size-306898e7359e?source=rss------golang-5)

---

### [Register allocation in the Go compiler](https://developers.redhat.com/articles/2024/09/24/go-compiler-register-allocation)

This post delves into the Go compiler's register allocator (RA), detailing its components, processes, and optimizations. Key points include the necessity of critical edge elimination in the control flow graph (CFG), the handling of SSA values, and the allocation of registers and stack slots. The Go RA processes basic blocks in CFG preorder and uses heuristics based on the longest distance to the next use for spilling values. It employs techniques like value rematerialization, register shuffling, and handling SSA Phis to ensure correct value allocation. The RA also integrates stack slot allocation, leveraging a conflict graph to optimize stack usage. The Go RA uses 64-bit masks for registers and maintains SSA values mapped to registers, employing special operations like StoreReg and LoadReg. The post compares Go's RA to other methods like linear scan and second-chance bin-packing, noting its local scope with some global optimizations. The Go RA assumes all registers are clobbered by calls, which can lead to inefficient code, but suggests using call-saved registers for better performance. The post concludes by highlighting the advantages and drawbacks of the current Go RA.

```go
// Example of handling SSA values in Go RA
func regalloc() {
    for _, bb := range cfg.preorder() {
        for _, v := range bb.values {
            if !v.needsReg() {
                continue
            }
            reg := findFreeReg(v)
            if reg == -1 {
                spillValue(v)
            } else {
                assignReg(v, reg)
            }
        }
    }
}
```

[Read more...](https://developers.redhat.com/articles/2024/09/24/go-compiler-register-allocation)

---

### [How Go Tests "go test"](https://atlasgo.io/blog/2024/09/09/how-go-tests-go-test)

Here's the distilled essence of that post:

1. **CLI Tools in 2024**: If you're a software engineer, you're likely using CLI tools like Docker, kubectl, Terraform, or Atlas (a database schema management tool written in Go).

2. **Testing CLI Tools**: Testing CLI tools involves unique challenges but follows the same basic phases as other software tests: Arrange, Act, Assert, and Cleanup.

3. **Go's Evolution in Testing**:
    - **Early Days**: Initially, Go's CLI tool was tested using a large shell script (`test.bash`), which became cumbersome.
    - **2015 Shift**: Introduced `go_test.go` with the `testgo` framework, making tests more manageable and compatible across platforms.
    - **2018 Upgrade**: Introduced `script_test.go`, a new framework using a shell-like language for writing tests, encapsulated in `txtar` archives.

4. **Testscript Package**: Roger Peppe created the `testscript` package, based on `script_test.go`, making it accessible for broader use. It allows writing tests in a shell-like language, running them in parallel, and automatic cleanup.

5. **Practical Example**: Demonstrated using `testscript` to test a simple CLI tool named `wordwrap`. The tool wraps text files at a specified width, and tests verify its behavior using custom commands.

6. **Impact on Atlas**: The Atlas team uses `testscript` extensively, creating custom commands to streamline their testing process, ensuring high reliability and quality of their schema management tool.

Here’s a quick code snippet to illustrate how `testscript` is used:

```go
// wordwrap_test.go
package main

import (
    "testing"
    "github.com/rogpeppe/go-internal/testscript"
)

func TestScript(t *testing.T) {
    testscript.Run(t, testscript.Params{
        Dir: "testdata",
        // Register custom commands if any
        Setup: func(env *testscript.Env) error {
            env.Setenv("CUSTOM_VAR", "value")
            return nil
        },
    })
}
```

This snippet sets up a basic test environment using `testscript`, allowing you to write and run shell-like test scripts for your CLI tool.

[Read more...](https://atlasgo.io/blog/2024/09/09/how-go-tests-go-test)

---

### [Distributed Transactions in Go](https://threedots.tech/post/distributed-transactions-in-go/)

In the realm of microservices, when transactions need to span multiple services, things can get messy. If Service A calls Service B, which then calls Service C, and something fails, you risk inconsistency. Distributed transactions or the saga pattern can solve this, but they often complicate your architecture unnecessarily. Instead, consider eventual consistency, where data becomes consistent over time, like a bank transfer that takes hours to reflect. This approach can simplify your system by avoiding complex rollbacks and distributed locks. Use event-driven patterns with tools like Watermill to publish events asynchronously, ensuring consistency without tight coupling. Implement the outbox pattern to store events in the same transaction as your data, then publish them to a message queue. This way, you avoid losing events if the network fails. Remember, well-designed events should state facts within a service's domain, not dictate actions in another service. Testing these systems involves running your Pub/Sub locally and using component tests to verify behavior. Monitor your message queue for unprocessed messages to catch issues early. Embrace eventual consistency to simplify distributed systems and avoid the pitfalls of distributed transactions.

[Read more...](https://threedots.tech/post/distributed-transactions-in-go/)

---

### [FAIR: Allocating Resources Fairly at Scale](https://medium.com/@mihsathe/fair-allocating-resources-fairly-at-scale-8c3a54ecee35?source=rss------golang-5)

The Go library FAIR, developed by Mihir Sathe, is designed to ensure fair resource distribution in multi-tenant environments using a stochastic fair BLUE algorithm with constant memory requirements. It aims to be a "fit-and-forget" solution, minimizing the need for tuning and operational overhead. FAIR addresses fairness by throttling heavy hitters, allowing clients with lower request rates to access resources with minimal throttling. It uses a multi-level counting Bloom filter structure to manage resource allocation efficiently, avoiding false positives through hash rotation. This approach maintains fairness without per-client quotas, ensuring equitable resource distribution even during resource contention. FAIR's constant memory usage and adaptability make it a valuable tool for distributed systems. The library is available on GitHub for contributions and feedback. Here's a snippet illustrating the hash rotation strategy:

```go
func rotateHashes(clientID string, levels int, buckets int) []int {
    hashes := make([]int, levels)
    baseHash := murmur3.Sum32([]byte(clientID))
    for i := 0; i < levels; i++ {
        hashes[i] = (baseHash + i) % buckets
    }
    return hashes
}
```

[Read more...](https://medium.com/@mihsathe/fair-allocating-resources-fairly-at-scale-8c3a54ecee35?source=rss------golang-5)