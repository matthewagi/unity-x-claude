# Unity MCP — Claude Workflow Instructions

## Overview

This plugin gives Claude direct control over Unity Editor via MCP (Model Context Protocol). Claude connects through a Node.js stdio bridge that forwards tool calls to Unity's HTTP server running inside the Editor. There are 24 tools covering scene hierarchy, components, assets, scripts, settings, build, and arbitrary C# execution.

---

## 1. Setup — New Project

### Step 1: Get the Unity project location from the user

Ask: "What's the full path to your Unity project?"

The plugin lives as an embedded UPM package at:
```
<ProjectRoot>/Packages/com.claude.unity-mcp/
```

### Step 2: Configure Claude Desktop

The user needs to add the MCP server to their Claude Desktop config. The config file is at:

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

Add this to `mcpServers`:
```json
{
  "mcpServers": {
    "unity": {
      "command": "node",
      "args": [
        "<ProjectRoot>/Packages/com.claude.unity-mcp/mcp-bridge.mjs"
      ]
    }
  }
}
```

**CRITICAL**: The `args` path must point to the `mcp-bridge.mjs` file inside the **currently open** Unity project. If the user switches projects, this path must be updated and Claude Desktop restarted.

### Step 3: Verify connection

After the user restarts Claude Desktop (or starts a new chat), test the connection:
```
unity_get_scene  (depth: 0)
```
This should return the scene hierarchy. If it fails with "Cannot connect", Unity isn't running or the plugin didn't load.

---

## 2. Architecture

```
Claude Desktop
    ↕ (stdio, JSON-RPC)
mcp-bridge.mjs  (Node.js process)
    ↕ (HTTP POST to localhost:9999/mcp)
Unity Editor  (StreamableHttpServer → MCPServer → Tool dispatch → MainThreadDispatcher)
```

**Key points:**
- `mcp-bridge.mjs` handles `initialize`, `ping`, `notifications/*` locally. Only `tools/list` and `tools/call` go to Unity.
- All tool execution happens on Unity's main thread via `MainThreadDispatcher` (queued into `EditorApplication.update`).
- The server auto-restarts after domain reloads (assembly recompilation). There may be a 1-5 second gap where calls fail — just retry.
- Port 9999 is default. Changed via `EditorPrefs.SetInt("MCP_Port", <port>)`.

---

## 3. How to Manipulate the Scene — DO THIS, NOT THAT

### DO: Use MCP tools directly to build scenes

Create objects, add components, set properties, and wire references all through tool calls. This is fast (no compilation cycle) and gives immediate feedback.

**Example — creating 4 colored cubes with physics:**

```
Step 1: Create primitives via execute_csharp (see "Primitives" gotcha below)
Step 2: Add components via unity_add_component
Step 3: Set properties via unity_set_property
Step 4: Wire references via unity_set_property with object_reference values
Step 5: Play and verify via unity_editor_command + runtime data checks
```

### DON'T: Write Editor scripts with [MenuItem] to set up scenes

Writing .cs files to `Assets/` triggers a domain reload cycle:
1. File written → Unity detects change → recompilation starts
2. MCP server stops (beforeAssemblyReload)
3. 3-10 seconds of downtime
4. Server restarts (afterAssemblyReload)
5. THEN you can run the menu item

This is slow and fragile. Only write scripts when you need **runtime behavior** (MonoBehaviours that run during Play mode) or **reusable editor tools**.

---

## 4. Creating Primitives (KNOWN GOTCHA)

`unity_create_gameobject` with the `primitive` parameter is **BROKEN** — it creates objects with only a Transform component. The MeshFilter, MeshRenderer, and Collider get stripped.

**Root cause**: Something in the tool dispatch context (possibly related to Undo group operations or UPM package compilation caching) strips components from `GameObject.CreatePrimitive()` when called through the SceneTools code path.

**Workaround**: Use `unity_execute_csharp` to create primitives:

```csharp
// This ALWAYS works
var go = GameObject.CreatePrimitive(PrimitiveType.Cube);
go.name = "MyCube";
go.transform.position = new Vector3(0, 1, 0);

// Set color (URP project)
var mat = new Material(Shader.Find("Universal Render Pipeline/Lit"));
mat.SetColor("_BaseColor", Color.red);
go.GetComponent<MeshRenderer>().sharedMaterial = mat;

// Remove collider if you don't want friction
UnityEngine.Object.DestroyImmediate(go.GetComponent<BoxCollider>());

return $"Created {go.name}, id={go.GetInstanceID()}";
```

**IMPORTANT**: In execute_csharp, use `UnityEngine.Object.DestroyImmediate()` not just `Object.DestroyImmediate()` — there's an ambiguity between `System.Object` and `UnityEngine.Object`.

After creating primitives via execute_csharp, you can use the normal MCP tools (add_component, set_property, etc.) on them.

---

## 5. Getting Runtime Data During Play Mode

### The Problem
`unity_get_components` returns SerializedProperty data, which is **STALE during Play mode**. Properties like `m_LocalPosition` won't update for physics-driven objects.

### The Solution
ComponentTools automatically appends a `runtime` section when `EditorApplication.isPlaying` is true:

```json
{
  "components": [...],
  "runtime": {
    "position": {"x": 1.5, "y": 1.0, "z": -2.3},
    "rotation": {"x": 0, "y": 45, "z": 0},
    "velocity": {"x": 0.5, "y": 0, "z": -1.2, "magnitude": 1.3},
    "is_sleeping": false
  }
}
```

### For Custom Runtime Data
Use `unity_execute_csharp` to read any runtime state:

```csharp
var go = GameObject.Find("MyCube");
var rb = go.GetComponent<Rigidbody>();
return $"pos={go.transform.position}, vel={rb.linearVelocity}";
```

---

## 6. Setting Object References (Transform targets, etc.)

To wire a reference (e.g., ChaseCube.target → another object's Transform), use `unity_set_property` with an `object_reference` value:

```
unity_set_property operations: [
  {
    "object_id": "CubeA",
    "component_type": "ChaseCube",
    "property_path": "target",
    "value": {"_type": "object_reference", "instance_id": <Transform instance_id>}
  }
]
```

**How to find instance IDs**: When you create objects, the response includes `instance_id`. A GameObject's Transform is typically `instance_id - 2` from the GameObject's ID (e.g., GO id=-34480 → Transform id=-34482). Verify with `unity_get_components`.

---

## 7. Testing Pipeline — ALWAYS VERIFY

Never assume things work. You MUST verify visually and programmatically.

### CRITICAL PREREQUISITE: Run In Background

Before ANY Play mode testing, ensure `runInBackground` is enabled. Without this, Unity does NOT advance game frames when focus is elsewhere (e.g., during MCP calls), meaning coroutines, physics, animations — nothing runs.

```csharp
// Run this ONCE at the start of any session:
Application.runInBackground = true;
PlayerSettings.runInBackground = true;
```

### Before Play:
1. **Check scene**: `unity_get_scene depth=0` — verify all objects exist
2. **Check components**: `unity_get_components object_id="X" depth=3` — verify components and properties
3. **Check references**: Verify target/reference fields point to the right objects
4. **Check input system**: `unity_get_settings category="Player" properties=["activeInputHandler"]`
   - `0` = Legacy (OnMouseDown works)
   - `1` = New Input System only (OnMouseDown does NOT work — use `Mouse.current` + raycast)
   - `2` = Both

### During Play:
1. **Start play**: `unity_editor_command command="play"`
2. **Wait 1-2 seconds** for initialization
3. **Verify frames are ticking**: `return $"frameCount={Time.frameCount}, time={Time.time}";`
   - If `frameCount=1` and `time=0`, `runInBackground` is off — fix it
4. **Read runtime data**: `unity_get_components` (check `runtime` section) or `unity_execute_csharp`
5. **Take screenshots**: Use `MCPTestHelper.CaptureScreenshot()` to SEE what the camera sees
6. **Describe the view**: Use `MCPTestHelper.DescribeView()` to get screen positions of all visible objects

### Visual Verification (Screenshots):

The project includes `Assets/Scripts/Editor/MCPTestHelper.cs` with these static methods:

```csharp
// Take a screenshot of what the main camera sees
MCPTestHelper.CaptureScreenshot(960, 540, "my_shot.png")
// Returns file path — read it with the Read tool to view the image

// Describe what's on screen (object names, positions, screen regions)
MCPTestHelper.DescribeView()

// Click an object by name (raycasts from camera to object center)
MCPTestHelper.ClickObject("Cube3")

// Click at specific screen coordinates
MCPTestHelper.SimulateClick(480, 270)
```

**Screenshot verification pattern:**
```
1. execute_csharp → MCPTestHelper.CaptureScreenshot(..., "before.png")
2. Read tool → view before.png (you can SEE the image)
3. execute_csharp → trigger the action
4. sleep 1-2 seconds (let animation/physics run)
5. execute_csharp → MCPTestHelper.CaptureScreenshot(..., "after.png")
6. Read tool → view after.png (visually confirm the change)
```

### Simulated Interaction Testing:

For click/interaction testing, scripts must expose a `SimulateClick()` public method:

```csharp
// In your MonoBehaviour:
public void SimulateClick()
{
    // Same logic as your click handler
    StartCoroutine(DoClickAction());
}
```

Then trigger from execute_csharp:
```csharp
var go = GameObject.Find("MyButton");
go.GetComponent<MyScript>().SimulateClick();
return "Clicked";
```

**IMPORTANT**: Between separate `execute_csharp` calls, Unity's update loop runs (frames tick, coroutines advance, physics steps). This is how you test: trigger action in one call, wait (sleep), verify result in next call.

### Verifying Visual Layout (e.g., card games, UI):

For games where visual appearance matters (poker, board games, UI layouts):

```csharp
// Check if objects are positioned correctly on screen
MCPTestHelper.DescribeView()
// Returns: "Visible: [Card1(scale=(1,1,1)), Card2(scale=...)]"
// with screen coordinates and regions (top-left, center, bottom-right, etc.)

// Take a screenshot and READ it to visually verify layout
var path = MCPTestHelper.CaptureScreenshot(1920, 1080, "layout_check.png");
```

Then use the `Read` tool on the screenshot PNG — you will see the actual rendered image and can verify card positions, text, colors, perspective, etc.

### Smart Snapshots (MCPTestRunner):

The goal of testing is to **prove things work with the minimum number of well-placed screenshots**. The number is YOUR judgment call — could be 2 for a simple color change, 5 for an animation, or 8 for a complex multi-step interaction. Think like a storyboard: what are the key moments that tell the whole story?

`MCPTestRunner` lives at `Assets/Scripts/MCPTestRunner.cs` and runs during Play mode.

**Core philosophy:** You know what the animation/action does (you wrote or read the script). Use that knowledge to place shots at exactly the right timestamps. Don't blindly record hundreds of frames.

**API:**
```csharp
MCPTestRunner.Clear()                              // reset previous snapshots
MCPTestRunner.Snap("label", trackObjects)           // immediate screenshot
MCPTestRunner.Click("ObjectName")                   // trigger interaction
MCPTestRunner.SnapSequence(times, labels, tracked)  // multiple timed shots
MCPTestRunner.GetResults()                          // summary of all shots
```

**Example — testing a 0.6s vanish animation:**
The script pops scale to 1.3x in the first 10%, then bounces down to 0 over 90%. So the right shots are:

```csharp
MCPTestRunner.Clear();
var tracked = new[] { "Cube3" };

// 1. Before
MCPTestRunner.Snap("1_before", tracked);

// 2. Click + schedule shots at animation keyframes
MCPTestRunner.Click("Cube3");
MCPTestRunner.SnapSequence(
    new[] { 0.04f,       0.2f,           0.45f,            0.7f },
    new[] { "2_pop",     "3_mid_shrink", "4_almost_gone",  "5_after" },
    tracked
);
```
Then sleep 1-2s, call `GetResults()`, and `Read` the screenshots.

**How to choose snapshot moments — think by situation:**

| Situation | Key moments to capture |
|-----------|----------------------|
| Click-to-vanish | before, pop, mid-shrink, almost-gone, after-destroy |
| Object spawning | empty scene, object appears, settled in place |
| Physics movement | start position, mid-flight, landing, at rest |
| Card game deal | hand empty, cards mid-air, cards fanned out, final layout |
| UI transition | old state, mid-transition, new state |
| Color/material change | before, during blend, after |

**For very fast animations (< 0.1s):** The editor may run at only 10-30fps, meaning a 0.06s pop could happen in a single frame. Options:
- Slow down time: `Time.timeScale = 0.2f;` before triggering, restore after
- Accept you can't see that frame and verify the result instead
- Place shots tighter: 0.01s, 0.03s, 0.06s

**For complex interactions (poker game, board game):** Don't try to capture every card flip. Instead:
1. Screenshot the initial deal layout — verify card positions
2. Screenshot after player action — verify state change
3. Screenshot final state — verify win/loss display

Screenshots saved to `ProjectRoot/Screenshots/` at 480x270. Use `Read` tool to view PNGs directly.

**Full test pattern:**
```
1. execute_csharp → Clear + Snap("before") + Click + SnapSequence(...)
2. sleep (animation_duration + 0.5s buffer)
3. execute_csharp → GetResults()   (verify data: scales, exists, timing)
4. Read tool → view key screenshots (verify visually)
5. execute_csharp → clean up screenshots when done
```

**IMPORTANT: Always clean up screenshots after each test.**
Once you've reviewed the results, delete ALL test screenshots via `execute_csharp` (not bash — the mounted folder doesn't have delete permissions, but Unity does):
```csharp
var dir = System.IO.Path.Combine(Application.dataPath, "..", "Screenshots");
var files = System.IO.Directory.GetFiles(dir, "*.png");
foreach (var f in files) System.IO.File.Delete(f);
return $"Deleted {files.Length} screenshots";
```

### If things don't work:
1. **Check console**: `unity_get_console` for errors
2. **Check frame advancement**: `Time.frameCount` should increase between calls. If not → `runInBackground` is off
3. **Check serialized vs runtime values**: A script's `Start()` may override serialized values
4. **Check input system**: `OnMouseDown` only works with legacy input (activeInputHandler=0 or 2)
5. **Stop and fix**: `unity_editor_command command="stop"`, make changes, play again

---

## 8. SerializedProperty Paths

Common property paths for `unity_set_property`:

| Component | Property | Path | Notes |
|-----------|----------|------|-------|
| Transform | Position | `m_LocalPosition` | Stale during Play |
| Transform | Rotation | `m_LocalRotation` | Quaternion |
| Transform | Scale | `m_LocalScale` | |
| Rigidbody | Mass | `m_Mass` | |
| Rigidbody | Drag | `m_LinearDamping` | NOT `m_Drag` |
| Rigidbody | Angular Drag | `m_AngularDamping` | |
| Rigidbody | Use Gravity | `m_UseGravity` | bool |
| Rigidbody | Is Kinematic | `m_IsKinematic` | bool |
| Rigidbody | Constraints | `m_Constraints` | Bitmask (122 = freeze rotation + Y position) |
| MeshRenderer | Materials | `m_Materials` | Array |
| Custom script | Public field | Field name directly | e.g., `chaseForce`, `target` |

**Custom MonoBehaviour fields** use their C# field name directly as the property path (not m_ prefix).

---

## 9. Domain Reloads and Server Lifecycle

When Unity recompiles scripts:
1. `beforeAssemblyReload` fires → server stops
2. Assemblies reload (2-10 seconds)
3. `afterAssemblyReload` fires → server restarts
4. Safety net: `EnsureServerRunning` checks every 5 seconds

**What triggers recompilation:**
- Writing/modifying .cs files in Assets/
- Writing/modifying .cs files in Packages/ (including this plugin)
- Adding/removing packages
- `unity_import_asset` with `force_update` on script files

**What does NOT trigger recompilation:**
- `unity_execute_csharp` (in-memory compilation via CodeDomProvider)
- Changing scene objects, components, properties
- Creating/modifying non-script assets (materials, textures, etc.)

**When calls fail with "Cannot connect"**: Just wait 2-5 seconds and retry. The server is restarting after a domain reload.

---

## 10. Port Conflicts

If port 9999 is already in use (e.g., previous Unity instance didn't clean up):

1. The server logs a warning once: `[MCP] Port 9999 is in use...`
2. It retries every 5 seconds via `EnsureServerRunning`
3. To fix: close the other Unity project, or change the port:
   ```
   unity_execute_csharp code="EditorPrefs.SetInt(\"MCP_Port\", 9998); return \"Port set to 9998\";"
   ```
   Then update `mcp-bridge.mjs` UNITY_PORT constant to match.

**Prevention**: The server registers `ProcessExit` and `DomainUnload` handlers to release the port when Unity closes.

---

## 11. execute_csharp Notes

- Uses `CodeDomProvider` for in-memory compilation — no temp files, no domain reload
- Has access to ALL Unity APIs: `UnityEngine`, `UnityEditor`, `System`, `System.Linq`
- Use `return <expression>;` to get a result back
- Use `Debug.Log()` for output (captured in `output` field)
- First compilation may take ~5 seconds (cold start); subsequent calls are faster
- **Safety blocks**: `Process.Start`, `Process.Kill`, `Environment.Exit`, `Registry.*`, `EditorApplication.Exit`
- **Ambiguity fix**: Always use `UnityEngine.Object` instead of just `Object` when calling `DestroyImmediate`, `FindObjectOfType`, etc.

---

## 12. Available Tools Quick Reference

**Scene**: `unity_get_scene`, `unity_create_gameobject`, `unity_delete`, `unity_duplicate`
**Components**: `unity_get_components`, `unity_set_property`, `unity_add_component`, `unity_remove_component`
**Assets**: `unity_search_assets`, `unity_get_asset`, `unity_create_asset`, `unity_import_asset`
**Scripts**: `unity_create_script`, `unity_modify_script`
**Editor**: `unity_editor_command`, `unity_get_selection`, `unity_set_selection`, `unity_scene_view`
**Settings**: `unity_get_settings`, `unity_set_settings`
**Build**: `unity_build`, `unity_manage_packages`, `unity_get_console`
**Code**: `unity_execute_csharp`

---

## 13. File Structure

```
Packages/com.claude.unity-mcp/
  package.json                    # UPM package manifest
  mcp-bridge.mjs                  # Node.js stdio-to-HTTP bridge
  Editor/
    MCPServer.cs                  # Main entry point, tool dispatch
    Communication/
      StreamableHttpServer.cs     # HTTP listener on port 9999
      MainThreadDispatcher.cs     # Queue work to EditorApplication.update
      JsonRpcHandler.cs           # JSON-RPC 2.0 response formatting
      MiniJson.cs                 # Lightweight JSON serializer
    Tools/
      SceneTools.cs               # Scene hierarchy operations
      ComponentTools.cs           # Component read/write + runtime data
      AssetTools.cs               # Asset database operations
      ScriptTools.cs              # C# script creation/modification
      EditorTools.cs              # Editor commands, selection, scene view
      SettingsTools.cs            # Project settings read/write
      BuildTools.cs               # Build, packages, console
      ExecuteTools.cs             # Arbitrary C# execution (in-memory)
    Serialization/
      GameObjectSerializer.cs     # Scene hierarchy serialization
      SerializedPropertyHelper.cs # Property value extraction
      AssetSerializer.cs          # Asset metadata serialization
      TypeConverter.cs            # Type parsing utilities
    Utils/
      UndoHelper.cs               # Undo group management
      InstanceTracker.cs          # Resolve objects by ID/name/path
    UI/
      MCPSettingsProvider.cs      # Settings UI in Project Settings
```
