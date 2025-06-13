---
title: Controller Orchestration Model in Controller Runtime Library
---



The `controller-runtime` library is utilized by numerous controller builders and templating engines ([Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder), [Operator SDK](https://github.com/operator-framework/operator-sdk) to name a few) to standardize the management of Kubernetes controllers. It handles the lifecycle of these controllers as well as webhooks, caches, servers and more while offering a fairly simple interface to build your operator on top of through it's `Reconciler`.

I've been curious about how it works internally and decided to start diving into how the `Manager` handles the life-cycle of all these runnables.

To clarify,`Runnable` is just a simple interface with a `Start` function which controllers, webhooks, caches and more all implement.

```go
type Runnable interface {
	Start(context.Context) error
}
```

Digging a bit deeper, you'll notice that the manager splits these runnables by functionality into `runnableGroups`, so that like-minded objects can be added, reconciled, and shutdown together. For instance, the way webhooks are handled internally is completely different from a leader elected controller, yet the life-cycle can be handled the same—they all need to start and stop. 

Each runnable spins off into it's own goroutine so that all of the runnables can do their jobs at the same time. The question then becomes, how exactly does the manager effectively orchestrate the lifetime of these runnables? Let's take a closer look:
#### Step by Step

1. The `Start` function starts by using first encapsulating the logic in a `sync.Once` callback called `startOnce`, to ensure the inside only runs one time.

	```go
	func (r *runnableGroup) Start(ctx context.Context) error {
		var retErr error
	
		r.startOnce.Do(func() {
            ...
	```

2. We kick off a goroutine with`r.reconcile` . This is the internal reconciler that kicks off all of the runnables. You'll see very shortly how it works.

	```go
	go r.reconcile()
	```

3. We attain the lock, mark the group as started and mark each runnable in the group as ready to start (Keep `signalReady` in mind, it'll come up later) and add it to the runnable dispatch channel called simply `ch`. 

	```go
	r.start.Lock()
	r.started = true
	for _, rn := range r.startQueue {
		rn.signalReady = true
		r.ch <- rn
	}
	r.start.Unlock()
	```

4. If there is nothing in the start queue, we simply return from `Start`. There's nothing to run, so there's nothing to do.

	```go
	if len(r.startQueue) == 0 {
		return
	}
	```

5. This next section involves coordination with the `reconcile` method which we recently started in it's own goroutine in step 2, so let's look at that method first. We start by reading off of the dispatch channel `ch` which we filled in step 3.

	```go
	func (r *runnableGroup) reconcile() {
		for runnable := range r.ch {
			...
	```

6. The first thing we do in the loop is very important. Part of the interesting bit to this logic is that the manager can support adding new runnables _after having already started_. With that in mind, we only want to add the runnable to the wait group if the manager is not in shutdown sequence. The reason is that executing `wg.Add` after `wg.Wait` is called will cause the program to panic.

   `wg.Wait` is a blocking call that waits for the wait group to decrement back to 0, so calling `wg.Add` after doesn't make any sense—hence the panic.

   So in the chance we are in shutdown, we simply continue and avoid adding the runner.

	```go
	{
		r.stop.RLock()
		if r.stopped {
			r.errChan <- errRunnableGroupStopped
			r.stop.RUnlock()
			continue
		}
		r.wg.Add(1)
		r.stop.RUnlock()
	}
	```

7. This is the portion that actually starts the individual runnable. There are some important aspects to look at. First, you'll notice this whole section is nested in a goroutine, that's because each runnable runs in parallel. 

   Another interesting bit is the nested goroutine. This block acts as a signaler back to the `Start` method, telling it "Hey, this runnable has started." It's put it it's own thread so that it's non-blocking to the actual start of the runnable. It signals by sending the runnable into the `startReadyCh` which the `Start` method is ready to receive from.

   Lastly, we make sure to `defer wg.Done()` to ensure that this runnable is correctly checked off in the shutdown process (decrementing that counter we spoke of previously).

	```go
	go func(rn *readyRunnable) {
		// Signal back to Start method
		go func() {
			if rn.Check(r.ctx) {
				if rn.signalReady {
					r.startReadyCh <- rn
				}
			}
		}()

		defer r.wg.Done()

		// Start the runnable
		if err := rn.Start(r.ctx); err != nil {
			r.errChan <- err
		}
	}(runnable)
	```

8. Going back to the `Start` method, we can see the relationship it has with `reconcile`.

	```go
	for {
		select {
		case <-ctx.Done():
			if err := ctx.Err(); !errors.Is(err, context.Canceled) {
				retErr = err
			}
		// Remove the runnable from the queue
		case rn := <-r.startReadyCh:
			for i, existing := range r.startQueue {
				if existing == rn {
					r.startQueue = append(r.startQueue[:i], r.startQueue[i+1:]...)
					break
				}
			}
			if len(r.startQueue) == 0 {
				return
			}
		}
	}
	```

   We read runnables off of the `startReadyCh` channel which sent the runnable down the pipe in step 7. We find it and eliminate it from the `startQueue`. Once the start queue is empty, we can finally quit out of the `Start` method because we've successfully launched all of our runnables! 
   
   Now, let's now see how the shutdown process handles these runnables when we decide to stop the manager.

9. Shutdown is signaled through the `StopAndWait` method. Once again we use `sync.Once` to execute the section exactly once.  

	```go
	func (r *runnableGroup) StopAndWait(ctx context.Context) {
		r.stopOnce.Do(func() {
			...
	```

10. We first `defer` the closing of the dispatch `ch` channel, we don't want any more runnables being added and spun up during the termination process.

	```go 
	defer func() {
		r.stop.Lock()
		close(r.ch)
		r.stop.Unlock()
	}()
	```

11. Next we internally call `Start`, this might seem backwards at first, but we trigger it here to ensure we kick off the reconcile loop and consume the `r.ch` channel. We want to clear any runnables in the queue and ensure the wait group is back to 0.

	```go
	_ = r.Start(ctx)
	r.stop.Lock()
	// Store the stopped variable so we don't accept any new
	// runnables for the time being.
	r.stopped = true
	r.stop.Unlock()
	```

12. The context is passed into all the runnables in step 7 line 12, so by calling `cancel` here, we are effectively signaling to all the runnables, "The shutdown process is in effect, please stop your reconcile loops", which causes those runnable loops to break and exit. 

	```go
	// Cancel the internal channel.
	r.cancel()
	```

13. Here we have an interesting concurrency pattern at play. This pattern let's us either wait for the wait group to complete, i.e, reach 0 or wait for a set timer to expire. This pattern is ideal because it achieves flexibility by creating multiple exit conditions.

	```go
	done := make(chan struct{})
	go func() {
		defer close(done)
		r.wg.Wait()
	}()

	select {
	case <-done:
		// We're done, exit.
	case <-ctx.Done():
		// Calling context has expired, exit.
	}
	```

And there you have it! The complete controller lifecycle orchestrated by the manager in controller-runtime. Feel free to view the code in it's entirety [here](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/manager/runnable_group.go).
