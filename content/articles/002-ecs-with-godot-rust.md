+++
title = "ECS with Godot."
date = 2021-11-07
[extra]
author = "Jacobsky"
tags = ["gamedev", "ecs", "rust", "specs", "godot", "godot-rust"]
+++

## Note: This article is still a work in progress.

If you've read the godot-rust book's [Chapter on Game Architecture](https://godot-rust.github.io/book/gdnative-overview/architecture.html), you may have some questions about how this could work with ECS.

The basic answer is going to be based on option #3. But what does this look like?

The answer can be found in the following repo: [godot-specs-integration](https://www.github.com/jacobsky/godot-specs-integration).

There are a number of examples that you can view with information about how these integrations work, but in brief, the high level concept is that there are many ways to integrate an ECS with Godot is that you need the following:

- Node that accesses the control flow of the game and updates the game state
- A node that owns the ECS World(s)

It is possible for both of these to be the same Node.

The next godot-specific part is synchronizing the data between the ECS world and the godot-scene tree/library

## 1. Synchronizing via Godot Nodes

TODO: Finish this

## 2. Synchronizing via Godot IO Layer (Server Singletons)

TODO: Finish this