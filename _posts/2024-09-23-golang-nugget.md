---
layout: post
title: "Golang Nugget - September 23, 2024"
date: 2024-09-23
categories: [golang]
excerpt_separator: <!--more-->
---
Welcome to this week's edition of **Golang Nugget**, your go-to source for the latest insights and updates in the Go programming world.

In our first highlight, we delve into the world of package-level logging in Go, inspired by Java's log4j. Discover how you can achieve detailed logging for specific code sections without recompilation using popular Go logging frameworks like Zap and Logrus. Learn about the pros and cons of each approach and explore the author's prototypes available on GitHub.

Next, we explore an exciting new feature in Go 1.24—generic alias types. This advancement simplifies refactoring large codebases by allowing seamless movement of generic types between packages. Understand how this feature maintains type identities and enhances code readability, making it a valuable tool for large-scale projects.

Stay tuned for more Go insights and happy coding!
<!--more-->
### [Building package-level, runtime configurable logging ala log4j in Go](https://dolthub.com/blog/2024-09-13-package-scoped-logging-in-go-log4j/)

This post explores the author's preference for Go over Java, with the exception of missing Java's package-level logging configuration feature found in log4j. The author elaborates on the advantages of package-level logging, which enables detailed logging for specific code sections without the need for recompilation. To implement this in Go, the author develops prototypes using two popular Go logging frameworks: Zap and Logrus. The Zap prototype utilizes the `zapfilter` package to filter logs based on logger names, while the Logrus prototype employs hooks to filter logs based on the runtime stack. Both solutions facilitate changing logging behavior through configuration files. The author observes that Zap is significantly faster than Logrus but notes a performance decline in Zap if `Sync()` is called frequently. The post concludes with a suggestion to enhance Zap's ergonomics by extending `zapfilter` to filter logs based on file names or functions. The full prototypes are available on GitHub, and the author welcomes feedback on the concept of package-level logging configuration for Go.

Snippet from the Zap prototype:
```go
func InitFromFile(configFile string) {
	file, err := os.ReadFile(configFile)
	if err != nil {
		panic(err)
	}
	filterRules = zapfilter.MustParseRules(string(file))
}

func NewProductionLogger(namespace string) *zap.Logger {
	config := zap.NewProductionConfig()
	config.OutputPaths = []string{"output.log"}
	config.Sampling = nil
	config.Level = zap.NewAtomicLevelAt(zap.DebugLevel)

	underlying, err := config.Build()
	if err != nil {
		panic(err)
	}

	return zap.New(zapfilter.NewFilteringCore(underlying.Core(), filterRules)).Named(namespace)
}
```

[Read more...](https://dolthub.com/blog/2024-09-13-package-scoped-logging-in-go-log4j/)
{% include mailerlite_main_embedded.html %}
---

### [What's in an (Alias) Name?](https://go.dev/blog/alias-names)

Imagine we're having a coffee chat, and here's the scoop: Go is introducing a fantastic new feature in version 1.24—generic alias types. This is a significant advancement for refactoring large codebases, particularly when moving generic types between packages without causing disruptions. Previously, you could alias types, but not with generics. Now, you can achieve something like this:

```go
package pkg2

import "path/to/pkg1"

type Constraint = pkg1.Constraint
type G[P Constraint] = pkg1.G[P]
```

This means `pkg2.G` is equivalent to `pkg1.G`, maintaining type identities. It's incredibly useful for incremental updates and ensures your code remains clean and readable. Additionally, you can adjust type parameters for greater flexibility. This update enhances Go's robustness for large-scale projects.

[Read more...](https://go.dev/blog/alias-names)