# Run-Loop
---
## What's a Run-Loop?
A run-loop is an event processing mechanism that comes out-of-the-box with each thread. 
We can think of threads as checkout lanes at the supermarket, and a run-loop as the cashier behind the register.
The cashier is either busy checking out items we're unloading from our shopping cart, and otherwise resting if there are no customers.

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
* Input - there are 3 types of input sources
  * Port-Based Sources
  * Custom Input Sources
  * Cocoa Perform Selector Sources
* Timer

<img src="images/runloop.jpg" height="280"/>
