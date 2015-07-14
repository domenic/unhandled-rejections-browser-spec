# Unhandled Rejection Tracking Browser Events

This repository is a place to work on some spec patches to [HTML](http://html.spec.whatwg.org/multipage/) and [ECMAScript](http://people.mozilla.org/~jorendorff/es6-draft.html) to add support for events to track unhandled promise rejections, as originally proposed [on the WHATWG mailing list](http://lists.w3.org/Archives/Public/public-whatwg-archive/2014Sep/0024.html).

## Changes to ECMAScript

### [Properties of Promise Instances](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-properties-of-promise-instances)

Add the following row to the table of internal slots:

Internal Slot        | Description
---------------------|-------------
[[PromiseIsHandled]] | A boolean indicating whether the promise has ever had a fulfillment or rejection handler, used in unhandled rejection tracking

### [Promise ( executor )](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-promise-executor)

Add an additional step between steps 6 and 7:

1. Set _promise_'s [[PromiseIsHandled]] internal slot to **false**.

### [RejectPromise( promise, reason )](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-rejectpromise)

Add an additional step between steps 6 and 7:

1. If the value of _promise_'s [[PromiseIsHandled]] internal slot is **false**, perform HostPromiseRejectionTracker(_promise_, `"reject"`).

### [PerformPromiseThen( promise, onFulfilled, onRejected, resultCapability )](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-performpromisethen)

Add an additional step nested inside step 9, between 9a and 9b:

1. If the value of _promise_'s [[PromiseIsHandled]] internal slot is **false**, perform HostPromiseRejectionTracker(_promise_, `"handle"`).

Add an additional step between steps 9 and 10:

1. Set _promise_'s [[PromiseIsHandled]] internal slot to **true**.

### [Promise Abstract Operations](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-promise-abstract-operations)

Add an additional section as follows:

#### HostPromiseRejectionTracker( promise, operation )

HostPromiseRejectionTracker is an implementation-defined abstract operation that allows host environments to track promise rejections better. Specifically, it is called in two scenarios:

- When a promise is rejected without any handlers, it is called with its _operation_ argument set to `"reject"`.
- When a handler is added to a rejected promise for the first time, it is called with its _operation_ argument set to `"handle"`.

The implementation of HostPromiseRejectionTracker must conform to the following requirements:

- It must complete normally in all cases.
- If _operation_ is `"handle"`, it must not hold a reference to _promise_ in a way that would interfere with garbage collection.

A typical implementation of HostPromiseRejectionTracker would try to notify developers of unhandled rejections, while also being careful to notify them if such previous notifications are later invalidated by new handlers being attached.

_NOTE_ an implementation may hold a reference to _promise_ if _operation_ is `"reject"`, since it is expected that rejections will be rare and not on hot code paths.

## Changes to HTML

### Stuff to put somewhere

Environment settings object seems to be a place to dump stuff? Need to define these.

- Outstanding rejected promises weak set
- About-to-be-notified rejected promises list

### [Perform a microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint)

Insert a step between steps 8 and 9:

1. _Done:_ If the about-to-be-notified rejected promises list is not empty,
  1. Queue a task to <a href="#user-content-notify-about-rejected-promises">notify about the rejected promises</a> currently in the about-to-be-notified rejected promises list.
  1. Clear the about-to-be-notified rejected promises list.

Modify step 9 to remove the "_Done:_" label from it.

### Unhandled promise rejections

(This section probably belongs somewhere within the [Scripting](https://html.spec.whatwg.org/multipage/webappapis.html#scripting) section, probably right after [Runtime script errors](https://html.spec.whatwg.org/multipage/webappapis.html#runtime-script-errors).)

In addition to synchronous [runtime script errors](https://html.spec.whatwg.org/multipage/webappapis.html#runtime-script-errors), scripts may experience asynchronous promise rejections, tracked via the `unhandledrejection` and `rejectionhandled` events.

#### The HostPromiseRejectionTracker implementation

ECMAScript contains an implementation-defined HostPromiseRejectionTracker(_promise_, _operation_) abstract operation. User agents must use the following implementation.

This implementation results in promise rejections being marked as **handled** or **unhandled**. These concepts parallel handled and not handled for [script errors](https://html.spec.whatwg.org/multipage/webappapis.html#concept-error-handled).

1. If _operation_ is `"reject"`,
    1. Add _promise_ to the about-to-be-notified rejected promises list.
1. If _operation_ is `"handle"`,
    1. If the about-to-be-notified rejected promises list contains _promise_, remove _promise_ from the about-to-be-notified rejected promises list and return.
    1. If the outstanding rejected promises weak set does not contain _promise_ then return.
    1. Remove _promise_ from the outstanding rejected promises weak set.
    1. Let _event_ be a new trusted `PromiseRejectionEvent` object that does not bubble and is not cancelable, and which has the event name `rejectionhandled`.
    1. Initialise _event_'s `promise` attribute to _promise_.
    1. Initialise _event_'s `reason` attribute to the value of _promise_'s [[PromiseResult]] internal slot.
    1. Dispatch _event_ at the current script's [global object](https://html.spec.whatwg.org/multipage/webappapis.html#global-object).

#### Notification of rejected promises

To <a id="notify-about-rejected-promises">**notify about a list of rejected promises**</a>, given a list _list_, perform the following steps:

1. For each entry _p_ in _list_,
    1. Let _event_ be a new trusted `PromiseRejectionEvent` object that does not bubble and is cancelable, and which has the event name `unhandledrejection`.
    1. Initialise _event_'s `promise` attribute to _p_.
    1. Initialise _event_'s `reason` attribute to the value of _p_'s [[PromiseResult]] internal slot.
    1. Dispatch _event_ at the current script's [global object](https://html.spec.whatwg.org/multipage/webappapis.html#global-object).
    1. If event was canceled, then the promise rejection is handled. Otherwise, the promise rejection is unhandled.
    1. If _p_'s [[PromiseIsHandled]] internal slot is false, add _p_ to the outstanding rejected promises weak set.

#### The PromiseRejectionEvent interface

```webidl
[Constructor(DOMString type, optional PromiseRejectionEventInit eventInitDict), Exposed=(Window,Worker,ServiceWorker)]
interface PromiseRejectionEvent : Event {
  readonly attribute Promise<any>? promise;
  readonly attribute any reason;
};

dictionary PromiseRejectionEventInit : EventInit {
  required Promise<any>? promise;
  any reason = null;
};
```

The `promise` attribute must return the value it was initialised to. It represents the promise which this notification is about.

The `reason` attribute must return the value it was initialized to. It represents the rejection reason for the promise.

### [WindowEventHandlers](https://html.spec.whatwg.org/multipage/webappapis.html#windoweventhandlers)

Add

```webidl
  attribute EventHandler onunhandledrejection;
  attribute EventHandler onrejectionhandled;
```

### [WorkerGlobalScope](https://html.spec.whatwg.org/multipage/workers.html#workerglobalscope)

Add

```webidl
  attribute EventHandler onunhandledrejection;
  attribute EventHandler onrejectionhandled;
```

### Notes

- Implementations should use the handled/unhandled state of promise rejections (as defined in HTML) when determining what to log on the console. That is, intercepting an `unhandledrejection` event and calling `preventDefault()` should prevent the corresponding rejection from showing up in the developer console.

- Implementations are free to limit the size of the rejected promises weak set.
