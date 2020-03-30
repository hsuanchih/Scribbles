# Event-Handling & Responders
---

We know that when we add a [button](https://developer.apple.com/documentation/uikit/uibutton) to a [view controller](https://developer.apple.com/documentation/uikit/uiviewcontroller)'s [view](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621460-view) or some [view](https://developer.apple.com/documentation/uikit/uiview) and wire up a [target-action](https://developer.apple.com/library/archive/documentation/General/Conceptual/Devpedia-CocoaApp/TargetAction.html#//apple_ref/doc/uid/TP40009071-CH3), our action method ends up getting called when the user taps on the button. 

We're going into detail about how all of this happens.

---
## Event Generation & Delivery

When a user taps on the phone's screen, a sequence of activities happen in the following order:
1. The finger creates a change in the electrostatic field at the location where the screen is touched, delivering an electric signal to the processor.

2. The operating system relays the event to the application's main event queue.

3. The event gets pulled off the event queue by the application's main run loop at some point, and is converted into an [UITouch](https://developer.apple.com/documentation/uikit/uitouch) object wrapped in an [UIEvent](https://developer.apple.com/documentation/uikit/uievent) object for dispatch.</br></br>

![Main Event Loop](images/main-event-loop.jpg)

Image from [Apple](https://developer.apple.com/library/archive/documentation/General/Conceptual/Devpedia-CocoaApp/MainEventLoop.html#//apple_ref/doc/uid/TP40009071-CH18-SW1)</br></br>


4. This event object travels down the dispatch hierarchy following path:

__Application__ ---[`sendEvent(_:)`](https://developer.apple.com/documentation/uikit/uiapplication/1623043-sendevent)---> __Key Window__ ---[`sendEvent(_:)`](https://developer.apple.com/documentation/uikit/uiwindow/1621614-sendevent)---> __View__ ---[`hitTest(_:with:)`](https://developer.apple.com/documentation/uikit/uiview/1622469-hittest)---> __Subviews__</br></br>

![Event Delivery](images/event-delivery.jpg)

Image from [Apple](https://developer.apple.com/library/archive/documentation/General/Conceptual/Devpedia-CocoaApp/EventHandlingiPhone.html#//apple_ref/doc/uid/TP40009071-CH13-SW1)

---
## Finding the Event's Intended Recipient

The purpose of event delivery is to forward the event to the view that is under the touch. To do that, the application hit-tests subviews under the `view controller`'s view and their subviews until it finds the intended recipient.

```Swift
class UIView : UIResponder {
    .
    .
    
    func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
        
        // If any of the following conditions are true,
        // then myself, my subviews and their subviews cannot be the intended recipient of the touch event:
        // * I'm not user interactive, or
        // * I'm hidden, or
        // * I'm transparent, or
        // * My rect doesn't contain the touch point
        guard isUserInteractionEnabled && !isHidden && alpha > 0.01 && self.point(inside: point, with: event) else {
            return nil
        }
        
        // Otherwise, recursively call hitTest on my subviews & their subviews to find the recipient of the touch event
        for subview in subviews {
            if let view = subview.hitTest(convert(point, to: subview), with: event) {
                return view
            }
        }
        
        // If none of my subviews meet the conditions to be the recipient of the touch event, 
        // then I must be the most appropriate recipient of the event 
        return self
    }
}
```

In our case, the intended event recipient is the [button](https://developer.apple.com/documentation/uikit/uibutton) we've added to our [view controller](https://developer.apple.com/documentation/uikit/uiviewcontroller)'s [view](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621460-view).

---
## Handling the Event

The event recipient can either choose to handle the event or not handle the event. In the case where the recipient wishes to handle the event, it does so by implementing any one of the following [`UIResponder`](https://developer.apple.com/documentation/uikit/uiresponder) methods (for practical reasons, however, it probably implements most of them):
* [`func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?)`](https://developer.apple.com/documentation/uikit/uiresponder/1621142-touchesbegan)
* [`func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?)`](https://developer.apple.com/documentation/uikit/uiresponder/1621107-touchesmoved)
* [`func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?)`](https://developer.apple.com/documentation/uikit/uiresponder/1621084-touchesended)
* [`func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?)`](https://developer.apple.com/documentation/uikit/uiresponder/1621116-touchescancelled)

Our [button](https://developer.apple.com/documentation/uikit/uibutton) implements all of them, and our action method will be invoked depending on the [control event(s)](https://developer.apple.com/documentation/uikit/uicontrol/event) we've wired our [target-action](https://developer.apple.com/library/archive/documentation/General/Conceptual/Devpedia-CocoaApp/TargetAction.html#//apple_ref/doc/uid/TP40009071-CH3) for, that is, when the button calls [`UIControl`](https://developer.apple.com/documentation/uikit/uicontrol)'s [`sendAction(_:to:for:)`](https://developer.apple.com/documentation/uikit/uicontrol/1618237-sendaction) method on its registered targets and action methods.

```Swift
class UIButton : UIControl {

    .
    .
    // A simplified touchesBegan implementation for UIButton to trigger target-action
    override open func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
    
        // Do a bunch of other stuff here
        .
        .
        // Trigger target-action
        sendActions(for: .touchDown)
    }
    
    // A simplified touchesEnded implementation for UIButton to trigger target-action
    override open func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
    
        // Do a bunch of other stuff here
        .
        .
        // Trigger target-action
        switch touches.first?.location(in: self) {
        case let .some(touchPoint) where hitTest(touchPoint, with: event) != nil:
            sendActions(for: .touchUpInside)
        case let .some(touchPoint) where hitTest(touchPoint, with: event) == nil:
            sendActions(for: .touchUpOutside)
        // Handle some other cases here
        .
        .
        default:
            break
        }
    }
}
```
