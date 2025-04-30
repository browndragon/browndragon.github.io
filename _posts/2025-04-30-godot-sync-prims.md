---
title: Godot Synchronization Primitives
date: 2025-04-30 11:20:00
tags:
  - godot
  - technical
  - async
---

# The Problem
So there you are, happily developing in Godot and GDScript.
It has `signal`s which work the way they do in many other languages -- they're effectively a list of callbacks which can be dispatched at a future date.
But!
You want to do a thing or, on timeout, a different thing.
This is two signals.
As soon as you need to coordinate more than one signal, you enter [Callback Hell](http://callbackhell.com/).

Why?

Well, a few reasons. One, Godot doesn't have Promises. Two, Godot doesn't have `try/catch` in any form.
Without something like promises, a signal can only fire in the future.
That means that code which wants to do something and then continue must be Ever! Vigilant! 
that the thing it's kicking off doesn't complete before the caller gets the chance to subscribe to the signal.
Since it also lacks `try/catch`, exceptional handling uses standard return.
And so you can't easily reuse exception handling or route all failures into the "unhappy" path; you must trudge through every decision and early exit at each and every one.

Worse, there are some tricks (or less kindly: bugs) around object lifetime and parameter currying that make everything harder.
Objects are refcounted.
But signals don't count towards the refcount.
But _custom_ callbacks (not just a method or method with bindings, but a standalone `func` lambda) do count towards the refcount.

There's an early free implementation bug; objects which are used only in capture in a single statement are freed too soon (and so must appear in an assignment).

# Is this code tested?
No, sadly. I wrote the blog post. This is similar to the code I use, but I haven't wrapped it into a lib.
I'm cheating; I should be working on NO DED, but I got sniped.

# TL;DR:
Implement most of, but not all of, promises.
What you actually need to know here are:

1. "Exactly once" happens-after events
2. Correct memory lifetimes for these objects.

We're losing everything about a "rejected" promise, because Godot lacks the architecture to do anything with it.
But with those parts, we can build Race, which _finally_ lets us do happens-after either of two things.

# A gist of the solution

Basically, implement the parts of promises that you actually want.
```
# Listens to a signal and captures the _next_ time it is called.
# This lets you write the standard and difficult pattern:
#
# foo.on_done.connect(_result_of_my_callback)
# foo.do_something_that_might_complete_immediately(params)
#   # Go look at _result_of_my_callback to continue execution w/ results.
#
# in an await-friendly way:
#
# var after := After.of_signal(foo.on_done)
# foo.do_something_that_might_complete_immediately(params)
# var result := await after.async_settled()
class After extends RefCounted:
  var is_settled := false
  var capture:= 0
  var captured:Array

  signal settled

  func settle(_v0: Variant=null, _v1: Variant=null, _v2: Variant=null, _v3: Variant=null) -> void:
    if is_settled: return  # Double trigger! Alas.
    is_settled = true
    captured = [_v0, _v1, _v2, _v3]
    captured.set_size(capture)
    settled.emit()
  func async_settled() -> Variant:
    while not is_settled:
      await settled
    match capture:
      0: return
      1: return captured[0]
      _: return captured

static func of_signal(signal_: Signal, p_capture := 0) -> After:
    var after := new()
    after.capture = p_capture
    signal_.connect(settle, CONNECT_ONE_SHOT)

# Builds on After. Captures the first After to occur.
# Particularly useful for timers!
# 
# var race := Race.new().
#  add(After.of_signal(foo.on_done)).
#  add(After.of_signal(get_tree().create_timer(kMyTimeout))
# foo.do_something_that_might_complete_immediately(params)
# match await race.async_settled():
#   0: do_something_with_result(race.conds[0].captured)
#   1: do_something_on_timeout()
#
class Race extends RefCounted:
  var conds: Array[After]
  var any:= After.new()
  var settled_from:= -1
  func add(after: After) -> Race:
    conds.append(after)
    if (after.is_settled): trigger(conds.size() - 1)
    else: conds.settled.connect(trigger.bind(conds.size() - 1), CONNECT_ONE_SHOT)
    return self
  func trigger(idx: int):
    if any.is_settled: return  # Double trigger! Alas.
    settled_from = idx
    any.settle()
  func async_settled() -> int:
    while not is_settled:
      await settled
    return settled_from
```
