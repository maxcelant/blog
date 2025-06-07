---
title: Controller Orchestration Model in Kubernetes Controller Runtime
---
The manager handles the life-cycle of all the runnable objects created by controller-runtime. To clarify,`Runnable` is just a simple interface with a `Start(context.Context) error` function. The manager splits these runnables by functionality into `runnableGroups`, so that like-minded objects can be added, reconciled, and shutdown together. Each runnable spins off into it's own goroutine so that all of the runnables can be orchestrated by the manager. The question then becomes, how exactly does the manager effectively orchestrate the lifecycle of one of these runnable groups? Let's take a closer look:
#### Step by Step
1. The `Start` function starts by using first encapsulating the logic in a `sync.Once` callback called `startOnce`, to ensure the inside only runs one time.

	```go
	func (r *runnableGroup) Start(ctx context.Context) error {
		var retErr error
	
		r.startOnce.Do(func() {
            ...
	```

2. We kick off a goroutine with`r.reconcile()` .

	```go
			go r.reconcile()
	```

3. We attain the lock, mark the group as started and mark each runnable in the group as `signalReady=true` (this will come up later) and add it to the runnable dispatch channel.

	```go
			r.start.Lock()
			r.started = true
			for _, rn := range r.startQueue {
				rn.signalReady = true
				r.ch <- rn
			}
			r.start.Unlock()
	```

4. If there is nothing in the start queue, we simply return.

	```go
			if len(r.startQueue) == 0 {
				return
			}
	```

5. This next section involves coordination with the `reconcile` function, so let's look at that first. We start by reading off of the dispatch channel which we filled in step 3.

	```go
	func (r *runnableGroup) reconcile() {
		for runnable := range r.ch {
			...
	```

6. Part of the interesting bit to this logic is that the manager can support adding new runnables _after having already started_. This portion is key, we only want to add the runnable to the wait group if the manager is not in shutdown sequence. The reason is that executing `wg.Add` after `wg.Wait` is called will cause the program to panic.

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

7. This is the portion that actually starts the individual runner. The are some important aspects. First, you'll notice this whole area is nested in a goroutine, that's because each runnable runs on its own. Another key is the goroutine nested within that, which is used to signal back to the `Start` method (I told you it would come back up!), telling it "Hey, this runnable has officially started!". 

   Lastly, we make sure to `defer wg.Done()` to ensure that this runnable is correctly marked off in the shutdown process (decrementing that counter we spoke of earlier).

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

   We read runnables off of the start ready queue which we triggered in step 7 in it's own goroutine. We find it and eliminate it from the start queue. Once the start queue is empty, we can finally quit out of the `Start` method because we've successfully kicked off the lifecycle of all our runnables! Let's now see how the shutdown process handles these runnables.


9. Shutdown is signaled through the `StopAndWait` method. Once again we use `sync.Once` to execute the section exactly once.  

	```go
	func (r *runnableGroup) StopAndWait(ctx context.Context) {
		r.stopOnce.Do(func() {
			...
	```

10. We first `defer` the closing of the dispatch channel, we don't want any more runnables being added and spun up. 

	```go 
			defer func() {
				r.stop.Lock()
				close(r.ch)
				r.stop.Unlock()
			}()
	```

11. Next we internally call `Start`, this might seem backwards at first but we call it here to make sure we kick off the reconcile loop and consume the `r.ch` channel. We want to make sure that any runnables stuck in the queue are consumed and our `wg` can be reduced down to 0. 

	```go
			_ = r.Start(ctx)
			r.stop.Lock()
			// Store the stopped variable so we don't accept any new
			// runnables for the time being.
			r.stopped = true
			r.stop.Unlock()
	```

12. The context is passed into all the runnables in step 7 line 12, so by calling `cancel` here, we are effectively signaling to all the runnables, "Hey guys, the shutdown process is in effect!", which causes those loops to break. 

	```go
			// Cancel the internal channel.
			r.cancel()
	```

13. Here we have an interesting concurrency pattern at play. This pattern let's us either wait for the wait group to complete, i.e, reach 0 or wait for a set timer to expire. This is nice because it gives us flexibility in a non-blocking fashion. We don't want to wait forever for the runnables to be cleaned up in situations where maybe some block occurred.

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
		})
	}
	```

And there you have it! The complete controller lifecycle orchestrated by the manager in controller-runtime. Feel free to view the code in it's entirety [here](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/manager/runnable_group.go).
