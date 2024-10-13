---
layout: post
title: "GoLang Nugget - September 16, 2024"
date: 2024-09-16
categories: [golang]
excerpt_separator: <!--more-->
---
Welcome to this week's edition of **GoLang Nugget**, your go-to source for all things Go!

In this issue, we dive into the world of Go binaries, exploring the differences between static and dynamic linking. Discover how to make your Go programs more portable and secure by choosing the right linking method.

Next, we venture into the browser with Go and WebAssembly. Learn how to run Go code in the browser, manipulate the DOM, and keep your UI responsive using Web Workers. Plus, find out how TinyGo can help you create smaller, faster-loading WASM binaries.

Finally, we explore building LLM-powered applications in Go. Uncover how Go's strengths in concurrency and performance make it ideal for creating scalable applications that leverage large language models. From setting up a RAG server to using frameworks like LangChainGo, we've got you covered.

Stay tuned for more insights and tips to enhance your Go programming journey!
<!--more-->
### [Statically and Dynamically Linked Go Binaries](https://packagemain.tech/p/golang-statically-and-dynamically-linked-binaries)

Imagine we're sipping coffee and discussing this: Go's compiler is a powerhouse—it simplifies cross-platform compilation. However, there's a twist: you can compile your Go program in various ways, resulting in either statically or dynamically linked executables.

**Static linking** bundles all necessary libraries into the final executable, enhancing portability. Your binary can run anywhere without concerns about missing libraries.

**Dynamic linking**, conversely, links libraries at runtime. This approach can save space and allow you to benefit from system updates without recompiling your program.

Here's a quick method to check if your Go binary is statically or dynamically linked:
```sh
file your_binary
ldd your_binary
```

For static linking, especially when using cgo (which calls C code), you can disable it with:
```sh
CGO_ENABLED=0 go build
```

This ensures your binary is fully self-contained.

Go also has an internal linker but can use external ones like `gcc`'s `ld` for more control. However, be cautious with libraries like `libnss` that can't be statically linked.

Cross-compilation is another strong suit of Go, but it gets tricky with cgo. It's best to avoid cgo for smooth cross-compiling.

Lastly, you can strip debug info to reduce binary size:
```sh
go build -ldflags="-w -s"
```

And watch out for the `LD_PRELOAD` trick in dynamically linked binaries—it can inject custom code, posing a security risk. Static binaries don't have this issue.

Go's compilation options offer flexibility, but knowing when to use static vs dynamic linking is crucial for portability and security.

[Read more...](https://packagemain.tech/p/golang-statically-and-dynamically-linked-binaries)

---

### [Notes on running Go in the browser with WebAssembly](https://eli.thegreenplace.net/2024/notes-on-running-go-in-the-browser-with-webassembly/)

I've been exploring compiling Go to WebAssembly (WASM) to run in browsers for some projects, and it's quite exciting. WebAssembly enables running Go code in the browser, which is excellent for reusing existing Go projects. Here's a summary of useful patterns for running Go in the browser via WASM, demonstrated with small programs in a GitHub repository.

#### Basics: Calling Go from JS

We create a Go function `calcHarmonic` to calculate the harmonic series and export it to JS:
```go
func calcHarmonic(nsecs float64) string {
  // ... calculation logic ...
  return r1.FloatString(40)
}

func main() {
  js.Global().Set("calcHarmonic", jsCalcHarmonic)
  select {}
}

var jsCalcHarmonic = js.FuncOf(func(this js.Value, args []js.Value) any {
  s := calcHarmonic(args[0].Float())
  return js.ValueOf(s)
})
```
Compile with:
```sh
GOOS=js GOARCH=wasm go build -o harmonic.wasm harmonic.go
```
Load in JS:
```js
const go = new Go();
WebAssembly.instantiateStreaming(fetch("harmonic.wasm"), go.importObject).then(
    (result) => { go.run(result.instance); }
);
```
Call from JS:
```js
document.getElementById("submitButton").addEventListener("click", () => {
    let input = document.getElementById("timeInput").value;
    let s = calcHarmonic(parseFloat(input));
    document.getElementById("outputDiv").innerText = s;
});
```
Include `wasm_exec.js` from the Go project.

#### DOM Manipulation from Go

Move more logic to Go:
```go
func main() {
  doc := js.Global().Get("document")
  buttonElement := doc.Call("getElementById", "submitButton")
  inputElement := doc.Call("getElementById", "timeInput")
  outputElement := doc.Call("getElementById", "outputDiv")

  buttonElement.Call("addEventListener", "click", js.FuncOf(
    func(this js.Value, args []js.Value) any {
      input := inputElement.Get("value")
      inputFloat, _ := strconv.ParseFloat(input.String(), 64)
      s := calcHarmonic(inputFloat)
      outputElement.Set("innerText", s)
      return nil
    }))

  select {}
}
```
Only the WebAssembly loader remains in JS.

#### Using TinyGo

TinyGo produces smaller WASM binaries but has limitations like slower compilation and lack of support for some stdlib packages. Use TinyGo for smaller, faster-loading binaries.

#### WebAssembly in a Web Worker

To keep the main thread free, use Web Workers:
```js
const worker = new Worker("worker.js");
worker.onmessage = ({ data }) => {
    if (data.action === "result") resultReady(data.payload);
};
```
`worker.js`:
```js
importScripts("wasm_exec.js");
const go = new Go();
WebAssembly.instantiateStreaming(fetch("harmonic.wasm"), go.importObject).then(
    (result) => { go.run(result.instance); }
);

onmessage = ({ data }) => {
    if (data.action === "calculate") {
        let result = calcHarmonic(data.payload);
        postMessage({ action: "result", payload: result });
    }
};
```
This keeps the UI responsive while computations run in the background.

For full code samples, check the provided GitHub repository.

[Read more...](https://eli.thegreenplace.net/2024/notes-on-running-go-in-the-browser-with-webassembly/)

---

### [Building LLM-powered applications in Go](https://go.dev/blog/llmpowered)

Picture this: you're sipping your coffee, and here's the essentials on building LLM-powered apps in Go. Here’s the lowdown:

1. **LLMs as Network Services**: LLMs (like OpenAI, Google Gemini) require significant compute power, so they’re typically accessed via APIs. Even DIY LLMs use REST APIs.
2. **Go’s Strengths**: Go is ideal for these apps because it excels in REST/RPC protocols, concurrency, and performance.
3. **RAG Server**: We’re discussing a Retrieval Augmented Generation (RAG) server in Go. It performs two functions:
   - **Add Documents**: Users can add documents to a knowledge base.
   - **Ask Questions**: Users can query the knowledge base, and the server uses an LLM to answer.
4. **Components**:
   - **Embedding Model**: Converts text to vector embeddings.
   - **Vector Database**: Stores and retrieves these embeddings.
   - **LLM**: Answers questions using the context from the knowledge base.
5. **Endpoints**:
   - `/add/`: Adds documents.
   - `/query/`: Queries the knowledge base.

Here’s a quick Go snippet for setting up routes:
```go
mux := http.NewServeMux()
mux.HandleFunc("POST /add/", server.addDocumentsHandler)
mux.HandleFunc("POST /query/", server.queryHandler)
```
6. **Concurrency**: Go’s goroutines handle multiple requests efficiently.
7. **Batch APIs**: For efficiency, batch processing is used for embeddings and database operations.

**Variants**:
- **Direct API Use**: Uses Google Gemini and Weaviate directly.
- **LangChainGo**: A framework that simplifies switching between different LLM and vector DB providers.
- **Genkit for Go**: Focuses on production features like prompt management and deployment.

In essence, Go makes it straightforward to build scalable, efficient LLM-powered applications with minimal code. Perfect for cloud-native environments.

[Read more...](https://go.dev/blog/llmpowered)