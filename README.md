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

### Unhandled Promise Rejection Tracking

(This section probably belongs somewhere within the [Scripting](https://html.spec.whatwg.org/multipage/webappapis.html#scripting) section.)

In addition to synchronous [runtime script errors](https://html.spec.whatwg.org/multipage/webappapis.html#runtime-script-errors), scripts may experience asynchronous promise rejections, tracked via the `unhandledrejection` and `rejectionhandled` events.

#### The HostPromiseRejectionTracker implementation

ECMAScript contains an implementation-defined HostPromiseRejectionTracker(_promise_, _operation_) abstract operation. User agents must use the following implementation.

This implementation results in promise rejections being marked as **handled** or **unhandled**. These concepts parallel the same ones for [script errors](https://html.spec.whatwg.org/multipage/webappapis.html#concept-error-handled).

1. If _operation_ is `"reject"`,
    1. Add _promise_ to the about-to-be-notified rejected promises list.
    1. Queue a task to perform the following steps:
        1. For each entry _p_ in the about-to-be-notified rejected promises list,
            1. Let _event_ be a new trusted `PromiseRejectionEvent` object that does not bubble and is cancelable, and which has the event name `unhandledrejection`.
            1. Initialise _event_'s `promise` attribute to _p_.
            1. Initialise _event_'s `reason` attribute to the value of _p_'s [[PromiseResult]] internal slot.
            1. Dispatch _event_ at the current script's [global object](https://html.spec.whatwg.org/multipage/webappapis.html#global-object).
            1. If event was canceled, then the promise rejection is handled. Otherwise, the promise rejection is not handled.
            1. If _p_'s [[PromiseIsHandled]] internal slot is false, add _p_ to the outstanding rejected promises weak set.
        1. Clear the about-to-be-notified rejected promises list.
1. If _operation_ is `"handle"`,
    1. If the about-to-be-notified rejected promises list contains _promise_, remove _promise_ from the about-to-be-notified rejected promises list and return.
    1. If the outstanding rejected promises weak set does not contain _promise_ then return.
    1. Remove _promise_ from the outstanding rejected promises weak set.
    1. Let _event_ be a new trusted `PromiseRejectionEvent` object that does not bubble and is not cancelable, and which has the event name `rejectionhandled`.
    1. Initialise _event_'s `promise` attribute to _promise_.
    1. Initialise _event_'s `reason` attribute to the value of _promise_'s [[PromiseResult]] internal slot.
    1. Dispatch _event_ at the current script's [global object](https://html.spec.whatwg.org/multipage/webappapis.html#global-object).

#### The PromiseRejectionEvent interface

```webidl
[Constructor(DOMString type, optional PromiseRejectionEventInit eventInitDict), Exposed=(Window,Worker,ServiceWorker)]
interface PromiseRejectionEvent : Event {
  readonly attribute Promise<any> promise;
  readonly attribute any reason;
};

dictionary PromiseRejectionEventInit : EventInit {
  Promise<any> promise;
  any reason;
};
```

The `promise` attribute must return the value it was initialised to. When the object is created, this attribute must be initialised to null. It represents the promise which this notification is about.

The `reason` attribute must return the value it was initialized to. When the object is created, this attribute must be initialised to null. It represents the rejection reason for the promise.

## [WindowEventHandlers](https://html.spec.whatwg.org/multipage/webappapis.html#windoweventhandlers)

Add

```webidl
  attribute EventHandler onunhandledrejection;
  attribute EventHandler onrejectionhandled;
```

## [WorkerGlobalScope](https://html.spec.whatwg.org/multipage/workers.html#workerglobalscope)

Add

```webidl
  attribute EventHandler onunhandledrejection;
  attribute EventHandler onrejectionhandled;
```

## Notes

- Implementations should use the handled/unhandled state of promise rejections (as defined in HTML) when determining what to log on the console. That is, intercepting an `unhandledrejection` event and calling `preventDefault()` should prevent the corresponding rejection from showing up in the developer console.

## WARNING!!

The above HTML-side algorithm is probably not be exactly right, especially in regard to timing. When implementing something similar in io.js ([nodejs/io.js#758](https://github.com/nodejs/io.js/pull/758)), we found that our initial implementation could cause strange issues, equivalent to the following:

```js
window.on("unhandledrejection", () => console.log("unhandledrejection"));

const a = Promise.reject(); // (1)

queueTask(() => {
  const b = Promise.reject();
  queueMicrotask(() => b.catch(() => {}));
});

// This logged "unhandledrejection" twice.
// If you commented out (1), it logged nothing at all (!?).
// The desired behavior is to log once! (i.e. log for a, and not for b).
```

There is a [suite of test cases](https://github.com/nodejs/io.js/blob/master/test/parallel/test-promises-unhandled-rejections.js) covering the desired behavior, and we hope to experiment while implementing to find appropriate tweaks to the spec to achieve the desired result.

You can discuss this further in [issue #2](https://github.com/domenic/unhandled-rejections-browser-spec/issues/2).
