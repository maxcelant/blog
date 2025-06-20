---
title: 100 Go Mistakes and How to Avoid Them
---

###### 21. Incorrectly initializing slices
If you know the length of your new slice, then you should set an initial capacity for it. If the allocation of the new slice is conditional, it's up to you whether you want to use capacity.

###### 22. Being confused about nil vs. empty slices
A nil slice requires no allocation if you're not sure about the length of resulting slice or whether there are any elements at all then you should make it a nil slice. If you do know the length then you should use `make`.
```go
var s []string   // nil slice
var s []string{} // empty slice
```
###### 31. Ignoring how arguments are evaluated in range loops
When using a range-based for loop, it'll make a copy of the type at the beginning of the loop. However, if you use a classic for loop, then the expression is evaluated on every iteration.
###### 32. Ignoring the impact of using pointer elements in range loops
For a range-based for loop that uses pointers, the loop variable is reused on each iteration, meaning it holds a new value each iteration but keeps the same memory address.

If you try to store the address in a slice or a map, all the stored pointers will reference the same final element.
###### 33. Making wrong assumptions during map iterations
If you are adding entries to a map while iterating, they may or may not appear during that iteration. Create a copy of the map, so you can use one to iterate and one to update. 
###### 34. Ignoring how break statements work
If you have a switch statement in a for loop with a `break`, it'll break the switch but not the for loop. To fix this you can use labels.

```go
loop:
  for i := 0; i < 5; i++ { 
    fmt.Printf("%d ", i) 
    switch i { 
      default:
      case 2:
        break loop 
    } 
  }
```

###### 35. Using `defer` in loops
`defer` only executes after function is ended so if you put in a loop it will never trigger, causing leaks. The better approach is to create a function that triggers on every iteration of the loop that calls `defer` at the end.
###### 36. Not understanding how runes work
The `len` built-in function applied on a string doesn’t return the number of characters; it returns the number of bytes. We can compose a string with an array of bytes

```go
s := string([]byte{0xE6, 0xB1, 0x89}) 
fmt.Printf("%s\n", s) // prints 汉
```

###### 38. Misusing trim functions
 `TrimRight` uses a set of values to determine removal. `TrimSuffix` just removes the given string from the end. `Trim` removes the given set of letters from both sides.
###### 39. Under optimized string concatenation
Don't use `+=` to build your strings, use `strings.Builder` instead.
###### 40. Useless string conversions
Most I/O is done with `[]byte`, not strings. Don't do unnecessary conversions by using strings. Most functions available in the `strings` package are also available in the `bytes` package.
###### 41. Strings and memory leak
Creating a substring with slice syntax will create a copy of the entire backing string which can cause large strings to be copied in memory.
###### 42. Not knowing which type of receiver to use
Use a pointer receiver if you know you're gonna mutate the receiver or if it contains a field that cannot be copied or if it's a large object. Use a value receiver if you want to enforce immutability, the receiver type is a map, function, or channel, or the receiver is a slice that doesn't need to be mutated, or it's a primitive type.
###### 43. Named Return Values
Good for interfaces and short functions. Should be avoided in long functions because the empty return obscured readability 
###### 45. Returning a Nil Receivers
Nil pointers are valid receivers because receivers are just syntax sugar on top of the first type in the argument list. In the example that we return an interface from a function, even though we return a nil struct, the interface will be not nil because it's a wrapper around the struct. And the struct is something, it's not nothing, even though the value is nil.

###### 46. Using a filename as a function input
Instead of passing in a file name for a function that does some read operation, pass in `io.Reader` interface instead because that allows you to abstract the function and use it for files, HTTP requests, and much more. It also makes testing a lot easier.
###### 47. Ignoring how defer arguments and receivers are evaluated
Inputs of a `defer` function are evaluated immediately upon the function being seen, not when it actually executes. So to ensure you have the correct value, whenever it executes, you should use either closure or use a pointer to the value.

`defer` acts differently whether you are using a pointer receiver or just a plain receiver. With a pointer receiver, it'll get the most up-to-date value at the end of the functions execution, but as for the normal receiver, it'll get whatever the value is evaluated at when the `defer` function is called.
###### 50. Checking an error type incorrectly
Use `errors.As` to check if the type of a wrapped error exists because using a `switch` statement on an typed error that's wrapped won't work.

**Note:** `errors.As` is to find a type (custom struct). `errors.Is` is to find a error value (sentinel).
###### 51. Checking an error value inaccurately
Sentinel errors are created using `errors.New` and are used for known errors. Think of something like `ErrFileNotFound`. 

Sentinel errors are not meant to be wrapped, because it'll obscure the type if you do so. If you do wrap it, you can still find it nested by using `errors.Is`.
###### 52. Handling errors twice
Use wrapping to add information to errors before returning.
###### 53. Not handling an error
Do `_ = f()` if you decide to not handle an error.
###### 54. Not handling defer errors
You can handle defer errors by using named return values.
###### 55. Understanding the difference between concurrency and parallelism
Concurrency provides a structure to solve a problem with parts that may be parallelized. Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once. Concurrency is about structure, and we can change a sequential implementation into a concurrent one by introducing different steps that separate concurrent threads can tackle.
###### 56. Thinking concurrency is always faster
OS Threads are switched on and off CPU cores, Goroutines are switched on and off OS thread by the Go runtime. Go scheduler uses GMP (Goroutine, Machine {OS thread}, Processor {core}) terminology.

Goroutine runs a OS thread which is assigned to a CPU core.`GOMAXPROCS` is the max OS threads used to run user-level code. 

So if we have 4 cores, then our goroutines will be scheduled among 4 OS threads using the Go scheduler. Go scheduler uses a global queue to assign goroutines to threads. There's also a local queue for each processor. Goroutines aren't efficient for handling minimal workloads.
###### 57. Being puzzled about when to use mutexes and channels
Channels should be used for concurrent goroutines. Mutexes are used for parallel goroutines. Parallel goroutines usually share and mutate a shared resource, which they need to lock to ensure safe changes (via mutexes). Concurrent goroutines usually involves transferring ownership of some resource from one step to another (via channels).
###### 58. Not understanding race problems
Data race is when two threads try update the same memory location at the same time. Also if one goroutine is writing and the other is reading, that's still a data race. An atomic operation can’t be interrupted, thus preventing two accesses at the same time.

Another option is to communicate through a channel. Using mutex is another option. A race condition occurs when the behavior depends on the sequence or the timing of events that can’t be controlled. Channels solve this by orchestrating and coordinating the order things should occur.
###### 59. Not understanding the concurrency impacts of a workload type
If the workload is CPU-bound, a best practice is to rely on `GOMAXPROCS`. `GOMAXPROCS` is a variable that sets the number of OS threads allocated to running goroutines.

The go scheduler is lazy and efficient. It avoids uselessly starting up OS threads if it can because that introduces latency. If a goroutine is blocking (for I/O) or CPU intensive, then it may start a new OS thread, but otherwise it'll avoid it and use its context stealing model. 

###### 60. Misunderstanding Go contexts
Using context as a deadline. `context.WithTimeout(ctx, time)` returns a `context, cancel`. Behind the scenes, context spins up a goroutine that will be retained in memory for `time` seconds or until `cancel` is called. 

Context is used to send a cancellation signal to a goroutine by using `WithCancel`. When the main calls the `cancel` func is called, it triggers the ending of the goroutine using the context. 

Using `context.WithValue`, you can pass down information to handlers and functions without explicitly creating variables. This can be good for trace ids.

When you close a channel, it immediately unblocks. An open channel with no value will block until it gets some content it can do something with. In a buffered channel, sending is blocked when its full and receiving is blocked when its empty.

`ctx.Done()` unblocks when it's closed, which acts as a signal when the channel is cancelled. 

We can use `ctx.Err()` to see what the reason for the cancellation was.

In general, a function that users wait for should take a context, as doing so allows upstream callers to decide when calling this function should be aborted.
###### 61. Propagating an inappropriate context
We should be cautious about propagating the context because if the context is cancelled in the parent goroutine, it'll also be cancelled for the child even in scenarios where the child may have not finished its task yet! 

In most cases, creating a new context is preferred. Keep in mind, you can also create your own context wrapper to have the functionality you want because `Context` is an interface.

###### 62. Starting a goroutine without knowing how to stop it
Starting a goroutine without knowing when to stop it is a design issue. Using a context to cancel a goroutine didn't necessarily mean that the parent goroutine won't finish up before the child goroutine is fine cleaning up it's resources. 

Having some way to `defer` the clean up of the child goroutine when the parent is ending is best practice. 
###### 63. Mishandling goroutines with loop variables.
**Note**: this may be deprecated now. 

If you spin up a goroutine in a loop and use a closured iterable from outside, the value at time of goroutine execution was non-deterministic. We can fix this by passing the variable into the goroutine as a param.
###### 64. Expecting deterministic behavior with select and channels
If multiple communications in a `for-select` statement can succeed, they are processed in a random order. This is to avoid starvation. where one channel has way more messages than another so the other channels are not able to receive.

`for-select` statements will run indefinitely until you explicitly `break` out of them.
In cases where you're receiving from multiple channels in a `for-select` and one of them is a disconnect channel, you might want a inner for select statement for receiving the rest of the messages before closing. you would use a default case once all the messages are handled to break out.

We must remember that if multiple options are possible, the first case in the source order does not automatically win.

```go title:"For-select channel example" fold
func main() {
	msgCh := make(chan int, 5)
	cancelCh := make(chan any)

	go func() {
		for i := range 10 {
			msgCh <- i
		}
		cancelCh <- struct{}{}
	}()

Loop:
	for {
		select {
		case v := <-msgCh:
			fmt.Println(v)
		case <-cancelCh:
			for {
				select {
				case v := <-msgCh:
					fmt.Println("closing out ", v)
				default:
					break Loop
				}
			}
		}
	}
}

```

###### 65. Not using notification channels
If you want to convey a signal or notification through a channel, use a `chan struct{}` , which sends no data. 
###### 66. Not using nil channels
Closing a channel is non-blocking.  You can set a channel to `nil` and not have to worry about wasting resources reading from a channel that is closed. Good for scenarios in which you are reading from two channels simultaneously, each with their own varying length. You can see if a channel is open or closed with its second return value:`v, open := <- ch`.
###### 67. Being puzzled about channel size
Unbuffered channels (also called synchronous channels) will block the send until the receive end is ready. 

Buffered channels are harder to work with, since the queue size is unblocking until it reaches full capacity, sometimes leading to weird deadlock issues. You should think deeply about your buffered channel capacity, start with a value of 1.

A good example is the worker pool, where goroutines need to send data to a shared channel. In that case, make the capacity equal to the threads.

Queues are typically always close to full or close to empty due to the differences in pace between consumers and producers. They very rarely operate in a balanced middle ground where the rate of production and consumption is evenly matched.
###### 68. Forgetting about possible side effects with string formatting
The following issues are only important in concurrent programs.

 Context can hold values with `WithValues`. Those values can be pointers to structs, which means they can be mutated after being added.  Doing a `fmt.Sprintf("%v", ctx)` will recursively traverse the object and print it's contents. 

If we mutate the struct during the `Sprintf`, this will cause a data race to trigger. We are reading and writing the same memory concurrently without synchronization — which is undefined behavior in Go.
Sure. Here's the content reformatted into paragraph form, keeping all the original meaning intact:

###### 69. Creating data races with append
When you hit the capacity of a slice, the backing array is doubled in size and the elements are copied over. If you try to `append` to a slice on two goroutines simultaneously, it creates a race condition if there is remaining length, such as with `make([]int, 0, 1)`. Since there's one spot remaining, both goroutines trying to append leads to a race condition. However, `make([]int, 1)` avoids this because any append triggers a new backing array. If you want to add an element to an existing slice and then work with it in a goroutine, make a copy of the original using `copy`.

######  70. Using mutexes inaccurately with slices and maps
Shallow copies of slices and maps can cause data races when you create a copy of that data structure and modify it while another goroutine reads it. This is due to shared backing data structures. The correct approach is to mutex-lock the entire function interacting with that data structure or create a deep copy to prevent race conditions.

######  71. Misusing sync.WaitGroup
`wg.Wait()` blocks until the counter reaches 0. A common mistake is calling `wg.Add` inside the child goroutine, which is unsafe since there's no guarantee it will be added before reaching `wg.Wait` in the parent. Always call `wg.Add` in the parent goroutine before launching children.

######  72. Forgetting about sync.Cond
When multiple goroutines receive from the same channel, only one will get the value. Channels are good for many-to-one communication, not one-to-many. In scenarios where one goroutine sends events to many listeners, `sync.Cond` is useful. `cond.Wait()` blocks until a `cond.Broadcast()` awakens it, and it also synchronizes with a shared mutex among listeners and the sender.

###### 73. Not using errgroup
`errgroup` is great for parallel tasks where you want to catch and return the first error. You use it by spinning up goroutines with `g.Go(fn)` and calling `err := g.Wait()` to wait for completion or error. Functions launched via `g.Go` must be context-aware; otherwise, cancelling the context won't stop them.

###### 74. Copying the sync types
Never use value receivers with sync primitives like `sync.Mutex`. Copying these types (e.g., by using value receivers or assigning structs directly) causes data races. Instead, use pointer receivers or make the mutex itself a pointer to ensure shared access.

######  75. Providing wrong time duration  
Always provide an `int64` type alongside a `time.Duration`. Since `time.Duration` represents nanoseconds, writing something like `1000 * time.Second` ensures the correct duration. Misunderstanding this can lead to incorrect timing behavior.

###### 76. time.After and memory leaks
Don't use `time.After()` inside a `for-select` loop—it causes memory leaks by creating new channels each loop iteration without releasing them. Instead, use `time.Timer` and call `t.Reset()` at the start of each loop iteration or use a context with timeout. Be cautious: timers leak resources until they expire or are garbage collected.

######  77. Common JSON handling mistakes
The `json.Marshaler` interface allows structs to customize marshalling. When you embed a type that implements it, the wrapper also implements it, which may cause unexpected behavior. Also, `time.Time` includes wall clock and monotonic time. When unmarshalled, only the wall clock is restored. This mismatch means the structs differ pre- and post-marshalling. Use `time.Equal` to compare them safely.
######  79. Not closing transient resources
Always close `http.Response.Body`, which implements `ReadCloser`, using `defer` to avoid memory leaks. No need for an `if resp != nil` check—just ensure you only defer the close if there was no error during the request. Any type implementing `io.Closer` should be closed properly.

######  91. Not understanding CPU caches
Modern CPUs have three cache levels: L1, L2, and L3, each larger and slower than the last. L3 is around 10× slower than L1. CPUs have physical and logical cores—logical cores share parts like FPUs but not ALUs, akin to chefs sharing a kitchen. A cache line is 64 bytes (e.g., 8 ints), so storing data contiguously improves access speed. 

How your code accesses memory (striding) matters: unit stride (e.g., arrays) is fastest, constant stride is predictable but slower, and non-unit (e.g., linked lists) is worst. Caches are partitioned; for example, a 2-way set-associative cache with 512 blocks has 256 sets. Placement is based on memory address, with a set index, tag bits, and block offset determining cache behavior. Poor striding can lead to only one set being used, harming performance through conflict misses.

######  92. Writing concurrent code that leads to false sharing
When two goroutines access variables in the same cache line and at least one writes, it causes false sharing. Cores mistakenly think they're sharing data, leading to unnecessary cache invalidation and coherence traffic, reducing performance even though no logical sharing exists.

######  93. Not taking into account instruction level parallelism
Instruction Level Parallelism (ILP) allows CPUs to execute independent instructions in parallel. Hazards occur when instructions depend on each other—like `C = A + B; E = C + D`—which must execute sequentially. Structuring code to reduce such dependencies allows the CPU to better utilize ILP.

######  94. Not being aware of data alignment
A word is 8 bytes, and CPUs prefer aligned access to word-sized chunks. Compilers insert padding to maintain alignment. For example, a 1-byte variable followed by an 8-byte one results in 7 bytes of padding. Organize struct fields to minimize padding—this optimizes for cache locality and GC efficiency.

###### 95. Not understanding stack vs heap
The stack allocates memory at compile time and is goroutine-local. If a function returns a local variable’s address, the variable escapes to the heap. The compiler uses escape analysis to decide where to allocate. "Sharing down" (passing to children) usually stays on the stack, while "sharing up" (returned or referenced after) forces heap allocation.

###### 96. Not knowing how to reduce allocations
Use `-gcflags` to inspect escape analysis decisions. Design APIs to favor sharing down. For example, `io.Read` lets the caller reuse buffers, avoiding heap allocations. Also, inlining simple expressions helps avoid heap allocations—like `string(foo)` used inline. Use `sync.Pool` to reuse objects across goroutines when allocation cost is high.

######  97. Not relying on inlining
Go inlines functions that fall below a complexity threshold. Mid-stack inlining enables the compiler to optimize deeper call stacks. Inlining removes call overhead and can improve escape analysis. You can improve performance by isolating slow paths into separate functions and letting hot paths benefit from inlining.

###### 98. Not using Go diagnostic tools
```
$ go tool pprof -http=:8080 <file>
```

###### 99. Not understanding how the garbage collector works 
The garbage collector runs concurrently with the application and causes some"stop the world" moments that will impact the performance of your code. 

Usually the GC will run every time a certain heap threshold is reached.  `GOGC` is set to `100`, which is 128 MB, and it can be updated. If your application has high spikes of traffic, it might be worth messing with this value. 
