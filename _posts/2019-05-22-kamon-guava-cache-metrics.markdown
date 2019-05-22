---
layout: post
title:  "Kamon metrics for Guava Caches in Scala"
description: "My implementation for collecting Guava Cache metrics using Kamon"
image: assets/images/guava-kamon-metrics/main-pic.jpg
comments: true
date:   2019-05-22 10:00:00
tags: [kamon, kamon.io, guava, cache, metrics, scala]
---

After searching in the world wide web for metrics collecting for [Google Guava Cache](https://github.com/google/guava/wiki/CachesExplained),
all I could find are implementations with [Dropwizard Metrics](https://metrics.dropwizard.io/) in java.  
Don't take me wrong, dropwizard metrics system are one of the best there is. But what about scala?  

So here comes [Kamon.io](https://kamon.io/), an automatically instrument, monitor and debug distributed system (lots of fancy words).  
I couldn't find any example for metrics collecting for guava cache using Kamon, so I thought of sharing my implementation;
     
{% gist 505cba3c5e7c141d61511b8600b96dda %}

With that, using it is as simple as
 
{% highlight scala %}
object Main extends App {
  import KamonUtils._
  val cache = CacheBuilder.newBuilder()
    .removalListener(Kamon.newCacheMetricsRemovalListener("my-cache")) // optional
    .build[String, Int]()

  Kamon.metricsForCache("my-cache", cache)
}
{% endhighlight %}
