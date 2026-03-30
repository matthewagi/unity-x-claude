# Unity MCP Skill — Claude's Guide to Controlling Unity Editor

You have full control over Unity Editor via 24 MCP tools. This skill teaches you how to use them effectively.

## Core Principles

1. **Start shallow, drill deep.** Use `unity_get_scene` at depth=2 first. Only increase depth for objects you're actively modifying.
2. **Batch everything.** `unity_set_property` takes an array of operations — send 10+ changes in a single call instead of 10 separate calls.
3. **Everything is undoable.** All modifications go through Unity's Undo system. Users can Ctrl+Z any change you make.
4. **Use flexible IDs.** Most tools accept instance IDs (int), hierarchy paths ("Player/Weapon"), or names ("MainCamera"). Pick whichever is clearest.

## Tool Quick Reference

### Read State
| Tool | When to Use |
|------|-------------|
| `unity_get_scene` | Survey scene hierarchy, find objects |
| `unity_get_components` | Read component properties on a specific object |
| `unity_search_assets` | Find assets by name/type (e.g., `t:Material`, `t:Prefab player`) |
| `unity_get_asset` | Read asset details and properties |
| `unity_get_selection` | See what the user has selected |
| `unity_get_settings` | Read project settings (Physics, Player, Quality, etc.) |
| `unity_get_console` | Read errors, warnings, logs |

### Modify State
| Tool | When to Use |
|------|-------------|
| `unity_set_property` | Change ANY component property (batched) |
| `unity_create_gameobject` | Spawn objects with components in one call |
| `unity_delete` | Remove objects or assets |
| `unity_duplicate` | Clone objects |
| `unity_add_component` / `unity_remove_component` | Manage components |
| `unity_set_selection` | Select objects for user visibility |
| `unity_set_settings` | Change project settings |

### Create Content
| Tool | When to Use |
|------|-------------|
| `unity_create_script` | Generate C# MonoBehaviours, SOs, etc. |
| `unity_modify_script` | Edit existing scripts |
| `unity_create_asset` | Create Materials, ScriptableObjects |

### Editor Control
| Tool | When to Use |
|------|-------------|
| `unity_editor_command` | Play/Pause/Stop, Save, Undo, Redo, menu items |
| `unity_scene_view` | Control viewport camera |
| `unity_build` | Build for platform |
| `unity_manage_packages` | Add/remove UPM packages |
| `unity_execute_csharp` | Escape hatch for anything not covered |

## Common Workflows

### Create a Character with Physics
```
1. unity_create_gameobject: {name: "Player", components: [
     {type: "CapsuleCollider", properties: {m_Height: 2, m_Radius: 0.5}},
     {type: "Rigidbody", properties: {m_Mass: 70, m_Constraints: 112}}
   ], transform: {position: {x:0, y:1, z:0}}}
2. unity_create_script: {path: "Assets/Scripts/PlayerController.cs", class_name: "PlayerController",
   base_class: "MonoBehaviour", body: "...movement code..."}
3. unity_add_component: {object_id: "Player", component_type: "PlayerController"}
```

### Set Up a Scene
```
1. unity_get_scene: {depth: 1}  — see what's there
2. unity_create_gameobject (multiple calls for ground, lights, camera, objects)
3. unity_set_property: {operations: [...batch all material/physics/transform changes...]}
4. unity_editor_command: {command: "save_scene"}
```

### Modify Materials in Bulk
```
1. unity_search_assets: {filter: "t:Material"}
2. unity_set_property: {operations: [
     {object_id: <mat1_id>, property_path: "_Color", value: {r:1,g:0,b:0,a:1}},
     {object_id: <mat2_id>, property_path: "_Color", value: {r:0,g:1,b:0,a:1}},
     ...
   ]}
```

### Debug an Issue
```
1. unity_get_console: {filter_type: "error", max_entries: 20}
2. unity_get_scene: {filter_components: ["Rigidbody"]} — find physics objects
3. unity_get_components: {object_id: <suspect>, depth: 3} — inspect deeply
```

## Property Path Reference

Unity's SerializedProperty uses internal field names (prefixed with `m_`). Common mappings:

### Transform
- `m_LocalPosition` → local position
- `m_LocalRotation` → local rotation (quaternion)
- `m_LocalScale` → local scale

### Rigidbody
- `m_Mass`, `m_Drag`, `m_AngularDrag`
- `m_UseGravity`, `m_IsKinematic`
- `m_Constraints` (int bitmask: FreezePositionX=2, Y=4, Z=8, FreezeRotationX=16, Y=32, Z=64, FreezeAll=126)

### Colliders
- BoxCollider: `m_Size`, `m_Center`
- SphereCollider: `m_Radius`, `m_Center`
- CapsuleCollider: `m_Radius`, `m_Height`, `m_Direction`

### MeshRenderer
- `m_Materials` (array of material refs)
- `m_CastShadows`, `m_ReceiveShadows`

### Camera
- `m_FieldOfView`, `m_NearClipPlane`, `m_FarClipPlane`
- `m_Orthographic`, `m_OrthographicSize`
- `m_BackGroundColor`

### Light
- `m_Color`, `m_Intensity`, `m_Range`, `m_SpotAngle`
- `m_Type` (0=Spot, 1=Directional, 2=Point, 3=Area)
- `m_Shadows.m_Type` (0=None, 1=Hard, 2=Soft)

### AudioSource
- `m_audioClip`, `m_Volume`, `m_Pitch`
- `m_Loop`, `m_PlayOnAwake`, `m_SpatialBlend`

## Settings Categories

Use with `unity_get_settings` / `unity_set_settings`:

| Category | What It Controls |
|----------|-----------------|
| Physics | Gravity, default material, layer collision matrix |
| Player | Company name, product name, resolution, icon, scripting backend |
| Quality | Quality levels, shadows, anti-aliasing, vsync |
| Audio | Volume, speaker mode, DSP buffer size |
| Time | Fixed timestep, maximum allowed timestep, time scale |
| Graphics | Render pipeline, transparency sort, fog |
| Input | Input axes, button mappings |
| Tags | Tags, sorting layers, layers |
| Editor | Asset serialization mode, sprite packer |
| Navigation | NavMesh settings |

## Depth Strategy

- **depth=0**: Just names and IDs. Use for "what's in the scene?" surveys of large scenes.
- **depth=1**: Names + transform + component type list + tags. Good default for scene overview.
- **depth=2**: Component property summaries. Good for inspecting specific objects.
- **depth=3+**: Full nested property trees. Use only for deep inspection of specific components.

## Tips

- When setting `Quaternion` rotations, prefer euler angles: `{euler_x: 0, euler_y: 90, euler_z: 0}`
- Object references can be set by instance_id or asset_path
- The `unity_execute_csharp` tool is your escape hatch — use it for anything the other tools don't cover
- After creating scripts, call `unity_editor_command: {command: "refresh_assets"}` to trigger compilation
- Use `unity_editor_command: {command: "menu:GameObject/Align With View"}` to access any Unity menu item
