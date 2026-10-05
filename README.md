# Credits

Special thanks to **Ambiennt** for the flipbook/animated-item geometries and much of the first-person item animation math used in this pack.

[Ambiennt — GitHub](https://github.com/ambiennt)
[Ambiennt — YouTube](https://www.youtube.com/@ambiennt)

Thanks to **CrisXolt** for the insight and methodology behind rendering animated items in the inventory.

[CrisXolt — X](https://x.com/CrisXolt)

Thank you to both for sharing your work and making this system possible.

# How Animated Items Work — Animated Items

This is a Minecraft Bedrock resource-pack system for making item textures animate across the places where Bedrock allows us to manipulate their rendering.

## Rendering Contexts

Bedrock can render an item in several different contexts. For this template, three main contexts are important to understand:

1. **UI** — hotbar, inventory, containers, etc.

2. **First-person & third-person held item** — handled through attachables, render controllers, and player animations.

3. **Entity rendering** — used for thrown projectiles, dropped item entities, and item-frame item entities.

This template focuses primarily on the **UI**, **first-person/third-person held-item systems**, and **thrown projectiles**, which can be directly manipulated through the UI, attachable, and entity systems.

Entity rendering is a separate rendering system. It is relevant to animated items because some item states are represented as entities rather than held attachables.

For example, a thrown ender pearl is rendered as an **ender pearl projectile entity**, which can be given its own entity geometry, textures, render controller, and animations.

However, **dropped items and item-frame item entities cannot be manipulated through this system in the same way**. Their entity rendering is controlled by Bedrock's item/entity rendering behavior, and this template does not provide a method for replacing those representations with the same frame-by-frame animation system used for UI icons, held items, and projectile items.

Therefore, the rendering paths covered by this template are:

| Rendering context                     | Covered by this template                                                         |
| ------------------------------------- | -------------------------------------------------------------------------------- |
| UI item icon                          | Yes                                                                              |
| First-person & third-person held item | Yes                                                                              |
| Thrown projectile entities            | Yes                                                                              |
| Dropped item entities                 | No — animated dropped-item entities cannot be manipulated through this system    |
| Item-frame item entities              | No — animated item-frame item entities cannot be manipulated through this system |

The distinction is important: **an item being rendered as an entity does not automatically mean that its entity representation can be controlled like an attachable.**


---

# 1. The UI Item Icon

**Files:** `ui/*.json`

The UI is responsible for the 2D representation of an item in places such as:

* Hotbar
* Inventory
* Chests
* Creative inventory
* Other item slots using the vanilla item renderer

Bedrock already provides a built-in UI animation type called `flip_book`. This lets a single vertically stacked texture act as an animated texture.

The basic animation definition is:

```json
"uv_base": {
  "anim_type": "flip_book",
  "orientation": "vertical",
  "initial_uv": [ 0, 0 ],
  "frame_count": "$frame_count",
  "fps": 10,
  "easing": "linear"
}
```

The texture is divided vertically into individual frames.

For example, a 16×400 texture containing 25 16×16 frames becomes:

```text
┌────────────┐
│   Frame 1  │
├────────────┤
│   Frame 2  │
├────────────┤
│   Frame 3  │
├────────────┤
│     ...    │
├────────────┤
│  Frame 25  │
└────────────┘
```

The UI renderer then moves through those regions automatically.

## Item IDs and Global Variables

The reusable UI renderer needs two pieces of information for each animated item:

* The item's numeric runtime ID
* The path to its animated texture

These values are defined in `_global_variables.json`:

```json
{
  "$item_id_ender_pearl": 448,
  "$item_texture_ender_pearl": "textures/items/flipbook/ender_pearl/ender_pearl",

  "$item_id_flint_and_steel": 323,
  "$item_texture_flint_and_steel": "textures/items/flipbook/flint_and_steel/flint_and_steel"
}
```

The item IDs shown above are specifically for **Minecraft Bedrock 1.21.114**.

**Item IDs can change between Minecraft versions**, so these values should not be assumed to be universal. When porting this system to another version, you should verify the numeric IDs for that version.

A convenient way to do this is to use an item-ID visualization pack such as **Item ID Visualizer**:

[Item ID Visualizer — MCBE / MCPE](https://www.planetminecraft.com/texture-pack/item-id-visualizer-mcbe-mcpe/)

This allows the numeric ID of an item to be displayed in-game, making it easier to determine the correct value for the version of Bedrock being targeted.

The global variables are then referenced by the reusable renderer instead of hardcoding the values directly into every item definition.

## Xenon's Reusable Item Renderer

Instead of manually creating an entire UI implementation for every animated item, Xenon's UI code provides a reusable base:

```json
"item_base": {
  "type": "image",
  "size": [ "100%", "100%" ],
  "anchor_to": "center",
  "anchor_from": "center",
  "layer": 1,
  "texture": "$texture_path",
  "uv": "@item_renderer_xenon.uv_base",
  "uv_size": "$item_uv_size",
  "bindings": [
    {
      "binding_name": "#item_id_aux",
      "binding_type": "collection",
      "binding_collection_name": "$item_collection_name"
    },
    {
      "binding_type": "view",
      "source_property_name": "(#item_id_aux / 65536)",
      "target_property_name": "#item_runtime_id"
    },
    {
      "binding_type": "view",
      "source_property_name": "(#item_runtime_id = $number_item_id)",
      "target_property_name": "#visible"
    }
  ]
}
```

The important part is that the renderer is **generic**.

The item-specific information is supplied through variables:

```json
"ender_pearl@item_renderer_xenon.item_base": {
  "$number_item_id": "$item_id_ender_pearl",
  "$texture_path": "$item_texture_ender_pearl",
  "$item_uv_size": [ 16, 16 ],
  "$frame_count": 25
}
```

And another item can use the exact same base:

```json
"flint_and_steel@item_renderer_xenon.item_base": {
  "$number_item_id": "$item_id_flint_and_steel",
  "$texture_path": "$item_texture_flint_and_steel",
  "$item_uv_size": [ 32, 32 ],
  "$frame_count": 12
}
```

The renderer determines whether the overlay should be visible by comparing the currently displayed item's runtime ID with the configured item's numeric ID:

```json
"source_property_name": "(#item_runtime_id = $number_item_id)",
"target_property_name": "#visible"
```

This is what allows the same renderer to be reused for many different items without having to build a completely separate UI system for each one.

## The Overlay

The reusable renderer is inserted into the vanilla item renderer through an overlay:

```json
"overlay": {
  "type": "panel",
  "$item_layer|default": 1,
  "controls": [
    {
      "ender_pearl@item_renderer_xenon.ender_pearl": {
        "$item_layer": "$item_layer"
      }
    },
    {
      "flint_and_steel@item_renderer_xenon.flint_and_steel": {
        "$item_layer": "$item_layer"
      }
    }
  ]
}
```

The individual animated items are therefore just entries in the renderer.

Adding another animated item primarily means supplying:

* Its numeric item ID
* Its sprite-sheet texture
* Its frame dimensions
* Its frame count

The underlying UI animation system remains unchanged.

## Sprite Sheets

The UI frames are stored as vertical sprite sheets:

```text
textures/items/flipbook/<item>/<item>.png
```

For example:

```text
textures/items/flipbook/ender_pearl/ender_pearl.png
textures/items/flipbook/flint_and_steel/flint_and_steel.png
```

The sprite sheet is only needed for the UI system.

The in-hand renderer works differently because render controllers select **whole textures**, not individual regions of a sprite sheet.

That is why the attachable system uses separate `frame_N.png` files.

---

# 2. First-Person & Third-Person Item Attachables

**Files:**

```text
attachables/*.json
render_controllers/item.render_controllers.json
animations/items.wield.animation.json
entity/player.entity.json
animation_controllers/*.json
animations/*.json
```

This is the core of the animated-item system.

Unlike the UI, the actual held item is a **3D attachable**.

An attachable tells Bedrock:

> "When this particular item is being rendered, use this model, these textures, these animations, and these render controllers."

The attachable is identified using the vanilla item's identifier:

```json
"identifier": "minecraft:ender_pearl"
```

or:

```json
"identifier": "minecraft:flint_and_steel"
```

Because the attachable uses the vanilla identifier, Bedrock automatically associates it with that item when it is equipped.

This is what allows the resource pack to replace the normal held-item rendering with our custom animated version.

---

## How the attachable is structured

A simplified attachable looks like this:

```json
{
  "format_version": "1.21.0",
  "minecraft:attachable": {
    "description": {
      "min_engine_version": "1.8.0",
      "identifier": "minecraft:ender_pearl",

      "materials": {
        "default": "entity_alphatest",
        "enchanted": "entity_alphatest_glint"
      },

      "textures": {
        "default": "textures/items/ender_pearl",
        "item_frame_1": "textures/items/flipbook/ender_pearl/frame_1",
        "item_frame_2": "textures/items/flipbook/ender_pearl/frame_2",
        "item_frame_3": "textures/items/flipbook/ender_pearl/frame_3"
      },

      "geometry": {
        "default": "geometry.items",
        "entity": "geometry.item_sprite"
      },

      "animations": {
        "wield": "animation.items.wieldv2",
        "flying": "animation.actor.billboard"
      },

      "scripts": {
        "animate": [
          "wield",
          {
            "flying": "!query.is_attached"
          }
        ]
      },

      "render_controllers": [
        {
          "controller.render.ender_pearl": "query.is_attached"
        },
        {
          "controller.render.ender_pearl_entity": "!query.is_attached"
        }
      ]
    }
  }
}
```

There are several important pieces here.

### Identifier

```json
"identifier": "minecraft:ender_pearl"
```

This connects the attachable to the actual Minecraft item.

For another item, this must be changed to that item's vanilla identifier.

---

## Textures

The attachable declares every frame as its own texture:

```json
"textures": {
  "default": "textures/items/ender_pearl",

  "item_frame_1":
    "textures/items/flipbook/ender_pearl/frame_1",

  "item_frame_2":
    "textures/items/flipbook/ender_pearl/frame_2",

  "item_frame_3":
    "textures/items/flipbook/ender_pearl/frame_3"

  // ...
}
```

This is one of the most important differences between the UI and in-hand systems.

The UI can animate regions of a single sprite sheet.

The render controller cannot simply say:

> "Use the third 16×16 section of this 16×400 image."

Instead, the render controller selects a texture from an array.

Therefore, an animated attachable needs:

```text
frame_1.png
frame_2.png
frame_3.png
...
frame_N.png
```

This is also why Xenon's generated pack contains both:

```text
<item>.png
```

for the UI sprite sheet, and:

```text
frame_1.png
frame_2.png
...
frame_N.png
```

for the attachable.

---

## Geometry

The attachable determines which geometry is rendered.

For a held item:

```json
"geometry": {
  "default": "geometry.items"
}
```

For the dropped/entity version of the ender pearl:

```json
"geometry": {
  "default": "geometry.items",
  "entity": "geometry.item_sprite"
}
```

The held-item geometry is the model that the attachable uses in the player's or mob's hand.

The geometry itself contains the item quad/bone that the texture is rendered onto.

The attachable does **not** determine the animation frame by itself.

It provides the available textures.

The render controller decides **which texture is currently displayed**.

---

# Wield Animations

The attachable also uses a wield animation:

```json
"animations": {
  "wield": "animation.items.wieldv2"
}
```

This controls how the item is positioned, rotated, and scaled in the hand.

This is important because the same item model can look completely different depending on how Bedrock expects it to sit in the player's hand.

The template has two important vanilla-style wield poses:

```text
animation.items.wieldv1
animation.items.wieldv2
```

### `wieldv1`

`wieldv1` is generally used for **tools**, such as:

* Swords
* Pickaxes
* Axes
* Shovels

### `wieldv2`

`wieldv2` is generally used for **non-tool items**, such as:

* Projectile items, consumables, resources
* Other held utility items
* Similar item sprites

For example:

```json
"animation.items.wieldv2": {
  "loop": true,
  "bones": {
    "rightitem": {
      "position": [
        "c.is_first_person ? -1.13 : 2.0",
        "c.is_first_person ? 3.2 : 0.0",
        "c.is_first_person ? 1.13 : -5.5"
      ],
      "rotation": [
        {
          "x": "c.is_first_person ? -90.0 : -15.0"
        },
        {
          "z": "c.is_first_person ? -90.0 : 20.0"
        },
        {
          "y": "c.is_first_person ? 25.0 : 0.0"
        }
      ],
      "scale": "c.is_first_person ? 0.68 : 0.55"
    }
  }
}
```

Notice that the animation checks:

```molang
c.is_first_person
```

This means the same wield animation can provide **different positioning for first-person and third-person**.

This is why the first-person and third-person systems belong together: they are both rendering the same attachable, with the wield animation determining how that attachable is posed.

Choosing the correct wield animation matters significantly when adding a new item. A sword-like item and a pearl-like item should not necessarily use the same hand positioning.

---

# 3. Render Controllers

**File:**

```text
render_controllers/item.render_controllers.json
```

The render controller is the **second major piece of the in-hand animation system**.

The attachable provides the textures.

The render controller decides **which texture to display at any given moment**.

For example, the ender pearl provides:

```text
texture.item_frame_1
texture.item_frame_2
texture.item_frame_3
...
texture.item_frame_25
```

The render controller collects them into an array:

```json
"arrays": {
  "textures": {
    "array.item_frames": [
      "texture.item_frame_1",
      "texture.item_frame_2",
      "texture.item_frame_3",
      "texture.item_frame_4",
      "texture.item_frame_5",
      "texture.item_frame_6",
      "texture.item_frame_7",
      "texture.item_frame_8",
      "texture.item_frame_9",
      "texture.item_frame_10",
      "texture.item_frame_11",
      "texture.item_frame_12",
      "texture.item_frame_13",
      "texture.item_frame_14",
      "texture.item_frame_15",
      "texture.item_frame_16",
      "texture.item_frame_17",
      "texture.item_frame_18",
      "texture.item_frame_19",
      "texture.item_frame_20",
      "texture.item_frame_21",
      "texture.item_frame_22",
      "texture.item_frame_23",
      "texture.item_frame_24",
      "texture.item_frame_25"
    ]
  }
}
```

The controller then uses Molang to select one of those textures:

```json
"textures": [
  "temp.frame = math.mod(math.floor(query.time_stamp * 0.87), 24); return array.item_frames[temp.frame];",
  "texture.enchanted"
]
```

The important expression is:

```molang
math.mod(math.floor(query.time_stamp * 0.87), 24)
```

`query.time_stamp` continuously increases with world time.

Multiplying it by `0.87` controls how quickly the animation advances.

`math.floor()` converts that into an integer frame index.

`math.mod()` wraps the index around so the animation loops.

Conceptually:

```text
World time
    ↓
query.time_stamp
    ↓
* 0.87
    ↓
math.floor()
    ↓
math.mod(frame_count)
    ↓
array.item_frames[index]
    ↓
Current texture
```

This means the animation does not need per-item state or scripting.

The frame is derived from the current world timestamp.

---

## Important frame-count detail

The divisor in `math.mod()` determines how many array entries are actually cycled through.

For example:

```molang
math.mod(value, 12)
```

produces indices:

```text
0 → 1 → 2 → ... → 11 → 0
```

So a 12-frame animation should use:

```molang
math.mod(..., 12)
```

Likewise, a 25-frame animation should use:

```molang
math.mod(..., 25)
```

The current example controller contains:

```molang
math.mod(math.floor(query.time_stamp * 0.87), 24)
```

even for the 25-frame ender pearl.

Because arrays are zero-indexed, the values generated by `mod 24` are:

```text
0 through 23
```

meaning the 25th texture entry is never selected.

Therefore, when generating a new animated item, the `math.mod()` divisor should match the **actual number of frames intended to be animated**.

---

## Enchantment glint

The render controller can also switch between the normal material and the enchanted material:

```json
"materials": [
  {
    "*": "variable.is_enchanted ? material.enchanted : material.default"
  }
]
```

The attachable defines those materials:

```json
"materials": {
  "default": "entity_alphatest",
  "enchanted": "entity_alphatest_glint"
}
```

This allows the animated item to retain Minecraft's enchantment glint while its underlying texture continues changing frames.

The render controller can therefore handle both:

```text
Animated frame
+
Enchantment glint
```

at the same time.

---

## Entity / Thrown Ender Pearl Rendering

The ender pearl attachable contains a second render controller:

```json
{
  "controller.render.ender_pearl_entity": "!query.is_attached"
}
```

This controller is **not used for dropped-item rendering**. It is used when the ender pearl is being represented as the **thrown projectile entity**.

The attachable uses:

```text
query.is_attached
```

to determine whether it is currently attached to a player or mob.

When the ender pearl has been thrown, it is no longer attached, so:

```text
!query.is_attached
```

becomes true and the entity render controller is used.

The entity version of the ender pearl uses a different geometry:

```json
"geometry": "geometry.entity"
```

while the held attachable uses:

```json
"geometry": "geometry.default"
```

The corresponding `minecraft:ender_pearl` client entity provides the projectile's entity rendering setup:

```json
{
  "format_version": "1.10.0",
  "minecraft:client_entity": {
    "description": {
      "identifier": "minecraft:ender_pearl",

      "materials": {
        "default": "ender_pearl"
      },

      "textures": {
        "default": "textures/items/ender_pearl"
      },

      "geometry": {
        "default": "geometry.item_sprite"
      },

      "render_controllers": [
        "controller.render.item_sprite"
      ],

      "animations": {
        "flying": "animation.actor.billboard"
      },

      "scripts": {
        "animate": [
          "flying"
        ]
      }
    }
  }
}
```

The important distinction is:

| State                              | Rendering path                            |
| ---------------------------------- | ----------------------------------------- |
| Ender pearl held by the player     | Attachable + `query.is_attached`          |
| Ender pearl thrown as a projectile | Ender pearl entity + `!query.is_attached` |

The `flying` animation uses `animation.actor.billboard` so the thrown ender pearl maintains the expected billboarded entity behavior while it is flying.

Therefore, the `controller.render.ender_pearl_entity` controller should be understood as the **thrown-projectile rendering path**, not as a general dropped-item or item-frame rendering system.

---

# First-Person Animation System

The attachable handles the basic model and wield pose, but first-person rendering has another problem:

**Bedrock's normal attachable attack/swing animation does not provide a reliable result for this type of custom animated item.**

The attachable's attack rotation can have a broken or incorrect swing animation.

Because of that, the template uses the player's own first-person animation system to apply the correct motion.

These files work together:

```text
entity/player.entity.json
animation_controllers/first_person.flipbook_item.animation_controller.json
animations/player_firstperson.animation.json
```

The player animation system handles things such as:

* First-person item positioning
* Attack/swing rotation
* Item switching
* Arm positioning
* First-person-specific offsets

---

# `player_firstperson.animation.json`

The animation file contains the actual first-person operations.

For example, the attack rotation:

```json
"animation.player.first_person_attack_rotation_flipbook_item": {
  "loop": true,
  "bones": {
    "rightitem": {
      "position": [
        "math.sin(math.sqrt(v.attack_time) * 180.0) * -9.0",
        "math.sin(math.sqrt(v.attack_time) * 360.0) * 4",
        "math.sin(v.attack_time * 180.0) * 3.2"
      ],
      "rotation": [
        {
          "y": "(math.sin(math.pow(v.attack_time, 2.0) * 180.0) * 20.0) + -45.0"
        },
        {
          "z": "math.sin(math.sqrt(v.attack_time) * 180.0) * -20.0"
        },
        {
          "x": "math.sin(math.sqrt(v.attack_time) * 180.0) * 80.0"
        },
        {
          "y": 45
        }
      ]
    }
  }
}
```

This uses:

```molang
v.attack_time
```

to calculate the attack animation.

The result is a custom first-person swing that is applied directly to the item bone.

---

## Main-hand positioning

The first-person main-hand animation establishes the base first-person position:

```json
"animation.player.first_person_main_hand_flipbook_item": {
  "loop": true,
  "override_previous_animation": true,
  "bones": {
    "rightarm": {
      "position": [
        "-this",
        "v.short_arm_offset_right + -22.0",
        "-this"
      ],
      "rotation": [
        "-this",
        "-this",
        "-this"
      ]
    },
    "rightitem": {
      "position": [
        "8.96 - this",
        "19.2 - this",
        "11.52 - this"
      ],
      "rotation": [
        "-this",
        "180.0 - this",
        "-this"
      ]
    }
  }
}
```

The important part here is:

```json
"override_previous_animation": true
```

This lets the custom first-person animation replace conflicting vanilla animation values for these animated items.

---

## Item switching

The swap animation handles the arm movement when changing items:

```json
"animation.player.first_person_swap_flipbook_item": {
  "loop": true,
  "bones": {
    "leftarm": {
      "position": [
        0,
        "query.get_equipped_item_name(1) == 'filled_map' ? 0.0 : (1.0 - variable.player_arm_height) * -10.0",
        0
      ]
    },
    "rightarm": {
      "position": [
        0,
        "(1.0 - variable.player_arm_height) * -10.0",
        0
      ]
    }
  }
}
```

This keeps item switching compatible with the normal first-person arm movement.

---

# First-Person Animation Controller

**File:**

```text
animation_controllers/first_person.flipbook_item.animation_controller.json
```

The animation controller determines **which items actually receive the custom first-person animations**.

The controller checks:

```molang
q.get_equipped_item_name
```

For the current template:

```json
"animations": [
  {
    "first_person_main_hand_flipbook_item":
      "q.get_equipped_item_name == 'ender_pearl' || q.get_equipped_item_name == 'flint_and_steel'"
  },
  {
    "first_person_attack_rotation_flipbook_item":
      "q.get_equipped_item_name == 'ender_pearl' || q.get_equipped_item_name == 'flint_and_steel'"
  },
  {
    "first_person_swap_flipbook_item":
      "q.get_equipped_item_name == 'ender_pearl' || q.get_equipped_item_name == 'flint_and_steel'"
  }
]
```

This is essentially the filter that says:

> Only apply these custom first-person animations when one of our animated items is equipped.

When adding another item, its name should normally be added to **all three conditions**.

For example:

```molang
q.get_equipped_item_name == 'ender_pearl'
|| q.get_equipped_item_name == 'flint_and_steel'
|| q.get_equipped_item_name == 'new_item'
```

This prevents the custom first-person system from affecting unrelated items.

---

# Connecting Everything Through `player.entity.json`

The final step is making the animations known to the player entity.

**File:**

```text
entity/player.entity.json
```

The relevant animation definitions are:

```json
"animations": {
  "first_person_controller_flipbook_item":
    "controller.animation.first_person_flipbook_item",

  "first_person_main_hand_flipbook_item":
    "animation.player.first_person_main_hand_flipbook_item",

  "first_person_attack_rotation_flipbook_item":
    "animation.player.first_person_attack_rotation_flipbook_item",

  "first_person_swap_flipbook_item":
    "animation.player.first_person_swap_flipbook_item"
}
```

These names connect the player entity to the actual animation controller and animations.

The relationship is:

```text
player.entity.json
        │
        ├── first_person_controller_flipbook_item
        │          │
        │          ▼
        │   first_person.flipbook_item.animation_controller.json
        │          │
        │          ├── main hand animation
        │          ├── attack rotation
        │          └── item swap
        │
        ▼
player_firstperson.animation.json
```

This is what completes the first-person side of the attachable system.

---

# Complete Rendering Flow

For a held animated item, the complete chain is:

```text
Vanilla Item
     │
     ▼
Attachable
     │
     ├── Item identifier
     ├── Frame textures
     ├── Geometry
     └── Wield animation
             │
             ▼
     Render Controller
             │
             ├── Frame texture array
             ├── World-time frame selection
             └── Enchantment material
             │
             ▼
       Animated Item
             │
             ├── Third person
             │
             └── First person
                    │
                    ▼
              Player animations
                    │
                    ├── Base positioning
                    ├── Attack rotation
                    └── Item swapping
```

The UI follows a separate path:

```text
Item ID
   │
   ▼
Xenon UI Renderer
   │
   ├── Texture
   ├── UV size
   └── Frame count
   │
   ▼
UI flipbook
   │
   ▼
Hotbar / Inventory icon
```

And dropped items/item frames use their own entity rendering path.

---

# Adding a New Animated Item

To add another item to the template, the item needs to be registered across the appropriate systems.

## 1. Create the animation frames

Create:

```text
textures/items/flipbook/new_item/
```

with:

```text
frame_1.png
frame_2.png
frame_3.png
...
frame_N.png
```

Then create the UI sprite sheet:

```text
new_item.png
```

with all frames stacked vertically.

---

## 2. Add the UI definition

Add the item's numeric ID and texture path to the UI variables.

Then create an item definition using the existing Xenon base:

```json
"new_item@item_renderer_xenon.item_base": {
  "$number_item_id": "$item_id_new_item",
  "$texture_path": "$item_texture_new_item",
  "$item_uv_size": [ 16, 16 ],
  "$frame_count": 20
}
```

Then add it to the overlay:

```json
{
  "new_item@item_renderer_xenon.new_item": {
    "$item_layer": "$item_layer"
  }
}
```

The underlying UI renderer does not need to be recreated.

---

## 3. Create the attachable

Create:

```text
attachables/new_item.attachable.json
```

Set:

```json
"identifier": "minecraft:new_item"
```

Add every frame texture:

```json
"textures": {
  "default": "textures/items/new_item",
  "item_frame_1": "textures/items/flipbook/new_item/frame_1",
  "item_frame_2": "textures/items/flipbook/new_item/frame_2"
}
```

and continue through the final frame.

Choose the appropriate geometry and wield animation.

For example, a tool would generally use:

```json
"wield": "animation.items.wieldv1"
```

while a non-tool item would generally use:

```json
"wield": "animation.items.wieldv2"
```

The correct choice depends on how the item is supposed to sit in the third-person hand.

---

## 4. Add the render controller

Create a frame array:

```json
"array.item_frames": [
  "texture.item_frame_1",
  "texture.item_frame_2",
  "texture.item_frame_3"
]
```

and make sure the Molang frame calculation uses the intended frame count:

```molang
math.mod(math.floor(query.time_stamp * 0.87), FRAME_COUNT)
```

The divisor should correspond to the number of frames being cycled.

---

## 5. Add the first-person animations

Add the new item to the conditions in:

```text
animation_controllers/first_person.flipbook_item.animation_controller.json
```

Normally, the new item should be included in all three:

```text
first_person_main_hand_flipbook_item
first_person_attack_rotation_flipbook_item
first_person_swap_flipbook_item
```

This gives the item the complete first-person treatment rather than only the base attachable pose.

---

## 6. Register the player animations

Make sure the required animation/controller names are registered in:

```text
entity/player.entity.json
```

The existing player animation infrastructure can then be reused by the new item.

---

# The Important Concept

The system is not one giant animation.

It is several Bedrock rendering systems working together:

```text
                    ANIMATED ITEM
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
         UI       Held Attachable     Entity
          │              │
          │              ├── Wield
          │              ├── Render Controller
          │              └── First Person
          │                    │
          │                    ├── Main Hand
          │                    ├── Attack
          │                    └── Swap
          │
          ▼
     Flipbook UV
```

The **attachable is the foundation of the held-item system**.

It establishes the item identifier, model, frame textures, materials, wield animation, and render controllers.

The **render controller is what actually turns those textures into an animation** by selecting a frame over time.

The **player first-person animation system then fixes the parts that an attachable alone cannot reliably handle**, particularly the first-person attack/swing behavior and other local-player operations.

Finally, Xenon's UI renderer handles the completely separate 2D representation used by the hotbar and inventory.
