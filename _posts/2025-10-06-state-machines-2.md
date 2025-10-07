---
title: "State Machines 2: l'État c'est moi"
date: 2025-10-06 16:00:00
mermaid: true
tags:
  - code
  - godot
  - ai
  - npc
---

The [Previous post]({% post_url 2025-10-06-state-machines %}) covers the theory and practice of programmer's state machines.
At the end I linked to code for a `State` which can be started/stopped/paused/unpaused based on its relationship to parent/parent/child/child state(s).

Now I want to use it to solve the actual problem I've got with animating my **M**obile **OB**jects around a scene in fun ways.

# The Problem

In my current game, I want the player to be able to disarm weapons from enemies.
This means it's a real possibility that an enemy will find themselves _without_ a weapon and within striking range of the player.
When that happens, I want them to get some distance to safety, find a weapon, and jump back into the fray.

So: their behavior has to have some branches that can prioritize somewhat complex behavior.

Let's start with walking, and build their overall behavior from the bottom up.

# Actions: It's the concrete steps of what we do

Let's assume we've put every "ability" a creature can have into a single folder under their node, something like:
```
\ Actions(Node) # Just a folder!
  \ Run
  \ Crouch
  \ WalkFunny  # Like a crab!
  \ Backstab
  \ ChargeAttack
  \ CastSpellFireball
  \ CastSpellInvisibility
  \ CastSpellTeleport
  \ FlinchInvoluntary
  \ DieInvoluntary
  \ ...
```

We can interrogate nodes by treating them as data!

```
@abstract
class_name Action extends State
@export var runner: State

# When we set this, we add ourselves as a child to the runner
# for the duration of the action, until we call `end`.
@abstract func execute(...params) -> void

# Return type: `Array[ParamDescriptor]`
# ParamDescriptor is `{"name": StringName, "optional"?: bool, "type": type}`
# where type = `VariantType|[TYPE_ARRAY, type]|[TYPE_DICTIONARY, type, type]`
func get_do_action_param_metadata(): Array
```

# Drives: It's the generic idea of what a MOB *could* do

One simple drive we could give our MOBs is the drive to WANDER.

```mermaid
flowchart LR
subgraph WANDER
WANDER_FIND[Find]
WANDER_GOTO[Goto]
WANDER_END[End]
end
WANDER_FIND --> WANDER_GOTO
WANDER_FIND --> WANDER_END
WANDER_GOTO --> WANDER_END
```

We know when we're done a-wandering: when we get where we were going (or decide we can't get there, etc).

There's a lot of actions we could choose to implement `WANDER_GOTO`: `Run`, `Crouch`, `WalkFunny`, or `CastSpellTeleport`.
`WANDER_FIND` is a little more restricted in this case, though in other drives it could be a lot more specific.

The idea is that WANDER's job is to make those decisions: figure out if we're done (very low utility to do it again!), figure out where we want to wander towards (might figure into utility if the only places we can wander suck), figure out how we want to wander (using which skills), and then apply the state machine until it terminates (or _is_ terminated, such as by some higher utility drive preempting WANDER).

This itself sounds like a state machine.

```
class_name Drive extends SubState
# When this is called, we get the chance to consider a new plan.
# This might do any amount of actual planning (to produce a better utility number).
# Or it might cheat and just provide a general estimate of utility.
# If this overestimates utility
# ("yes, run to safety", 'ok, running away wins! what's the plan?' "oh.. nowhere is safe ;_;")
# then we'll make pretty bad decisions.
@abstract func recalculate_utility() -> float

class_name Drives extends Drive
# These are the drives we'll hold elections between; the winner is set as this Drives' substate.
# Might just be `get_children()` in practice!
#
# It's critical that we call `recalculate_utility` at relevant intervals, because that's what allows
# preemption of previous plans.
var subdrives: Array[Drive]
```

## Interactions between lots of drives make things interesting
```mermaid
flowchart LR

WANDER

subgraph HUNT
subgraph ATTACK
ATTACK_FIND[Find]
ATTACK_GOTO[Goto]
ATTACK_ATTACK[Attack]
ATTACK_END[End]
end
HUNT_FEED[Feed]
HUNT_END[End]
end
ATTACK_FIND --> ATTACK_GOTO
ATTACK_GOTO --> ATTACK_ATTACK
ATTACK_ATTACK --> ATTACK_END
ATTACK --> HUNT_FEED
HUNT_FEED --> HUNT_END

subgraph FLEE
FLEE_FIND[Find]
FLEE_GOTO[Goto]
FLEE_END[End]
end
FLEE_FIND --> FLEE_GOTO
FLEE_GOTO --> FLEE_END

subgraph HIDE
HIDE_FIND[Find]
HIDE_GOTO[Goto]
HIDE_HUNKER[Hunker]
HIDE_WAIT[Wait]
HIDE_END[End]
end
HIDE_FIND --> HIDE_GOTO
HIDE_GOTO --> HIDE_HUNKER
HIDE_HUNKER --> HIDE_WAIT
HIDE_WAIT --> HIDE_END

HIDE_WAIT --[inrange]--> ATTACK_ATTACK
```

Complex stuff! states, nested subgraphs, drives that include sub-drives to ensure delightful behaviors.

# Social Objects: Keeping the campfire stoked

I have a second concern. I'm not going to build it up front necessarily, but it has a place in the game.

Social objects.

Imagine a scene with a campfire. If there's nothing else going on, I think it would be nice if NPCs
that were capable of certain actions (a suite of actions like "sit", "move", "pickup", and "drop") could sit around the campfire!

The campfire has a freefloating drive which it can temporarily grant others:
```mermaid
flowchart LR
subgraph ENJOY
ENJOY_GOTO
ENJOY_SIT
ENJOY_WAIT
ENJOY_END
end
subgraph REFUEL
REFUEL_CHECK_LEVEL
REFUEL_FIND
REFUEL_GOTO_FUEL
REFUEL_PICKUP
REFUEL_GOTO_SITE
REFUEL_STOKE
REFUEL_END
end
REFUEL_CHECK_LEVEL --> REFUEL_FIND
REFUEL_FIND --> REFUEL_GOTO_FUEL
REFUEL_GOTO_FUEL --> REFUEL_PICKUP
REFUEL_PICKUP --> REFUEL_GOTO_SITE
REFUEL_GOTO_SITE --> REFUEL_STOKE
REFUEL_STOKE --> REFUEL_END
```
with the idea that we give any PC or NPC that _can_ take Social Object drives into account an additional Social ObjectDrive as a child, whose set of electable subdrives is populated as clones from those social objects to which they are attached, or those which they were recently executing, etc.

Neat, huh?