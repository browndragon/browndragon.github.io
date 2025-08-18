---
title: "Godot Debug Cheatsheet: Null Pointers"
date: 2025-08-18 02:19:00
tags:
  - code
  - godot
  - debug
  - newbie
  - npe
  - nullptr
---

I see a lot of people get hung up on some of Godot's error messages, so I'm writing a little article to summarize how to read and resolve common issues I see on the discord.

(Back to [the index]({% post_url 2025-08-17-godot-debug-idx %}))

# Invalid call. Nonexistent function 'stop' in base 'Nil'

**TL;DR:** Godot's null pointer exception.

**More words please:** Some variable you expected to have set *isn't* set at the time you are trying to run your code. That means that when you try to call `my_thing.stop()`, `my_thing` is set to `null` (which is the default value for objects!).

You can encounter this error in lots of different ways; for instance, I picked an example where the unexepected method  `stop` is undefined, but it could be any method you tried to invoke at all, or a property access, or who knows what really.

> Note: `null` is the only value of type `Nil`, which is why the error message refers to `base 'Nil'`.

## How do I fix a null pointer error?

It depends why you're encountering it!

But overall, the strategies rely on ensuring you only call methods on variables which you are confident have already been assigned.

### Your call is happening too early. 

Look at the stack trace along with the break. The stuff at the top of the stack trace are recent calls and the stuff at the bottom are the oldest calls. If the oldest calls look like it's doing object initialization for someone else, check in the debugger whether *this* node is already `is_node_ready()`. It probably isn't! That means that no `@onready var` will have run, your own `func _ready` won't have run...

You usually can't make your references defined earlier (they themselves have to initialize, and that happens after your call to `_enter_tree()`...). So all you can do is move this call later.
Try:
1. Move this whole call to the other object's `_ready` if it's a common parent
2. Use `.call_deferred()` to make the whole call run at the end of the frame
3. Wait for some kind of signal, like for both nodes' `is_node_ready()/ready.emit()` to have run
4. Change the order of the children (!!!, yes, these init methods run in DFS tree order)

### Your node path isn't correct

This can happen for a few reasons overall.

If you're using a `%UniqueName`, then:  
**The script and the node must be in the same scene.** It's not enough to have the UniqueNode be in a scene "Child", and to link "Child" in "Parent", and put the script in "Parent". The script and the unique node must both be in the exact same scene.

If you're using a `$some_path`, then:
**You must have a child node from the script with the exact name `some_path`**. You can't use variable references here (~~`var some_path = "SomePath"~~ is not good enough!), you have to get spelling and underscores etc correct, and the path must calculate from the node with the script attached. (This is also true for `$"some/path"` and `$"../Some/OtherPath"`; they're all literal syntaxes)

Neither of the above syntaxes keep track of node renames, so that could be how the mistake snuck in.

If you're using an `@export var some_node: Node`, then:
**You must set that value everywhere you use it!**. Different instances of the node are different instances of the script each have their own `some_node`.

## It's still broken :(

If all else fails, it's worth auditing your codebase for uses of the script. The break is likely coming from somewhere you didn't expect; after all, if you expected it, you wouldn't be asking for help!

You could try adding a `print(get_path())` before the failing line and running again. That will print the actual path to the script instance; then you can check that it has the expected values and children you want it to have. Maybe it's extra, and the script went wandering or something...?

Happy debugging. Stay tuned, I'll write up more of these as I think of new topics we

> Note: Tony Hoare, inventor of the null reference, has referred to this class of errors as the [Billion Dollar Mistake](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/). Honestly? Underestimate.