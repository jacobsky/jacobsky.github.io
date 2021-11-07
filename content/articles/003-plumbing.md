+++
title = "ECS Plumbing"
date = 2021-11-08
[extra]
author = "Jacobsky"
tags = ["gamedev", "ecs", "rust", "specs", "godot"]
+++

Something that you may find yourself in need of when working with Godot and an ECS is the ability to send arbitrary commands/queries to the ECS, this can be complicated due to the borrowing semantics of Rust.

While it is possible to just create methods for each query, this method is cumbersome and leads to extremely large source code files with all of the various commands/queries that are required for interaction with your simulation.


TODO