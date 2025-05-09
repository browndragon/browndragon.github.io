---
title: Swipe this controller (please)
date: 2025-05-09 05:42:00
tags:
  - code
  - noded
  - godot
  - ux
---
If you want touch controls to map to standard input actions, steal the code from below.

# Backstory
I was paying my tutorial tithe (give help 10x the number of times you ask for help) and some brilliant people I respect pointed out that Godot 4 is still gently broken with respect to swipes.

In general UI is hard, so this isn't criticism.

I thought of a "yes and", and then decided it'd be too weird to necro the thread to post it, so I'm posting it here for posterity.

## NODED tie in!
I want my horror game on tablet ideally. And I have at least two modes of input (longpress the omen to indicate it's wrong, but also some mechanism of scrolling around the world).
The former is a tap and the latter is a swipe. I might get _really_ fancy and do multifinger gestures.
But anyway, being able to map complex user gestures back into simple command inputs would be very nice.

And in fact I already do this!
The user's **keyboard** input is mapped into **minimap** movement.
The minimap movement triggers various areas on the minimap and they inject **camera** movement events
(which isn't directly the same as the user's input -- the camera is moving 6dof while the user only controls WASD).

So! Here's the code.

# Godot's input 

```python
# Turns screen events into real actions.
# Put me in the scene that wants it, or into an autoload if you like to live dangerously.
# I use `unhandled_input` so that e.g. drag handlers can grab my event away from me.
extends Node

# In standard `Input.get_vector` order.
@export var actions: Array[StringName] = [&"ui_left", &"ui_right", &"ui_up", &"ui_down"]
const directions: Array[Vector2] = [Vector2.LEFT, Vector2.RIGHT, Vector2.UP, Vector2.DOWN]

# If you cover 1/64th of the screen in one gesture, you're outputting max strength controls. Less? Less.
# Probably there should be some sensitivity here.
# You could also imagine an onscreen joystick, using displacement from a fixed point rather than relative swipes.
# You could imagine buffering the input so you can smooth the touch (acceleration, etc).
# But this is what the specific user wanted.
# I have not tuned this param at all.
@export var max_speed_as_frac_of_screen := 1.0 / 64 * Vector2.ONE

func _unhandled_input(e: InputEvent):
  if e is InputEventScreenDrag:
    # Could cache this value and only recalc on window resize.
    # Assumes we are under root, not a subviewport.
    var pixels_for_one := get_viewport().size * max_speed_as_frac_of_screen as Vector2
    var scaled := e.screen_velocity / pixels_for_one
    for i in directions.size():
      var event := InputEventAction.new()
      event.action = actions[i]
      event.strength = clampf(directions[i] * scaled, 0, 1)
      event.pressed = !!event.strength  # 0.0 is false, which the clampf ensures is slightly likely.
      Input.parse_input_event(event)
```
