# Mutable Graph Toolset: guide for AI agents

You are an AI agent working in a user's Unreal Engine 5.8 project through the editor's MCP server.
The toolset `MutableGraphToolset.MutableGraphToolset` lets you build and edit Mutable Customizable
Objects. This guide tells you how to set it up, how to work with it and which mistakes to avoid.
Read it completely before your first Mutable task.

## 1. Setup checklist

Do these in order and stop at the first step that fails; tell the user what is missing.

1. Locate the `.uproject`. Its `EngineAssociation` must be `5.8`.
2. Check that the plugin is installed: `Engine/Plugins/Marketplace/MutableGraphToolset` (from Fab)
   or `<Project>/Plugins/MutableGraphToolset`. Do not download it from anywhere else.
3. In the `.uproject`, make sure the `Plugins` array enables `MutableGraphToolset`, `Mutable`,
   `ToolsetRegistry` and `ModelContextProtocol`. Add missing entries as
   `{ "Name": "<Plugin>", "Enabled": true }`. Change nothing else in the file.
4. The editor must run with its MCP server: launched with `-ModelContextProtocolStartServer`, or
   with auto start enabled in Project Settings > Plugins > Model Context Protocol. If the editor is
   already open without the server, ask the user before restarting it; unsaved work could be lost.
5. Register `http://127.0.0.1:8000/mcp` (or the configured port) in your own MCP client
   configuration as an HTTP server, and reload your MCP servers if needed.
6. Call `list_toolsets`; `MutableGraphToolset.MutableGraphToolset` must be listed. Call
   `describe_toolset` on it and keep the schemas at hand.

## 2. Calling tools

- Everything goes through `call_tool` with `toolset_name`, `tool_name` and `arguments`.
- Argument names are lower camel case: `packagePath`, `fromNode`, `bIncludeHiddenPins`.
- Assets, nodes and classes are `{ "refPath": "..." }`. Use the exact `refPath` values the tools
  return; never build node paths yourself.
- Positions are `{ "x": ..., "y": ... }`. Space nodes about 300 units apart, inputs to the left.
- Errors start with `MutableGraphToolset:` and say what is wrong. Read them and correct the call;
  do not retry the same call unchanged.

## 3. Rules

1. **Inspect before you edit.** Call `ListNodes` (or `GetNode`) to learn real pin names. Pin names
   are not always the labels the editor shows, for example `Mesh_Input_Pin`.
2. **Find node classes, do not guess them.** Use `ListNodeClasses` with a filter such as
   `"Material"`, `"Group"`, `"Texture"` or `"Parameter"`.
3. **Refresh after pin-changing edits.** After changing a property that adds or removes pins (a
   mesh, a material, a table), call `RefreshNode`, then read the pins again.
4. **Compile after every logical change** with `CompileObject` and read `messages`. A call that
   succeeds can still have compile errors; `bCompiled` and `bHasErrors` tell you.
5. **Parameters and previews need a compiled object.** Compile again after graph changes before
   `ListParameters`, `GetInstanceParameters`, `SetInstanceParameter` or a capture.
6. **Save child objects before compiling their parent.** After `SetParentObject`, call `SaveAsset`
   on the child; Mutable only finds children through saved references.
7. **Name child options with a Static String node.** In UE 5.8 an object's name comes from the
   `Name` pin of its base object node: add `CustomizableObjectNodeStaticString`, set its `Value`
   with `SetNodeProperty` and connect it to that pin.
8. **Verify visually.** After a meaningful change, render with `CaptureObjectPreview` or
   `CaptureInstancePreview` and look at the image before you report success.
9. **Use project content only.** Create assets under `/Game/...` (or a project plugin). Engine
   content is refused on purpose; do not try to work around it.
10. **Do not delete or overwrite the user's work without asking.** `RemoveNode` on nodes you did
    not create, replacing existing assets or restarting the editor needs the user's consent.
11. **Save deliberately.** Edits mark assets dirty but do not save them. Use `SaveAsset` when a
    step is complete and the user wants it kept.
12. **Report honestly.** Tell the user which compile messages remain and what you could not do.

## 4. Typical workflows

### Build a renderable base object

1. `CreateObject` → keep the object ref.
2. `ListNodeClasses "SkeletalMesh"`; `AddNode` for: `CONodeComponentSkeletalMesh`,
   `CustomizableObjectNodeSkeletalMesh`, `CONodeSkeletalMeshSection`, `CONodeSkeletalMeshMake_V2`,
   `CONodeSkeletalMeshObjectMake`.
3. Set the skeletal mesh on the *Skeletal Mesh* node and the reference mesh on the component node
   (`SetNodeProperty` or `ObjectTools`), then `RefreshNode` the mesh node.
4. `ListNodes`, then connect: Component.`Component` → Base Object.`Components`;
   Skeletal Mesh.`LOD 0 - Section 0 - Mesh` → Mesh Section.`Mesh_Input_Pin`;
   Mesh Section.`Mesh Section_Output_Pin` → Skeletal Mesh Make.`LOD 0`;
   Skeletal Mesh Make.`Skeletal Mesh` → Skeletal Mesh Object Make.`Skeletal Mesh`;
   Skeletal Mesh Object Make.`Passthrough Skeletal Mesh` → Component.`Passthrough Skeletal Mesh`.
5. `CompileObject`, fix `messages`, `CaptureObjectPreview`.

### Add selectable options (clothing, hair, body parts)

1. In the parent: `AddNode CustomizableObjectNodeObjectGroup`; connect its `Group` output to the
   base object's `Children`; set `GroupName` and `GroupType` (`COGT_ONE`, `COGT_ONE_OR_NONE`,
   `COGT_ALL`, `COGT_TOGGLE`) with `SetNodeProperty`.
2. For each option: `CreateObject`, build its mesh graph, name it with a Static String node on the
   `Name` pin, `SetParentObject` with the group node, `SaveAsset`.
3. `CompileObject` the parent; `ListParameters` now shows the group as an `Int` parameter whose
   `options` are the child names (plus `None` for one-or-none groups).

### Configure and check a variant

1. `CreateInstance` of the compiled object.
2. `ListParameters` to see names, types and options; `SetInstanceParameter` per value, in the
   formats below.
3. `GetInstanceParameters` to confirm; `CaptureInstancePreview` to look; `SaveAsset` to keep.

### Copy a setup

`ExportNodes` the nodes you need, then `ImportNodes` into the target object at a free position.
Connections to nodes outside the copied set are dropped; reconnect them with `ConnectPins`.

## 5. Value formats

| Use | Format |
| --- | --- |
| Int parameter | Option name from `options` |
| Bool parameter | `true` / `false` |
| Float parameter | `0.5` |
| Color parameter | `(R=1,G=0.5,B=0,A=1)` |
| Transform parameter | `X,Y,Z\|Pitch,Yaw,Roll\|SX,SY,SZ` |
| Texture, SkeletalMesh, Material parameter | `/Game/Path/Asset.Asset` |
| Node properties (`SetNodeProperty`) | Unreal text format; read the current value with `GetNodeProperty` first and keep its shape |
| Enum node properties | The enum value name, for example `COGT_ONE_OR_NONE` |
| Object reference properties | `/Script/Engine.SkeletalMesh'/Game/Path/Asset.Asset'` or the plain object path |

## 6. When something fails

| Error contains | Do this |
| --- | --- |
| `has no pin` | Use one of the pins listed in the message. |
| `Cannot connect` | Wrong direction or category; compare `category` and `bIsOutput` in `ListNodes`. |
| `is not compiled` | `CompileObject` first. |
| `is not an option` | Use a value from `options`; save the children and recompile if one is missing. |
| `outside the project's content` | Use a `/Game/...` path. |
| `already exists` | Pick another name, or ask the user whether to reuse the existing asset. |
| `preview captures are already running` | Wait for the earlier capture result, then try again. |
| `internal graph state` / `transient or deprecated` | That property is off limits; find the supported way (a pin, another node). |

## 7. Limits

UE 5.8 on Windows; editor only; graphs inside macro libraries and multidimensional parameter
values are not editable in this version. If the user asks for one of these, say so instead of
improvising with other tools.
