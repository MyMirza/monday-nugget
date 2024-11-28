---
layout: post
title: "Golang Nugget - September 9, 2024"
date: 2024-09-09
categories: [golang]
excerpt_separator: <!--more-->
redirect_to: https://golangnugget.com/p/golang-1-23-telemetry-google-sheets-database-proxies-09-09-2024
---
Welcome to this week's edition of **Golang Nugget**, your go-to source for the latest in the Go programming world.

This week, we dive into the new telemetry features introduced in Go 1.23. Telemetry helps improve the Go toolchain by collecting data to fix bugs and make informed decisions. Learn how to enable or disable telemetry and discover its role in enhancing Go's growth.

Next, we explore how to read Google Sheets using Go. This guide walks you through setting up a service account and using OAuth 2.0 for authentication, making it easy to access and process data from Google Sheets in your Go applications.

Lastly, we discuss the use of Database Proxies. These proxies act as a middle layer between applications and databases, simplifying complex systems and enhancing security. Check out a simple Go example that demonstrates how to create a proxy to manage SQL queries effectively.

Stay tuned for more insights and happy coding!
<!--more-->
### [Telemetry in Go 1.23 and beyond](https://go.dev/blog/gotelemetry)

Go 1.23 introduces telemetry to enhance the Go toolchain by enabling users to share data about toolchain programs. This data aids in bug fixes and informed decision-making. By default, telemetry data is stored locally, but users can enable uploading with `go telemetry on` and disable it with `go telemetry off`. Initially, explicit user consent was required for remote uploading. Telemetry was first implemented in the Go language server `gopls` in October 2023, with early adopters helping to identify bugs. To boost participation, a prompt was added to the VS Code Go plugin, increasing the sample size to around 1800 weekly participants. Telemetry has been instrumental in identifying and fixing bugs, with future plans to expand telemetry to other tools and improve user experience metrics. Automated crash reporting is also introduced in Go 1.23 with the `runtime.SetCrashOutput` API. Telemetry will be pivotal in the ongoing growth and enhancement of Go.

```go
// Enable telemetry
go telemetry on

// Disable telemetry
go telemetry off
```

[Read more...](https://go.dev/blog/gotelemetry)
{% include mailerlite_golang_nugget_postpage.html %}
---

### [Reading Google Sheets from a Go program](https://eli.thegreenplace.net/2024/reading-google-sheets-from-a-go-program/)

This post explores methods to process data from a Google Sheet in a Go program, focusing on using a service account and OAuth 2.0 for authentication. To utilize the Sheets API, you need a GCP project and the gcloud command-line tool to enable the API. The simplest approach involves creating a service account, downloading its private key, and using it to authenticate and access the Sheets API. The provided Go code demonstrates reading the key file, setting up JWT configuration, and retrieving data from a specified Google Sheet. The post also briefly mentions an alternative OAuth 2.0 approach, which requires initial setup in the GCP console and handles token exchange automatically. Additionally, the author notes that Application Default Credentials (ADC) can also be used, though they initially faced issues with it. The full code for all methods is available on GitHub.

Snippet:
```go
package main

import (
  "context"
  "flag"
  "fmt"
  "io/ioutil"
  "log"

  "golang.org/x/oauth2/google"
  "google.golang.org/api/option"
  "google.golang.org/api/sheets/v4"
)

func main() {
  keyFilePath := flag.String("keyfile", "", "path to the credentials file")
  flag.Parse()

  ctx := context.Background()
  credentials, err := ioutil.ReadFile(*keyFilePath)
  if err != nil {
    log.Fatal("unable to read key file:", err)
  }

  scopes := []string{
    "https://www.googleapis.com/auth/spreadsheets.readonly",
  }
  config, err := google.JWTConfigFromJSON(credentials, scopes...)
  if err != nil {
    log.Fatal("unable to create JWT configuration:", err)
  }

  srv, err := sheets.NewService(ctx, option.WithHTTPClient(config.Client(ctx)))
  if err != nil {
    log.Fatalf("unable to retrieve sheets service: %v", err)
  }

  docId := "1qsNWsZuw98r9HEl01vwxCO5O1sIsI-fr0bJ4KGVvWsU"
  doc, err := srv.Spreadsheets.Get(docId).Do()
  if err != nil {
    log.Fatalf("unable to retrieve data from document: %v", err)
  }
  fmt.Printf("The title of the doc is: %s\n", doc.Properties.Title)

  val, err := srv.Spreadsheets.Values.Get(docId, "Sheet1!A:B").Do()
  if err != nil {
    log.Fatalf("unable to retrieve range from document: %v", err)
  }

  fmt.Printf("Selected major dimension=%v, range=%v\n", val.MajorDimension, val.Range)
  for _, row := range val.Values {
    fmt.Println(row)
  }
}
```

[Read more...](https://eli.thegreenplace.net/2024/reading-google-sheets-from-a-go-program/)

---

### [The Developer's Guide to Database Proxies: When to Use Them and How to Create One](https://packagemain.tech/p/the-developers-guide-to-database)

A Database Proxy serves as a middle layer between applications and databases, simplifying complex systems by managing tasks like query routing, validation, and security. It isolates database details from developers, facilitating easier schema changes and enhanced security. Use cases include intercepting and routing SQL queries, enforcing security policies, and improving performance through connection pooling. However, it introduces complexity and potential latency. A simple Go example demonstrates a proxy that rewrites SQL queries to route from one table to another. Here's a basic implementation:

```go
package main

import (
  "fmt"
  "io"
  "log"
  "net"
  "os"
)

func main() {
  proxy, err := net.Listen("tcp", ":3307")
  if err != nil {
    log.Fatalf("failed to start proxy: %s", err.Error())
  }

  for {
    conn, err := proxy.Accept()
    if err != nil {
      log.Fatalf("failed to accept connection: %s", err.Error())
    }
    go transport(conn)
  }
}

func transport(conn net.Conn) {
  defer conn.Close()
  mysqlAddr := fmt.Sprintf("%s:%s", os.Getenv("MYSQL_HOST"), os.Getenv("MYSQL_PORT"))
  mysqlConn, err := net.Dial("tcp", mysqlAddr)
  if err != nil {
    log.Printf("failed to connect to mysql: %s", err.Error())
    return
  }
  go pipe(mysqlConn, conn, true)
  go pipe(conn, mysqlConn, false)
}

func pipe(dst, src net.Conn, send bool) {
  if send {
    intercept(src, dst)
  }
  _, err := io.Copy(dst, src)
  if err != nil {
    log.Printf("connection error: %s", err.Error())
  }
}

const COM_QUERY = byte(0x03)

func intercept(src, dst net.Conn) {
  buffer := make([]byte, 4096)
  for {
    n, _ := src.Read(buffer)
    if n > 5 && buffer[4] == COM_QUERY {
      clientQuery := string(buffer[5:n])
      newQuery := strings.Replace(strings.ToLower(clientQuery), "from orders_v1", "from orders_v2", -1)
      fmt.Printf("client query: %s\n", clientQuery)
      fmt.Printf("server query: %s\n", newQuery)
      writeModifiedPacket(dst, buffer[:5], newQuery)
      continue
    }
    dst.Write(buffer[0:n])
  }
}

func writeModifiedPacket(dst net.Conn, header []byte, query string) {
  newBuffer := make([]byte, 5+len(query))
  copy(newBuffer, header)
  copy(newBuffer[5:], []byte(query))
  dst.Write(newBuffer)
}
```

This proxy listens on port 3307, intercepts SQL queries, and rewrites them before forwarding to MySQL.

[Read more...](https://packagemain.tech/p/the-developers-guide-to-database)