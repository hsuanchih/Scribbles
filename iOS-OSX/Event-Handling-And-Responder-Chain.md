# Event-Handling & Responders
---

We know that when we add a `button` to a `view controller` or a `view` and wire up a target-action, our action method ends up getting called when the user taps on the button. 

We're going into detail about how all of this happens.

---
## Event Generation & Handling

When a user taps on the phone's screen, a sequence of activities happen in the following order:
1. The finger creates a change in the electrostatic field at the location where the screen is touched, delivering an electric signal to the processor.
2. The operating system relays the event to the application's main event queue.
3. The event gets pulled off the event queue by the application's main run loop at some point, and is converted into an [UIEvent](https://developer.apple.com/documentation/uikit/uievent) object for dispatch.

![Main Event Loop](images/main-event-loop.jpg)

Image from [Apple](https://developer.apple.com/library/archive/documentation/General/Conceptual/Devpedia-CocoaApp/MainEventLoop.html#//apple_ref/doc/uid/TP40009071-CH18-SW1)
