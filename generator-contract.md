# Generator Contract
This document defines the contract between Highland and generator
implementators. Writers of Highland generators (via the generator constructor of
`consume`) must follow this contract or risk undefined and hard to debug
behavior. This contract aims to ease implementation of generators as much as
possible while still maintaining proper guarantees that Highland can rely on.

## Introduction
A *generator* is any handler in Highland that is provided with a `push` and
`next` functions. These functions have the following signature:

```haskell
push(err, x) :: (E, Either[X, NIL]) -> Unit
next(xs) :: Optional[Stream] -> Unit
```

## The contract for push
`push` allows the user to push an error or a value into the stream. The stream
will buffer that value until the stream's consumer have signaled that they are
ready, at which point it will be emitted to the consumer. Pushing the special
`nil` value ends the stream. The contract for `push` is as follows:

1. The `push` function provided to the generator for a particular stream will
   always be the same. Users MAY cache the first `push` that they get and reuse
   it instead of updating the value with every call to the generator.
2. To push a *value* (including the special `nil` value), call `push(null,
   value)`. `push(undefined, value)` is also OK.
3. To push an error, call `push(err)` or `push(err, null)`;
4. Users MUST NOT call `push` with both a non-null error and a non-null value.
5. Users MUST NOT call `push` after pushing the `nil` value.
6. Other than requirement (4), users MAY call `push` at any time, synchronously
   or asynchronously.
7. Users MUST NOT make any assumptions about whether or not a `push`ed value has
   reached the stream's consumer.
8. A call to `push` will never cause a synchronous re-entry into the generator
   unless it also provokes a call to `next`. Highland code will never do this,
   so it can only happen in a downstream consumer. In general, consumers should
   not be provided with a reference to the `push` or `next` functions.

## The contract for next
`next` is a way for implementors to signal that they are done with the current
execution of the generator and Highland is free to invoke the generator again
*at any time*. Judicious calls to `next` can be used to implement true
backpressure, where no values are produced, instead of the default buffering
that Highland does by default. `next` may also be called with another Highland
stream. This will delegate value generator to the specified stream.

1. The `next` function provided to the generator for a particular stream will
   always be the same. Users MAY cache the first `next` that they get and reuse
   it instead of updating the value with every call to the generator.
2. Users MUST NOT invoke `next` after calling it with a delegate stream.
3. Users MAY call `next` multiple times *with no arguments* in between
   invocations of the generator.  Highland will treat such multiple calls as the
   same as a single call.
4. Users MUST NOT invoke the `next(delegate)` version of `next` after invoking
   the no argument version without a call to the generator in between.
5. Highland MUST NOT invoke the generator until after the user has called `next`
   at least once. The exception is the first time the generator is invoked,
   which will be done whenever Highland is ready.
6. Highland MUST NOT invoke the generator after the user calls the
   `next(delegate)` version of `next`.

## The contract for interleaving of calls to next and push
1. Users MUST NOT invoke `next` or `push` after they push the `nil` value.
3. Users MUST NOT invoke `next` or `push` after they call `next(delegate)`.
2. Users MAY invoke `push` even after calling `next()` and before the generator
   is invoked again. That is, this series of event is OK: `generator() ->
   push(err) -> next() -> push(null, value) -> push(err) -> generator()`
