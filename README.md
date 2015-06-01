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

1. If the value of _promise_'s [[PromiseIsHandled]] internal slot is **false**, perform HostPromiseRejectionTracker(_promise_, **false**).

### [PerformPromiseThen( promise, onFulfilled, onRejected, resultCapability )](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-performpromisethen)

Add an additional step nested inside step 9, between 9a and 9b:

1. If the value of _promise_'s [[PromiseIsHandled]] internal slot is **false**, perform HostPromiseRejectionTracker(_promise_, **true**).

Add an additional step between steps 9 and 10:

1. Set _promise_'s [[PromiseIsHandled]] internal slot to **true**.

### [Promise Abstract Operations](https://people.mozilla.org/~jorendorff/es6-draft.html#sec-promise-abstract-operations)

Add an additional section as follows:

#### HostPromiseRejectionTracker( promise, handled )

HostPromiseRejectionTracker is an implementation-defined abstract operation that allows host environments to track promise rejections better. Specifically, it is called in two scenarios:

- When a promise is rejected without any handlers, it is called with its _handled_ argument set to **false**.
- When a handler is added to a rejected promise for the first time, it is called with its _handled_ argument set to **true**.

The implementation of HostPromiseRejectionTracker must conform to the following requirements:

- It must complete normally in all cases.
- It must not hold a reference to _promise_ in a way that would interfere with garbage collection.

A typical implementation of HostPromiseRejectionTracker would try to notify developers of unhandled rejections, while also being careful to notify them if such previous notifications are later invalidated by new handlers being attached.

## Changes to HTML

TODO. Draw on https://gist.github.com/domenic/9b40029f59f29b822f3b#host-environment-algorithm.
