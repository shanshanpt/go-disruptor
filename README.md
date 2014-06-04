Notes
=====

Disruptor Overview
----------------------------

This is a port of the [LMAX Disruptor](https://github.com/LMAX-Exchange/disruptor) into the Go programming language. It retains the essence and spirit of the Disruptor and utilizes a lot of the same abstractions and concepts, but does not maintain the same API.

On my MacBook Pro (Intel Core i7 3740QM @ 2.7 Ghz) using Go 1.2.x and Go 1.3, I was able to push over **900 million messages per second** (yes, you read that right) from one goroutine to another goroutine. The message being transferred between the two CPU cores was a simple, incrementing number, but literally could be anything. Note that your mileage may vary and that different operating systems can introduce significant “jitter” into the application by taking control of the CPU and invalidating the various CPU caches. Linux and Windows have the ability to assign a given process to specific CPU cores which reduces jitter significantly by keeping all the CPU caches hot.  Parenthetically, when the Disruptor code is compiled and run on a Nexus 5, it can push about 15-20 million messages per second.

Once initialized and running, one of the preeminent design considerations of the Disruptor is to process messages at a constant rate. It does this using two primary techniques. First, it avoids using locks at all costs which cause contention between CPU cores prevents true scalability. Secondly, it produces no garbage by allowing the application to preallocate sequential space on a ring buffer. By avoiding garbage, the need for a garbage collection and the stop-the-world application pauses introduced can be almost entirely avoided.

The current channel implementation maintains a big, fat lock around enqueue/dequeue operations and maxes out on the aforementioned hardware at about 25M messages per second for uncontended access-—more than an order of magnitude slower when compared to the Disruptor.  The same channel, when contended between OS threads (`GOMAXPROCS=2` or more) only pushes about 7 million messages per second.

Benchmarks
----------------------------
Each of the following benchmark tests sends an incrementing sequence message from one goroutine to another. The receiving goroutine asserts that the message is received is the expected incrementing sequence value. Any failures cause a panic. Unless otherwise noted, all tests were run using `GOMAXPROCS=2`.

* CPU: `Intel Core i7 3740QM @ 2.7 Ghz`
* Operation System: `OS X 10.9.2`
* Go Runtime: `Go 1.3-beta2`
* Go Architecture: `amd64`

Scenario | Per Operation Time
-------- | ------------------
Channels: Buffered, Blocking, GOMAXPROCS=1 | 58.6 ns/op
Channels: Buffered, Blocking, GOMAXPROCS=2 | 86.6 ns/op
Channels: Buffered, Blocking, GOMAXPROCS=3, Contended Write | 194 ns/op
Channels: Buffered, Non-blocking, GOMAXPROCS=1| 73.9 ns/op
Channels: Buffered, Non-blocking, GOMAXPROCS=2| 72.3 ns/op
Channels: Buffered, Non-blocking, GOMAXPROCS=3, Contended Write | 259 ns/op
Disruptor: Writer, Reserve One | 4.3 ns/op
Disruptor: Writer, Reserve Many | 1.1 ns/op
Disruptor: Writer, Await One | 3.5 ns/op
Disruptor: Writer, Await Many | 1.0 ns/op
Disruptor: SharedWriter, Reserve One | 13.6 ns/op
Disruptor: SharedWriter, Reserve Many | 2.5 ns/op
Disruptor: SharedWriter, Reserve One, Contended Write | 56.9 ns/op
Disruptor: SharedWriter, Reserve Many, Contended Write | 3.5 ns/op

When In Doubt, Use Channels
----------------------------
Despite Go channels being significantly slower than the Disruptor, channels should still be considered the easy, best, and most desirable choice for the vast majority of all use cases. The Disruptor's target use case is ultra-low latency environments where application response times are measured in nanoseconds and where stable, consistent latency is paramount and latency spikes cannot be tolerated.

Pre-Alpha
---------
This code is pre-Alpha stage and is not supported or recommended for production environments. That being said, it has been run non-stop for days without exposing any race conditions. Also, it does not yet contain any unit tests and is meant to be spike code to serve as a proof of concept that the Disruptor is, in fact possible, on the Go runtime despite some of the limits imposed by the [Go memory model](http://golang.org/ref/mem). The goal is to have an alpha release sometime in June 2014 and a series of beta releases during the months thereafter until we are satisfied. Following this, a release will be created and supported moving forward.

We are very interested to receive feedback on this project and how performance can be improved using subtle techniques such as additional cache line padding, memory alignment, utilizing a pointer vs a struct in a given location, replacing less optimal techniques with more optimal ones, especially in the performance critical paths of `Reserve`/`Commit` in the various `Writer`s and `Receive`/`Commit` in the `Reader`

Caveats
-------
One last caveat worth noting.  In the Java-based Disruptor implementation, a ring buffer is created, preallocated, and prepopulated with instances of the class which serve as the message type to be transferred between threads.  Because Go lacks generics, we have opted to not interact with ring buffers at all within the library code. This has the benefit of avoiding an unnecessary type conversion ("cast") during the receipt of a given message from type `interface{}` to a concrete type.  It also means that it is the responsibility of the application developer to create and populate their particular ring buffer during application wireup. Pre-populating the ring buffer at startup should ensure contiguous memory allocation for all items in the various ring buffer slots, whereas on-the-fly creation may introduce gaps in the memory allocation and subsequent CPU cache misses which introduce latency spikes.

The reference to the ring buffer can easily be scoped as a package-level variable. The reason for this is that any given application should have very few Disruptor instances. The instances are designed to be created at startup and stopped during shutdown. They are not typically meant to be created adhoc and passed around like channel instances. In any case, it is the responsibility of the application developer to manage references to the ring buffer instances such that the producer can push messages in and the consumers can receive messages out.

Vendoring and Dependency Management
-----------------------------------
Because the Disruptor will ultimately be library code and may itself be used by other libraries over which you have little to no control over the source code (e.g. logging libraries, etc.), it is highly recommended to vendor the Disruptor directly into your application or library workspace. The Disruptor will be maintained as a single package. Vendoring this package directly (once we reach a stable release) will allow the calling code to work with a specific and known version of the Disruptor. Without vendoring, it's very possible for another library or application to utilize the Disruptor but perhaps depend upon a slightly different version as compared to your code.  By vendoring, such a complex dependency chain is avoided because each library ultimately compiled into the application will have it's own, unique copy of the Disruptor source at the particular version it requires. Apart from copying the source code directly, it may be worthwhile to utilize a technique such as Git subtrees to enable vendoring in your specific environment.