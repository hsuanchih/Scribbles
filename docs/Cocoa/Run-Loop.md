---
layout: default
title: Run-Loop
parent: Cocoa
---

# Run-Loop
---
## What's a Run-Loop?
A run-loop is an event processing mechanism that comes out-of-the-box with each thread. 
We can think of threads as checkout lanes at the supermarket, and a run-loop as the cashier behind the register.
The cashier is either busy checking out items we're unloading from our shopping cart, or otherwise resting if there are no customers.

Unlike a typical program that runs to the end and exits, 
our application stays active after launch and remains responsive to touch & network events - 
this is because we use an active run-loop to keep our application alive. 
Although every thread comes with a run-loop, for most threads the run-loop is inactive unless explicitly activated.
The only run loop that's active by default is the main run-loop - the run-loop of the main thread.

---
## Run-Loop Modes
The term mode here is a bit misleading with respect to what run-loop modes really are. 
A run-loop mode is a grouping of events needed to be processed by the run-loop. Several run-loop mode exists, but the 2 in particular that are of our interest:
* `RunLoop.Mode.tracking` - User interaction related events are grouped into this mode
* `RunLoop.Mode.default` - Everything else not user interaction related is grouped into this mode
A run-loop runs in one of these modes until after all events from the mode are processed.  
There are times when we need the run-loop to process run-loop events regardless of the mode it is in. 
A logical mode `RunLoop.Mode.common` caters this particular scenario.

---
## Run-Loop Event Sources
The run-loop receives events from 2 sources:
* Timer
* Input - there are 3 types of input sources
  * Port-Based Sources
  * Custom Input Sources
  * Cocoa Perform Selector Sources

<img src="images/runloop.jpg" height="280"/>

Image Source: [Apple](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW3)

---
## Timer Source
Let's see some practical examples with the timer source. One of the ways to schedule a timer would be as follows:
```swift
var count = 1
let timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { _ in
    guard count <= 5 else { return }
    print(count)
    count+=1
}
```
The `scheduledTimer(withTimeInterval:repeats:block:)` type method creates a timer and adds it to the main run-loop's `RunLoop.Mode.default` mode. What we'd see with adding a timer this way (apart from it not being accurate) is that user interaction will intefere with the timer being fired. This is because user interaction will keep the run-loop running in `RunLoop.Mode.tracking` mode, and hence the timer will not get a chance to service sources added to `RunLoop.Mode.default`.

A solution is to add the timer to the main run-loop's `RunLoop.Mode.common` mode. Recall that `RunLoop.Mode.common` is not a real mode for a run-loop to run in. When a source is added to a run-loop's `RunLoop.Mode.common`, this source is added to both the run-loop's `RunLoop.Mode.tracking` and `RunLoop.Mode.default` modes. Adding the timer this way will ensure that the timer gets fired regardless of the mode the run-loop is in.

```swift
// Running from the main thread
var count = 1

// Create a timer identical to the one we had before
let timer = Timer(timeInterval: 1, repeats: true) { _ in
    guard count <= 5 else { return }
    print(count)
    count+=1
}

// There's no direct way to get the run-loop of a specific thread
// from random places. The only way to get a thread's run-loop is
// by calling RunLoop.current from the thread - the main thread in
// this case.
RunLoop.current.add(timer, forMode: RunLoop.Mode.common)
```
We can also try adding a timer to a background thread's run-loop:

```swift
DispatchQueue.global(qos: .background).async {

    // Get a random thread from the thread pool and add a
    // timer to its run-loop
    var count = 1
    let timer = Timer(timeInterval: 1, repeats: true) { _ in
        guard count <= 5 else { return }
        print(count)
        count+=1
    }
    RunLoop.current.add(timer, forMode: RunLoop.Mode.default)
    print("Run loop activated")
}
```
`Run loop activated` get printed to the console, but the timer never fires. Why is that?

This is because the thread's run-loop has not been activated. Run-loops for all threads apart from the main thread are inactive by default. In this example, the thread accepts the timer, prints `Run loop activated` and is deallocated immediately. We'll need to activate the thread's run-loop for the added timer to take effect.

```swift
DispatchQueue.global(qos: .background).async {
    
    // Get a random thread from the thread pool and add a
    // timer to its run-loop
    var count = 1
    let timer = Timer(timeInterval: 1, repeats: true) { _ in
        guard count <= 5 else { return }
        print(count)
        count+=1
    }
    RunLoop.current.add(timer, forMode: RunLoop.Mode.default)
    
    // This is similar to the snippet from before, except now we activate
    // the thread's run-loop to kick-off the timer
    RunLoop.current.run()
    
    // This never gets called
    print("Run loop activated")
}
```

Now we have a firing timer, but `Run loop activated` doesn't get printed - this is because when we activate the run-loop, the run-loop loops forever. The print statement is never reached, and we now have a background thread that we'll never be able to deallocate. We can verify this retain cycle with the following setup:

```swift
// SomeThread lets us verify that the deallocation will never happen
class SomeThread : Thread {
    deinit {
        print("\(type(of: self)) \(#function)")
    }
}

// Again, here we add a timer to a background thread's run-loop.
// We then activate the thread's run-loop
var thread : SomeThread? = SomeThread {
    var count = 1
    let timer = Timer(timeInterval: 1, repeats: true) { _ in
        guard count <= 5 else { return }
        print(count)
        count+=1
    }
    RunLoop.current.add(timer, forMode: RunLoop.Mode.default)
    RunLoop.current.run()
}
thread?.start()

// Setting the thread to nil doesn't deallocate the thread.
thread = nil
```

And we see that once the run-loop on a thread is activated, the thread can't be deallocated - `SomeThread deinit` never gets printed to the console. So how would we add a timer to a thread's run-loop and not have the thread end up in a retain cycle? We don't let the run-loop loop forever. Instead, we use the run-loop's `run(until:)` instance method to enforce the run-loop's life time, and wrap this logic inside a loop we want to condition upon. This next snippet follows the same logic as what we saw earlier, except now we'll be able to break the retain cycle.

```swift
// SomeThread lets us verify that the deallocation will never happen
class SomeThread : Thread {
    deinit {
        print("\(type(of: self)) \(#function)")
    }
}

// We add a timer to a background thread's run-loop,
// but enforce the run-loop to run until 1 second post-current time
var thread : SomeThread? = SomeThread {
    var count = 1
    let timer = Timer(timeInterval: 1, repeats: true) { _ in
        print(count)
        count+=1
    }
    RunLoop.current.add(timer, forMode: RunLoop.Mode.default)
    
    // The run-loop will deactivate shortly after the timer counts to 5
    while count <= 5 {
        RunLoop.current.run(until: Date().addingTimeInterval(1))
    }
}
thread?.start()

// The thread can now be deallocated.
thread = nil
```
---
## Adding Observer to a Run-Loop
Apart from being able to add sources to a run-loop, we can optionally add observers to the run-loop. Let's add an observer to the run-loop and see the kind of information we can extract from the observer.

```swift
// Here we resort to Core Foundation for the purpose of adding an observer.
// We first create an observer to monitor all run-loop activities, and
// print to the console the current state of the run-loop.
let runLoopStateObserver = CFRunLoopObserverCreateWithHandler(
    kCFAllocatorDefault, 
    CFRunLoopActivity.allActivities.rawValue, 
    true, 0) { (observer, activity) in
    switch activity {
    case .entry:
        print("entry")
    case .beforeWaiting:
        print("beforeWaiting")
    case .afterWaiting:
        print("afterWaiting")
    case .beforeTimers:
        print("beforeTimers")
    case .beforeSources:
        print("beforeSources")
    case .exit:
        print("exit")
    default:
        print("\(activity)")
    }
}

// Now we add the observer to the current (main) run-loop in
// default mode
CFRunLoopAddObserver(
    CFRunLoopGetCurrent(),
    runLoopStateObserver,
    CFRunLoopMode.defaultMode
)
```

The print-out from the observer let's us visualize the order in which events are handled by the run-loop:
<img src="images/run-loop-observer.png" height="400"/>

Image Source: [Advanced Mac OS X Programming: The Big Nerd Ranch Guide by Mark Dalrymple., O'Reilly](https://www.oreilly.com/library/view/advanced-mac-os/9780321706560/ch15s06.html)

Apart from allowing us to better understand what events are handled when, what are observers good for? We can use observers inject logic to be performed by the run-loop on each iteration, thereby controlling what work and how much of it is done in one loop. 

For example, the system adds an observer to listen on the main run-loop's `CFRunLoopActivity.entry` to create the initial auto-release pool, and another observer to listen on the main run-loop's `CFRunLoopActivity.beforeWaiting` to flush the existing auto-release pool before starting anew - this same observer also listens for the main run-loop's `CFRunLoopActivity.exit` state to flush the auto-release pool for the last time when the main run-loop exits.

Another example would be to use the observer to control the amount of content rendering we want to perform in each pass of the run-loop. Recall that a run-loop can only run in one mode at a time - a one-off content rendering pass consuming a lot of time might end up blocking the run-loop from handling user interactive events, causing the user interaction to run prickly. We can use an observer to enforce incremental content render, thus preventing default mode from hijacking the run-loop.
