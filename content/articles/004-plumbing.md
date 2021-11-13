+++
title = "ECS Plumbing"
date = 2021-11-08
[extra]
author = "Jacobsky"
tags = ["gamedev", "ecs", "rust", "specs", "godot"]
+++

Something that you may find yourself in need of when working with Godot and an ECS is the ability to send arbitrary commands/queries to the ECS, this can be complicated due to the borrowing semantics of Rust.

While it is possible to just create methods for each query, this method is cumbersome and leads to extremely large source code files with all of the various commands/queries that are required for interaction with your simulation.

```rs
struct Simulation {
    world: World
}

impl Simulation {
    fn query_entity_position(&self, entity: Entity) -> Vector2 {
        // Do Query
    }
    fn query_something_else(&self, entity: Entity) -> SomethingElse {
        // Do Query
    }
}
```

As most of these queries may be relatively arbitrary when related to the UI, it would be much more useful to be able to define some kind of interface for dispatching arbitrary queries. Some ECS frameworks may already have an approach defined for it, but in the case of other frameworks -- such as specs -- it is necessary to come up with our own pattern.

While there's a lot of room to allow for dynamic dispatch, one option that I stumbled upon is using a Trait to let monomorphization build all of the underlying functions for us.

It starts with a single trait.

```rs
trait WorldQuery {
    type Args;
    type Output; 
    fn query(world: &World, args: Self::Args) -> Self::Output;
}
```

Then in our `World` container we can implement a generic function over this trait similar to the following:

```rs
impl Simulation {
    pub fn query<WC: WorldQuery>(&self, args: WC::Args) -> Self::Output {
        WC::query(self.world, args)
    }
}
```

The interesting part of these queries are that they contain no state and can be defined in any module. I use this extensively in my GUI code as the GUI holds a shared reference to the `Simulation` class.

In this way you can easily write something like the following GUI Class.

```rs
struct QueryGUIStuff;
impl WorldQuery for QueryGUIStuff {
    type Args = ();
    type Output = ();
    fn query(world: &World, args: Self::Args) -> Self::Output {
        // Do the query
    }
}

pub struct MyGUI {
    // We are assuming that the simulation uses internal mutability and handles the mutex on it's own.
    simulation: Arc<Simulation>,
}

impl MyGUI {
    fn update(&mut self) {
        let gui_stuff = self.simulation.query::<QueryGUIStuff>();
        // draw the `gui_stuff`
    }
}
```

Using the above pattern is not without risks as I use this with `godot-rust` I know that all GUI code will only ever query in a singlethreaded fashion, if you wish to have the GUI and engine running on separate threads, it will be necessary to find some other option for signalling updates and ensuring thread safe communication via `Arc<Mutext<T>>`

This is not without risks. This pattern could also be implemented for a `&mut World` which would allow for arbitrary code to be made such that it can change the world. This may be useful in certain cases, but should be used with caution.

Anyways, I hope this pattern proves as useful for you as it has for me.
