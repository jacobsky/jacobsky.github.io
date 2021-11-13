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

One option is to create some kind of Node2D/Node3D that is aware of the associated ECS Entity, normally Godot types won't be able to know about non-ecs, but with NativeScript, any Rust script can have Rust specific data that can also be used.

Pros
- Editor Compatible
- Nodes manage their SceneTree lifetime.
- Easier to implement
- Integrates easier with GDScript

Cons
- Not necessarily cache friendly
- Can be challenging to synchronize certain data
- Updating the SceneTree is __never__ thread safe

As can be seen in the examples directory of the  [gd-specs crate](https://github.com/jacobsky/godot-specs-integration/tree/master/gdnative/gd-specs/src/examples), there are two solutions outlined. The common functionality for both scenarios is as follows:

- The Node lifetime determines the entity lifetime, if the `GDEntity` is freed, it will also free it's corresponding entity in the world, the opposite is also true. 
- The World itself completes the update and then emits a signal which allows each individual node to update it's own state with two separate methods.

### GDEntity and GDWorld

In this scenario, the Node itself handles the lifetime as well as updating the individual information for the entity. In this case, all of the updates are completed via a signal that each individual node is responsible for.

Note: it would also be possible to have a single manager class that is responsible for updating each node's properties, which may lead to better performance than using signals.

### GDEntityHybrid/GDWorldHybrid

In this scenario, the entity data only exists in the node when set by the editor. The GDEntityHybrid then passes all the information to the ECS World and has no further tasks. It is a simple data container in the SceneTree. It still manages it's own lifetime based on the Entity lifetime.

To update the node state, Systems are used that reference the CanvasItem/Spatial Rid to make changes to it's position/rotation/scale via the `VisualServer`. This minimizes the number of calls into Godot and can lead to some major performance gains.

## 2. Synchronizing via Godot IO Layer (Server Singletons)

TODO: Finish this

Another much more advanced option is to use the Godot IO Layer also know as Server Singletons such as the VisualServer and AudioServer directly when attempting to synchronize everything related to the Godot state.

Pros
- Synchronization can be done directly by a System.
- Full control and optimizations for this.
- Minimal duplicate data (no SceneTree nodes).
Cons
- Requires a lot of additional work to integrate with the editor.
- Requires manually implementing a lot of functionality already available in Godot.

In this case, Systems will create the CanvasItems/Spatials with the necessary `Transform` data to properly render it into the server. The same happens for lighting, animations, audio playback etc.

Note: This also removes much of the editor configuration. For a middle-ground solution, I would highly recommend exploring the GDEntityHybrid option above.,

Here are some samples for how this can be accomplished in 2D

```rs
// Components/Resources
struct CanvasRoot(Rid);
struct SpriteSheet(Texture);
struct Material(Material)
struct Position { x: f32, y: f32};
struct Renderable;

#[derive(Debug, Component)]
#[storage(FlaggedStorage)]
#[allow(non_camel_case_types)]
pub struct CanvasItem {
    pub rid: Rid,
    // The canvas root
    pub parent_rid: Rid,
    // A reference needed to keep the material reference alive
    pub material: Option<Ref<Material>>,
    // A reference needed to keep the sprite reference alive
    pub sprite: Option<Ref<Texture>>,
    // Z sorting can be useful....
    pub z_index: i64,
}

struct CreateCanvasItemSystem;

impl<'a> System<'a> for CreateCanvasItemSystem {
    type SystemData = (
        Entities<'a>,
        ReadExpect<'a, CanvasRoot>,
        ReadExpect<'a, Material>
        ReadStorage<'a, Position>,
        ReadStorage<'a, Renderable>,
        WriteStorage<'a, CanvasItem>,
    );
    fn run(&mut self, data: Self::SystemData) {
        let (entities, canvas_root, positions, renderables, mut canvas_items) = data;
        let vs = unsafe { VisualServer::godot_singleton() };
        let parent_rid = canvas_root.0;

        // TODO: Convert these to proper resources loaded at initialization.
        let sprite_rsrc = ResourceLoader::godot_singleton()
            .load(crate::SPRITE_SHEET_RESOURCE, "", false)
            .expect("Resource not found")
            .cast::<Texture>()
            .expect("should be Texture");

        let material_rsrc = ResourceLoader::godot_singleton()
            .load(crate::ENTITY_SHADER_MATERIAL_RESOURCE, "", false)
            .expect("resource not found")
            .cast::<Material>()
            .expect("should be able to cast to Shader");

        let mut to_insert_canvas_items: Vec<(Entity, CanvasItem)> = Vec::new();

        // For every entity that has a position but not a canvas_item, we need to create it properly.
        for (entity, renderable, position, _canvas_item) in
            (&entities, &renderables, &positions, !&canvas_items).join()
        {
            // Create the canvas_item
            log::trace!("creating relevant rids");
            let rid = vs.canvas_item_create();
            vs.canvas_item_set_parent(rid, parent_rid);
            
            let transform = Vector2::new(
                position.x,
                position.y,
            ).to_transform();

            let shader_material_rsrc = unsafe {
                material_rsrc
                    .assume_safe()
                    .duplicate(false)
                    .expect("this should work")
                    .cast::<Material>()
                    .expect("should cast")
            };

            // Add the visibility mask
            // vs.canvas_item_set_light_mask(rid, crate::LIGHT_MASK_ENTITY_VISIBILITY_CULLING);

            // TODO: Move this into a lighting related thingy
            // Add the render order
            let z_index = renderable.render_order;
            vs.canvas_item_set_z_index(rid, z_index);
            
            // TODO: Move this into a canvas_item_z_index related thingy
            vs.canvas_item_set_light_mask(rid, crate::LIGHT_MASK_ENTITY_VISIBILITY_CULLING);

            // Put this information to the list to add to the entity.
            to_insert_canvas_items.push((
                entity,
                CanvasItem {
                    rid,
                    parent_rid,
                    material: Some(shader_material_rsrc.clone()),
                    sprite: Some(sprite_rsrc.clone()),
                    z_index,
                },
            ));
        }
        // Add them in after the loop to make the borrow checker happy.
        for (entity, canvas_item) in to_insert_canvas_items {
            canvas_items
                .insert(entity, canvas_item)
                .expect("Could not insert into the variable.");
        }
    }
}

#[allow(non_camel_case_types)]
pub struct PruneCanvasItemSystem;
impl<'a> System<'a> for PruneCanvasItemSystem {
    type SystemData = (
        Entities<'a>,
        ReadStorage<'a, Position>,
        WriteStorage<'a, CanvasItem>,
    );
    fn run(&mut self, data: Self::SystemData) {
        let (entities, positions, mut canvas_items) = data;
        let mut canvas_items_to_free = Vec::new();
        {
            for (entity, _, _) in (&entities, &canvas_items, !&positions).join() {
                canvas_items_to_free.push(entity);
            }
        }

        for entity in canvas_items_to_free {
            canvas_items.remove(entity);
        }
    }
}
```


Updating the canvas items is much simpler. The main work comes in the form of converting the `Position` into a `Transform2D` such as in the below example.

```rs
pub struct UpdateCanvasItemsSystem;
impl <'a> System <'a> for UpdateCanvasItemsSystem {
    type SystemData = (ReadStorage<'a, Position>, WriteStorage<'a, CanvasItem>);
    fn run(&mut self, data: Self::SystemData) {
        let (positions, canvas_items) = data;
        // Synchronize the position of any canvas items that exist

        let vs = unsafe { VisualServer::godot_singleton() };
        for (canvas_item, position) in (&canvas_items, position).join() {
            let transform = Transform2D::new(
                1.0, 0.0,
                0.0, 1.0,
                position.x, position.y
            )
            vs.canvas_item_set_transform(canvas_item.rid, transform);
        }
    }
}
```

These are not the only possibilities for handling synchronization, but I hope that it's enough to get you started.
