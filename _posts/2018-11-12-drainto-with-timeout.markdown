---
layout: post
title:  "Java BlockingQueue With Timeout Option In drainTo Method"
summary: "My implementetion for drainTo with timeout method for BlockingQueue in Java usins a Semaphore"
date:   2018-11-12 01:00:00
comments: true
tags: [java, BlockingQueue, drainTo]
---

Lots of options for `drainTo` with timeout option are out there.
In my option, I used a [Semaphore](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Semaphore.html) to control the insertion and extraction of the [BlockingQueue](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/BlockingQueue.html).  
The main logic: for every insertion, first `add` to the Queue and then use `release` on the Semaphore and for every extraction, first `acquire` from the Semaphore and then `take` from the Queue. 

{% gist 6e49fe39458295d3e35a1eb62e7e43ce %}
_For the purpose of the Gist, I extend the `LinkedBlockingQueue` class and the override all the methods. Of course they are not all needed :)_
