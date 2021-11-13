+++
title = "ECS Message Passing"
date = 2021-11-08
[extra]
author = "Jacobsky"
tags = ["gamedev", "ecs", "rust", "specs", "godot"]
+++

One common issue that you can encounter when working with entity component systems is that interactions can be too complex to easily encapsulate in a single system. I ran into this issue a lot when I was working on my own tactics RPG.

## Scenario

Characters can use Abilities. The `Ability` struct is a template that allows me to define what spells are. Abilities have a number of components that all require different ways of managing the `Ability`.

For example:
1. Firebolt is a direct damage spell that immediately deals electricity damage to any entity that it's projectile hits
2. Lightning Bolt is a direct damage spell that immmediately deals electricity damage to all entities in a radius of 3 tiles
3. Punch is an ability that deals damage to any entity in an adjacent tile
4. Suplex is an ability that deals damage to any entity in an adjacent tile and moves them behind the caster

With the above, it may be aparent the challenges with the above scenario that abilities can cause a wide variety of effects that need to be accounted for. Accounting for them within a single system is very confusing and difficult and heavily impacts the parallelizability.

### Message Queues to the Rescue

To resolve this, the most effective way I have found is to divide the work up into stages.

Stage 1 - Validate Targets and Spawn projectiles/Effects
Stage 2 - Calculate the interrim effects based on caster/enemy stats, resistances, etc. to determine the __actualized__ effects on the entity
Stage 3 - Apply each actualized effect for each entity.

Each stage will have it's own system or set of systems that can be configured in the `Dispatcher` or `Scheduler` to ensure that all Stages are run to completion before starting the next stage. To facilitate this communication there's nothing better than a message queue.

### An Example:

As mentioned in previous articles, my ECS of choice is [SPECS Parallel ECS](https://github.com/amethyst/specs), so these examples will talk with that in mind.

As Specs is parallel in nature, this is a natural fit for the [crossbeam](https://docs.rs/crossbeam/latest/crossbeam/index.html) library which has MPMC queues which can be used to easily send these messages in a thread safe way.

We can insert a given queue into the world using the following pattern

```rs
enum MySystemMessage {
    // System's variants
}

let world = specs::World::new();
world.insert(SegQueue<SystemMessage>::new());
```

This may work for a number of systems, but it can be a little boilerplatey within the systems, in addition specs only allows a single instance of any Type inserted as a resource. For example: The following will panic.

```rs
let world = specs::World::new();
world.insert(SegQueue<SystemMessage>::new());
// Panic will occur here:
world.insert(SegQueue<SystemMessage>::new());
```


As specs only allows one resource of a type to be inserted, we can use the newtype pattern and implement the `Deref` operator for each in order to create the various world queues.

```rs
#[derive(Debug, Default)]
struct MySystemQueue(crossbeam::queue::SegQueue<MySystemMessage>);
impl Deref for SystemQueue {
    type Target = crossbeam::queue::SegQueue<MySystemMessage>
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

As this can get rather boilerplatey, I wrote a macro help automate this.

```rs
macro_rules! define_seg_queue {
    ($name:ident, $message_type:ty) => {
        #[derive(Debug, Default)]
        pub struct $name(crossbeam::queue::SegQueue<$message_type>);

        impl std::ops::Deref for $name {
            type Target = crossbeam::queue::SegQueue<$message_type>;
            fn deref(&self) -> &Self::Target {
                &self.0
            }
        }
    };
}

define_seg_queue!(MySystemQueue, MySystemMessage);
```

Now that the seg queue is located in the system, we can access it such as in the following pair of send/receive system.

```rs
struct MyTxSystem;

impl <'a> System <'a> for MyTxSystem {
    // ReadExpect<'a> because the system cannot function without the seg-queue
    type SystemData = (ReadExpect<'a, MySystemQueue>);
    fn run(&mut self, data: Self::SystemData) {
        let queue = data;
        // Do things and then when done 
        queue.push(
            // MySystemMessage variant here
        )
    }
}

struct MyRxSystem;
impl <'a> System <'a> for MyRxSystem {
    // WriteExpect<'a> in this case is unnecessary due to interior mutability. This is useful to acknolwedge that this system should be the consumer of the messages. 
    type SystemData = (WriteExpect<'a, MySystemQueue>);
    fn run(&mut self, data: Self::SystemData) {
        let queue = data;
        while Some(msg) = queue.pop() {
            match msg {
                // Handle each variant
            }
        }
    }
}
```

One important note is that due to `crossbeam`'s queues being MPMC they allow you to also create local parallized systems, such as via a thread queue if needed. In addition, this can allow you to specialize systems by having interrim "Dispatch" systems that work to translate specific message types.

## How it works in my game

For example in my game I use the following message queues for each stage:

```rs
// Pushed in Stage 1, resolved in Stage 2
define_seg_queue!(SpawnProjectileQueue, crate::systems::SpawnProjectileMessage);
define_seg_queue!(GameActionQueue, crate::systems::GameActionMessage);
// Pushed in Stage 2, resolved in Stage 3
define_seg_queue!(EffectDispatchQueue, crate::systems::EffectContainer);
```
Stage 1 consists of the `UseAbilitySystem` which handles any "Intents" related to ability usage and spell casting. An intent is an internal term that I use to differentiate "Input" from "Desired Outcome", this helps me to handle contextual prompts across any control configuration. This specific system looks at the ability and determines which system it should work from. At the moment, there are only `SpawnProjectileSystem` and `GameActionSystem`. 

`SpawnProjectileSystem` is a system that handles the spawning of projectiles, there are multiple systems that can feed Messages into it including the `UseAbilitySystem` projectiles can be spawned for a variety of reasons and are each tracked as their own entity in the system. Some projectiles represent simple "particle" effects while others may be arrows that deal damage, mines, etc.

The `GameActionSystem` is what handles calculating the game board rules, such as AoE, effects and calculations, if it gets more complicated in the future, I may break this system into a subset of systems. This system has receives messages from multiple queues including the `CollisionSystem` (projectiles can spawn GameActions), the `UseAbilitySystem` and the `StatusEffectSystem`. This system serves to translate the `GameActions` which is an enum that encapsulates the "Rule-based" game actions that I have defined in the game. These actions have their values calculated and the effects sent to the `EffectDispatchSystem`.

The `EffectDispatchSystem` is the point where mutation and change is committed into the system. These are the final changes to entities and the game world.

As an example Entity1 that casts the `Firebolt` ability with MagicPower 100 against EntityB which has 75% Magic damage reduction would result in a total effect of 25 health damage on EntityB. `GameActionSystem` handles the effect calculation while `EffectDispatchSystem` would handle decrementing the health of EntityB by 25 HP.

One thing that you may have noticed is that there is currently no mention of animation state, at the moment my game uses ascii graphics and animations are second class citizens. Fortunately, as the message queues keep these systems loosely coupled, there's no reason that it wouldn't be possible to add an animation component to the entities and only dispatch the effect when the animation is completely or reaches a specific frame.

For games that are real-time this may be a bit more important, but if you are using a hurtbox to track whether an attack should deal damage, there's no reason that you cannot add that in as a separate stage.

## Conclusion

With that, you can probably work our your own way to interlink your systems with message queues when encoding the data in your entities may not be appropriate. Thanks for reading!
