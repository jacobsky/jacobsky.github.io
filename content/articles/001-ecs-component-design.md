+++
title = "How do design your ECS components."
date = 2021-11-05
[extra]
author = "Jacobsky"
tags = ["gamedev", "ecs", "rust", "specs"]
+++

Architecture is a constant thorn in my side as I continue to develop my game. While migrating to using [SPECS](https://github.com/amethyst/specs) helped alleviate many issues and make my architecture more flexible and parallelilzable, the point that I struggled with the most was "What constitutes a good component design?".

As [kyren put so eloquently](https://kyren.github.io/2018/09/14/rustconf-talk.html) "ECS is SQL for games" and I think this is a good place to start when considering how we should lay out our data. 

Note: This article is in reference to Rust and [SPECS Parallel ECS](https://github.com/amethyst/specs)

If we think about an ECS `World` as a `database instance`, we can see the following:

- `Entity` is a `PrimaryKey`
- `Component` are `Tables`
- `System` is both the App Logic and the Query logic that uses the Database. This often makes use of copious table joins to work with various data.

## Structuring Component Data

In most Rust based ECS all of our components are stored as structs and it is possible to put any data that you want into it.

For example, when working with rendering/spatial data we could write the `Transform3D` structs as either:
```rust
#[derive(Component)]
struct Transform3D {
    position: (f32, f32, f32),
    rotation: (f32, f32, f32),
    scale: (f32, f32, f32),
}
// Alternatively 
#[derive(Component)]
struct Transform3D {
    position_x: f32
    position_y: f32
    position_z: f32
    //
    rotation_x: f32
    rotation_y: f32
    rotation_z: f32
    //
    scale_x: f32
    scale_y: f32
    scale_z: f32
}
```

Fundamentally, these are the same data, just layed out in slightly different ways. There may be _some_ differences in the performance characteristics, but that requires profiling.

From an architectural standpoint, this does mean that it is not possible to have an entity with a `Transform3D` that only represents position.

If you want to make it so that the various portions are optional, you need to take one of two approaches.

Making each part internally optional:
```rs
#[derive(Component)]
struct Transform3D {
    position: Option<(f32, f32, f32)>,
    rotation: Option<(f32, f32, f32)>,
    scale: Option<(f32, f32, f32)>,
}
```

Or representing them as separate components:
```rs
#[derive(Component)]
struct Position3D {
    x: f32,
    y: f32,
    z: f32,
}
#[derive(Component)]
struct Rotation3D {
    x: f32,
    y: f32,
    z: f32,
}
#[derive(Component)]
struct Scale3D {
    x: f32,
    y: f32,
    z: f32,
}
```

Note: this is a trivial example that is intended to clearly demonstrate some different ways to organize the same data. You should evaluate each approach to ensure that it has the necessary performance characteristics for your game.

A more complex example of something that you might need to work with in ECS is an inventory system.

If items in your game are relatively simple and don't require complex data, you could probably make an inventory system work pretty easily with something like the following:

```rs
enum ItemKind {
    A, B, C, D
}

struct InventoryItem {
    kind: ItemKind,
    quantity: i32
}

#[derive(Component)]
struct Inventory {
    items: Vec<InventoryItem>
}
```

There are some additional changes that you could also make to allow for more configurable kinds of items, but the assumption in the above code is that Inventory Items are not entities and cannot exist in the world.

What if we want to allow items to also be entities. This may be useful for games with a much higher degree of simulation, e.g. Clockwork games like Hitman where items can also exist in the world and may have many types of possible interactions.

Another advantage is that rather than always duplicating the item information, it would be possible to organize the items as references to other items.

One way to handle this would be to change the inventory from `InventoryItem` to `Entity` like:

```rs
#[derive(Component)]
struct Inventory {
    items: Vec<Entity>
}
```

Then we can define any data on our item entity that we want. We can also create separate components to indicate stackability, add description information, etc.

But there's a problem with this solution. In case you didn't catch it, this allows for any `Inventory` to store any `Entity` and it is possible for the same `Entity` to be stored in two separate `Inventory`s. One option is to manually check every inventory to ensure that an Entity isn't stored else-where (this is a bad idea as it is super performance intensive).

As this highlights a common issue with ECS, I'm going to spoil it for you a bit. This is the "relational" part of relational database.

## Component Relationships

Something that we can use as a bit of a guide when considering this is [Database Normalization](https://en.wikipedia.org/wiki/Database_normalization), in particular, Third Normal Form which creates the following constraints with regard to table design.

| Constraint | ECS Equivalent |
| --- | --- |
| Primary key | Entity |
| Atomic columns where cells cannot have tables as values | Components cannot have properties that are also Components |
| Every non-primary-key attribute is fully functionally dependent on the primary key | No struct should be dependent upon more than the Entity itself | 
| No non-primary-key attribute is transitively dependent on the primary key | Don't make tables that both depend on another Entity and contain duplicate data from the entity |

Now this is a rough guide on the translation between the two. In 99% of cases, common sense will keep your data conforming with 3NF as structuring your data often wouldn't make any sense otherwise.

The next step is to determine the actual relationships that we need to create.

Considering the inventory example:

An inventory can have 0 or more entities, but an entity can only belong to a single entity. As such we can identify that this is a one to many relationship. While the previous option _technically_ models this. It is more convenient to model it in reverse.


```rust
/// This component will add any entity. It is the system's responsibility for ensuring that the 
#[derive(Component)]
struct StoredInContainer {
    stored_in: Entity
}

// This is any entity that can act as a container. NullStorage allows us to effectively create a `Table` with only a single column.
#[derive(Component)]
#[storage(NullStorage)]
struct InventoryContainer;
```
Using specs we can create the relationship between two entities in the following code:

```rust
let world = World::new();
let inventory = world.create_entity()
    // Add other data of interest
    .with(InventoryContainer{})
    .build();
let item = world.create_entity()
    // Add other data of interest
    .with(Storage)
    .with(StoredInContainer { stored_in: inventory })
    .build();
```

One point that may confuse some newer developers is how you can figure out which entity owns a given item.

The general answer is that you would do a join across the `Entity` table checking whether the `StoredInContainer` entity refers to the entity in question. There may be some additional optimizations that you can look into, but as most Iventory look up would happen relatively infrequently.

In specs this looks like:

```rs
// Assume that you have an entity named entity_with_inventory
let entities = world.entities();
let stored_in_container();

let entities : Vec<Entity> = (&entities, &stored_in_containers)
    .join()
    .filter(|(_entity, stored)|{ entity_with_inventory == stored_in_container.stored_in})
    .map(|entity, _stored| entity)
    .collect();
```
