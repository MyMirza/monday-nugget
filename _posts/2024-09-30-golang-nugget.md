---
layout: post
title: "Golang Nugget - September 30, 2024"
date: 2024-09-30
categories: [golang]
excerpt_separator: <!--more-->
redirect_to: https://golangnugget.com/p/probabilistic-early-expiration-cache-stampedes-30-09-2024
---
Welcome to this week's edition of **Golang Nugget**, your go-to source for the latest insights and tips in the Go programming world.

In this issue, we dive into the concept of probabilistic early expiration in Go, a technique designed to tackle cache stampedes. These occur when multiple requests hit an empty cache simultaneously, leading to resource strain and potential service disruptions.

The article explores how to implement this method using Go, Redis, and a simple HTTP server. By leveraging the XFetch algorithm, cache values are updated based on a probability that increases as expiration nears, effectively spreading out cache updates and preventing simultaneous misses.

Stay tuned for more practical Go solutions and happy coding!
<!--more-->
### [Probabilistic Early Expiration in Go](https://dizzy.zone/2024/09/23/Probabilistic-Early-Expiration-in-Go/)

Cache stampedes occur when multiple parallel requests simultaneously miss a cache and independently load data from the source, leading to wasted resources and potential denial of service. The provided Go code demonstrates this issue using Redis for caching and a simple HTTP server. When the cache is empty, all incoming requests trigger a long-running operation, causing an initial spike in load. To mitigate this, the article explores probabilistic early expiration, based on the XFetch algorithm, which recomputes cache values based on a probability that increases as the cache expiry approaches. This method reduces the likelihood of simultaneous cache misses by updating the cache value early. The implementation involves wrapping the cached data with metadata, including the expiry time and computation duration, and using a probabilistic function to decide whether to update the cache. The results show that this technique effectively prevents cache stampedes by spreading out cache updates over time.

```go
func (pv probabilisticValue) shouldUpdate() bool {
    beta := 1.0
    now := time.Now()
    scaledGap := pv.Delta.Seconds() * beta * math.Log(rand.Float64())
    return now.Sub(pv.Expiry).Seconds() >= scaledGap
}
```

[Read more...](https://dizzy.zone/2024/09/23/Probabilistic-Early-Expiration-in-Go/)
{% include mailerlite_golang_nugget_postpage.html %}