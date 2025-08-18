---
title: Godot Debug Cheatsheet Index
date: 2025-08-17 02:19:00
tags:
  - code
  - godot
  - debug
  - newbie
---

I see a few issues recur in the Godot support discord; I thought I'd write up some common issues somewhere so they'd be easier to understand.

* Null Pointer Exception (Link TODO)
  * Check `@onready`, `.call_deferred()`, correct paths, script attachment
* Missing Physics Collision (TODO)
  * Check area vs body, areas are monitoring & monitorable, mask vs layer, signals are connected to methods, everything has a shape
* Connections Between Scenes (TODO)
  * Use common parent requirements, groups, physics & similar discovery, SignalBus & other static data
* Responsive UI (TODO)
  * Read the docs; do your layout responsively
* Routing Input events
  * Use MouseFilter, especially the new recursive settings.

Anything else I should add?