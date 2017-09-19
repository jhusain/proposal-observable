# Extending EventTarget with Observable

EventTarget has a variety of well-known issues which impact ergonomics. EventTargets have unergonomic unsubscription semantics, and are difficult to compose. This proposal attempts to address both of these concerns by introducing an ObservableEventTarget interface which integrates Observable into the DOM.

## ObservableEventTarget interface

The `ObservableEventTarget` interface inherits from `EventTarget` and introduces a new method: `on`. The `on` method creates an`Observable` and forwards events dispatched to the `ObservableEventTarget` to the Observers of that Observable.

```
interface Event { /* https://dom.spec.whatwg.org/#event */ }

dictionary OnOptions {  
  // listen for multiple events on the EventTarget,
  // and send it to each Observer's next method
  sequence<DOMString> nextOn = [];
  
  // listen for multiple events on the EventTarget,
  // and send the first one to the Observer's error method
  sequence<DOMString> errorOn = [];

  // listen for multiple events on the EventTarget,
  // and invoke each Observer's complete method when
  // one of these events is received.
  sequence<DOMString> completeAfter = [];
  
  // member indicates that the callback will not cancel
  // the event by invoking preventDefault().
  boolean passive = false;

  // handler function which can optionally execute stateful
  // actions on the event before the event is dispatched to
  // Observers (ex. event.preventDefault()).
  EventHandler handler = null;

  // member indicates that the Observable will complete after
  // one event is dispatched.
  boolean once = false;
}

interface ObservableEventTarget extends EventTarget {
  Observable<Event> on(DOMString type, OnOptions options);
  Observable<Event> on(OnOptions options);
}
```

Any implementation of `EventTarget` can also implement the `ObservableEventTarget` interface to enable instances to be adaptated to `Observable`s.

## Design Considerations

The semantics of `EventTarget`'s and `Observable`'s subscription APIs overlap cleanly. Both share the following semantics...

* the ability to synchronously subscribe and unsubscribe from notifications
* the ability to synchronously dispatch notifications
* errors thrown from notification handlers are caught and reported to the host.

`EventTarget`s have semantics which control the way events are propagated through the DOM. The `on` method accepts an `OnOptions` dictionary object which allow event propagation semantics to be specified when the ObservableEventTarget is adapted to an Observable. The `OnOptions` dictionary extends the DOM's `AddEventListenerOptions` dictionary object and adds four additional fields:

1. `handler`
2. `nextOn`
3. `errorOn`
4. `completeAfter`


### The  `OnOptions` `handler` member

The `handler` callback function is invoked on the event object prior to the event being dispatched to the Observable's Observers. The handler gives developers the ability execute stateful operations on the Event object (ex. `preventDefault`, `stopPropagation`),  within the same tick on the event loop as the event is received.

In the example below, event composition is used build a drag method for a button to allow it to be absolutely positioned in an online WYSWYG editor. Note that the `handler` member of the `OnOptions` object is set to a function which prevents the host browser from initiating its default action. This ensures that the button does not appear pressed when it is being dragged around the design surface.

```js
import "_" from "lodash-for-observable";

const button  = document.querySelector("#button");
const surface = document.querySelector("#surface");

// invoke preventDefault() in handler to suppresses the browser default action
// which is to depress the button.
const opts = { handler(e) { e.preventDefault(); } };
const mouseUps   = _(button.on( "mouseup",   opts));
const mouseMoves = _(surface.on("mousemove", opts));
const mouseDowns = _(surface.on("mousedown", opts));

const mouseDrags = mouseDowns.flatMap(() => mouseMoves.takeUntil(mouseUps));

mouseDrags.subscribe({
  next(e) {
    button.style.top = e.offsetX;
    button.style.left = e.offsetY;
  }
})
```
### The `OnOptions` `nextOn` member

The `nextOn` member contains an Array of event types which should be passed to the `next` method of the Observable's Observers. The `nextOn` member is only used if no type string is provided to the `ObservableEventTarget.prototype.on` method.

### The  `OnOptions` `errorOn` member

The `errorOn` member is an array of event types which should be passed to the  `error` method on the Observable's Observers.

In the example below the  `on` method is used to create an `Observable` which dispatches an Image's "load" event to its observers. Setting the `"once"` member of the `OnOptions` dictionary to `true` results in a `complete`  notification being dispatched to the observers immediately afterwards. Once an Observer has been dispatched a `complete` notification, it is unsubscribed from the Observable and consequently the `ObservableEventTarget`.

```js
const displayImage = document.querySelector("#displayImage");

const image = new Image();
const load = image.on('load', { errorOn: [`error`], once: true });
image.src = "./possibleImage";

load.subscribe({
  next(e) {
    displayImage.src = e.target.src;
  },
  error(e) {
    displayImage.src = "errorloading.png";
  },
  complete() {
    // this notification will be received after next ()
    // as a result of the once member being set to true
  }
})
```

Note that the `errorOn` Array in the `OnOptions` object contains the `"error"` event type. Therefore if the Image receives an `"error"` Event, the Event is passed to the `error`  method of each of the `Observable`'s `Observer`s. This too results in unsubscription from all of the Image's underlying events.


The `nextOn` member is useful in conjuction with `errorOn` and `completeAfter` for observing a DOM process which fires a variety of different events.

### The  `OnOptions` `completeAfter` member

The `completeAfter` member is an array of event types which when received will cause the `complete` method of the Observable's Observers to be invoked.

```js
function handleResponse(xhr) {
  return xhr.on({ 
      nextOn: ["loadstart", "progress", "load"], 
      errorOn: ["abort"], 
      completeAfter: ["load"] 
    }).
    finally(() => view.hideProgressBar()).
    subscribe({
      next(ev) {
        switch(ev.type) {
          case “loadstart”: 
            return view.showProgressBar(xhr);
          case “progress”:  
            return view.shiftProgressBar(ev);
          case “load”:      
            return view.showData(ev);
        }
      },
      error(err) {
        if (err.type !== “abort”) {
          view.showError(err);
        }
      }
    });
}
```

## Example Implementation

This is an example implementation of ObservableEventTarget. The `on` method delegates to
`addEventListener`, and adds a handler for an `"error"` event if the `receiveError` member on the `OnOptions` object has a value of `true`.

```js
class ObservableEventTarget {
  on(type, opts) {
    return new Observable(observer => {
      const normalizedOpts =
        typeof type === "string" ?
          Object.assign({}, opts, {nextOn: [type]}) :
          Object.assign({}, type);

      const handler = (typeof normalizedOpts.handler === "function")
        ? normalizedOpts.handler : null;

      const once = normalizedOpts.once;

      const nextOn =
        Array.isArray(normalizedOpts.nextOn) ? normalizedOpts.nextOn : [];
      const errorOn =
        Array.isArray(normalizedOpts.errorOn) ? normalizedOpts.errorOn : [];
      const completeAfter =
        Array.isArray(normalizedOpts.completeAfter) ? normalizedOpts.completeAfter : [];

      const nextHandler = e => {
        try {
          if (handler != null) {
            handler(e);
          }

          observer.next(e);
        }
        finally {
          if (once) {
            observer.complete();
          }
        }
      };

      for (let type of nextOn) {
        this.addEventListener(type, nextHandler)
      }

      const errorHandler = observer.error.bind(observer);
      for (let type of errorOn) {
        this.addEventListener(type, errorHandler)
      }

      const completeHandler = observer.complete.bind(observer);
      for (let type of completeAfter) {
        this.addEventListener(type, completeHandler)
      }

      // unsubscription logic executed when either the complete or
      // error method is invoked on Observer, or the consumer
      // unsubscribes.
      return () => {
        for (let type of nextOn) {
          this.removeEventListener(type, nextHandler)
        }

        for (let type of errorOn) {
          this.removeEventListener(type, errorHandler)
        }

        for (let type of completeAfter) {
          this.removeEventListener(type, completeHandler)
        }
      };
    });
  }
}
```



## More Compositional Web Applications with ObservableEventTarget

The web platform has much to gain by including a primitive which can allow EventTargets and Promises to be composed. This proposal, along with the Observable proposal currently being considered by the TC-39, are incremental steps towards a more compositional approach to concurrency coordination. If the Observable proposal is accepted, the Observable prototype will have the opportunity to be enriched with useful combinators over time, eliminating the need for a combinator library in common cases.
