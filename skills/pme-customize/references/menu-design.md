# Menu design

Choose the user's next Blender action first. A faithful Maya preset is a separate
request: preserve its requested directions/order and identify adaptations or
unmatched functions. For new Blender workflows, start with Blender's modes,
tools and existing shortcuts instead of inheriting that reference's whole tree.

## Choose the entry and conditions

| Decision | Use |
|---|---|
| Different editor or mode already has a Keymap | The corresponding Keymap; avoid duplicating that distinction in every branch |
| The whole menu needs one eligibility check | Menu Poll |
| The same entry needs different targets for selection, tool or state | Router with individually editable Conditions |
| Several real entries share useful content | Reuse that menu as their target; do not duplicate content just to obtain another entry |

Do not insert a one-to-one Router or speculative empty branches. A direct menu
has one saved shortcut binding, possibly registered in several Keymaps. Those
Keymaps are not independent key combinations. Key Mod does not consume the
modifier key's own default action; check real input and active-tool priority.
Addon/user Keymap rows for one binding do not by themselves mean duplicate keys.

Use common Poll to narrow the entry. Put branch-specific facts in individual
Conditions, with an explicit AND/OR choice. Use Getter code only for computation
that ordinary conditions cannot express, not as a container for an entire Router.
For example, an active Mesh is not necessarily selected: `select_get()` requires
a suitable existing Property Getter rather than a function call disguised as an
RNA path. Callback authoring uses the UI/JSON route in [editing](editing.md).

## Create and edit a Router

For a UV Editor entry whose common Poll requires Edit Mesh, this branch selects
an observed `target_uid` when UV Sync Selection is on. The unmatched input passes
through. Connect the new Router's requested shortcut/Poll using [editing](editing.md).
This snippet only creates its definition, not that input registration.

```python
created = pme.create_pm(pme.PmCreationSpec("ROUTER", name="UV Selection Router"))
if not created.accepted:
    raise RuntimeError(created.diagnostics)
router_uid = created.pm_uid
condition = {
    "source": {"kind": "rna_path", "root": "context", "detected_type": "bool",
               "readonly": True, "segments": [{"kind": "attr", "name": part}
               for part in ("scene", "tool_settings", "use_uv_select_sync")]},
    "op": "is_true", "value": None, "value_type": "bool",
}
branches = [{"name": "UV Sync On", "combine_mode": "all", "conditions": [condition],
             "target": {"action": {"type": "menu", "value": target_uid, "opts": {}},
                        "fallback_name": ""}}]
router = pme.editing.read_pm(router_uid)["router"]
checked = pme.editing.update_router(router_uid, branches=branches,
    otherwise={"pass": True}, expected_digest=router["digest"], dry_run=True)
if checked["status"] == "validated":
    result = pme.editing.update_router(router_uid, branches=branches,
        otherwise={"pass": True}, expected_digest=router["digest"])
    print(result["status"], result["diagnostics"])
else:
    print(checked)
```

Repeat the original inputs/digest to commit: Router preview's candidate branch
UIDs are provisional, not existing branches to resubmit. A preview does not
evaluate conditions. Unresolved-target warnings can coexist with a commit.
The first matching branch wins; an empty ordinary branch is not unconditional.
`otherwise={"pass": True}` is Pass Through, not a target menu.

To change one condition, select observed branch/condition UIDs after reading the
current names/sources. Copy the complete condition list to retain other rows:

```python
from copy import deepcopy
router = pme.editing.read_pm(router_uid)["router"]
branch = next(b for b in router["logic"]["branches"] if b["branch_uid"] == branch_uid)
conditions = deepcopy(branch["conditions"])
next(c for c in conditions if c["condition_uid"] == condition_uid)["op"] = "is_false"
result = pme.editing.update_router(router_uid,
    branches=[{"branch_uid": branch_uid, "conditions": conditions}],
    expected_digest=router["digest"])
print(result)
```

The example changes the Bool test above. `conditions` replaces that branch's
whole list; omitted branch fields/branches and Otherwise remain. Omit a UID only
for a new item. Use `move_router_branch` / `remove_router_branch` for ordering or
deletion, with the freshly read digest. Test matches, nonmatches and the
underlying shortcut's action, not just the Router's displayed name.

## Show state and choose an action

Use a visible property control for an ordinary switch. A single Custom direction
can draw, for example, `L.prop(C.scene.tool_settings, 'use_snap', text='Snap', toggle=True)`.
Store that as Custom code through `update_slots`; `L` and `C` belong to the stored
draw context. Custom redraws repeatedly: do not run an action unconditionally in
its draw body. One control per direction makes the intended action clear; put
extra controls in extras/children. Verify click and direction release separately.
Extra slots 9/10 use pointer controls, not directional selection.

Choose the control from how Blender itself shows the setting:

| The setting | Use |
|---|---|
| An ordinary Bool with a property path | Property slot with that path, or the Custom `L.prop(...)` above |
| Drawn by Blender's own header or panel as `operator(..., depress=...)`, such as the Mesh select mode buttons | A Custom slot with the same single line: `L.operator('mesh.select_mode', text='Vertex', icon='VERTEXSEL', depress=C.scene.tool_settings.mesh_select_mode[0]).type = 'VERT'`. The operator keeps its own Shift/Ctrl behavior. Read `bl_ui` to see how Blender draws it |
| One value of an Enum per direction | A Custom slot whose body is the single line `L.prop_enum(data, 'property', 'VALUE')`. Wrapped in a column or drawn several times, it stays an ordinary layout button |
| No operator and no single path: one element of an array, a computed state, a two-value Enum shown as a switch | A PME Property with Getter and Setter, linked from a Menu slot. See the [Property switch recipe](https://docs.pie-menu-editor.com/reference/authoring/recipes/property_switch.html) |
| Several settings that form one state | A Property Stack, below |

- A Vector Menu direction shows on/off state for exactly one directly drawn
  control: a Bool property, a `prop_enum` choice, or an operator button. Put
  further controls in extras or children.
- A PME Property's Setter can read the input that wrote the value as `E` when a
  Vector Menu or Floating Panel writes it. Other writers give no input, so read
  it with `globals().get('E')` and keep a plain branch. When Blender has an
  operator for the setting, prefer the `depress` form over modifier logic in a
  Setter.
- A Property slot path cannot index one element of an array; that case is the
  PME Property row. A Property Stack row for an Enum value writes it on TRUE
  only. For a two-way switch over a two-value Enum use a PME Property, or one
  Stack per value when no Python is wanted.
- Linked exclusive flags are one setting. Write one of them and read how the
  others change before composing a Command that writes several.

For continuing work such as Knife or Move, tool activation may fit better than
one immediate operator call. Use Auto Icon where the tool can be recognized;
do not drop necessary command logic just to obtain an icon. For an interactive
operator, decide Invoke versus Execute from its actual behavior. Invoke alone
does not prove F9/Adjust Last Operation or Undo works. A Macro may poll children
before running them, so mode-switch plus edit-only child is not a general recipe.
[Actions](actions.md) chooses between a single operator, a compound Command, a
Macro Operator and a Modal Operator. Use [verification](verification.md) for
action/Undo checks.

## Unavailable items: hide, gray out, or explain

| Wanted behavior | Use |
|---|---|
| The item is absent while it cannot be used | A Custom slot that draws only when the condition holds: `if C.active_object and C.active_object.particle_systems: L.operator('object.mode_set', text='Particle Edit', icon='PARTICLEMODE').mode = 'PARTICLE_EDIT'`. A one-statement `if` fits one line. Check both states through normal input |
| The item stays in place, grayed out | A Command that is one operator call: Vector Menu and Floating Panel follow the operator's poll. For a child menu, the child's Poll |
| The user should learn why | A Command that shows a message; keep it for placeholders and rare cases, not routine switching |

A slot disabled in the editor is not drawn; that is the author's switch, not a
runtime state. A Vector Menu grays out a Nest whose child Poll is false on
every level: on the opened menu when it opens, and on a deeper level when that
level is shown. The gray Nest keeps its place and cannot be entered by click,
dwell or gesture. Check the gray state through normal input in each mode.

## Icons, descriptions and names

Choose useful category icons for Nest; VMENU already draws its arrow. Let a single
operator use its built-in tooltip. Give a menu, Nest or compound action a short
description of its result. Keep a requested but unmatched item as `TODO: <function>`
with a message explaining the missing function. A message box is useful for that
placeholder, not for routine state switching. Preserve the user's chosen labels.

- A compound Command without a description shows the generic "Execute python
  code" tooltip. This covers a mode switch joined to an operator, a condition,
  a bare assignment, and a call to a PME helper. Add the description in the
  edit that writes the slot.
- A description says what the slot or menu does, for the person who receives
  the preset. Leave out directions ("Left: ...", which go stale on every
  rearrangement), issue numbers, workarounds, and reasons for an empty slot.
- Name a group with Blender's own category name, and keep one category in one
  Nest. An invented group name hides which Blender item a label means.

## Placement heuristics (candidates)

These come from one mode-switch preset. Treat them as starting points; the
user's own layout and stated preference decide.

- Nests aligned in one direction are a single stroke: releasing after a
  continued flick runs the item at the end. A catch-all list at the end of a
  repeated direction is easy to reach. A direction entered twice in a row
  should hold an action that is harmless when triggered by mistake.
- Read the default keymap before spending a direction. An action with a near
  default key, or one a modifier key already gives while working, needs a
  direction less than an action with no key.
- Leave some directions empty for the user's later additions.

## Keep a combined setting in one direction

Use a Property Stack (`CONDITION`) when several values form one state/control.
For an existing Stack, prefer `add_property_stack_row`, `update_property_stack_row`
or `remove_property_stack_row` for a single condition. Read its digest, use the
operation's preview, and preserve other rows and writes. The owner maintains
derived SET entries. A new empty row is not yet an executable preset.

For a new Surface preset, the example below sets Connected on and Projected off.
Its fixed writes are `manual_resolved`: TRUE applies the preset; FALSE does
nothing. This does not enable proportional editing itself.

```python
created = pme.create_pm(pme.PmCreationSpec("CONDITION", name="Surface"))
if not created.accepted:
    raise RuntimeError(created.diagnostics)
stack_uid = created.pm_uid
before = pme.editing.read_pm(stack_uid)["property_stack"]
def fixed_bool(uid, property_name, value):
    source = {"kind": "rna_path", "root": "context", "detected_type": "bool",
              "readonly": False, "segments": [{"kind": "attr", "name": part}
              for part in ("scene", "tool_settings", property_name)]}
    row = {"condition_uid": uid, "enabled": True, "authoring_state": "complete",
           "label": property_name, "source": source,
           "op": "is_true" if value else "is_false", "value": None, "value_type": "bool"}
    entry = {"entry_uid": "set_" + uid, "kind": "derived", "status": "manual_resolved",
             "derived_from_condition_uid": uid, "source": source, "value": value,
             "value_type": "bool", "custom_set_body": "", "enum_set_write_mode": None,
             "enum_set_write_values": []}
    return row, entry
pairs = [fixed_bool("connected", "use_proportional_connected", True),
         fixed_bool("projected", "use_proportional_projected", False)]
logic = {"version": 1, "combine_mode": "all", "predicate_rows": [r for r, e in pairs],
         "write_plan": {"false_policy": "no_op", "entries": [e for r, e in pairs]}}
check = pme.editing.update_property_stack(stack_uid, logic=logic,
    expected_digest=before["digest"], dry_run=True)
if check["status"] != "validated":
    raise ValueError(check["diagnostics"])
result = pme.editing.update_property_stack(stack_uid, logic=check["candidate"],
    expected_digest=before["digest"])
if result["status"] != "committed":
    raise RuntimeError(result["diagnostics"])
stack = pme.editing.read_pm(stack_uid)["property_stack"]
property_id = stack["registered_property_id"]
if not property_id:
    raise RuntimeError(f"Property Stack is not registered: {stack['diagnostics']}")
before = pme.editing.read_pm(pm_uid, slot_uids=[slot_uid])
result = pme.editing.update_slots(pm_uid, {slot_uid: {"mode": "CUSTOM",
    "custom": f"L.prop(props(), {property_id!r}, text='Surface', toggle=True)"}},
    expected_version=before["version"], expected={slot_uid: before["slots"][0]})
print(result)
```

Use observed destination `pm_uid` / `slot_uid` for the final widget. The names
inside the helper are local identities for this new Stack; do not rebuild an
existing Stack with it. Inspect `scope="current_storage"` diagnostics before
copying readback's normalized `logic`: it may omit entries or disable writes.
Preserve condition/entry UIDs and derived relationships in a full logic update.
Submitted semantic/identity repairs reject; harmless canonicalization is reported.
Stack preview's validated `candidate` may be submitted as above, unlike Router's
provisional branch identities. Neither preview proves Getter/Setter execution or
actual writability. Read actual values and diagnostics after applying.

Use the returned `registered_property_id`; do not derive it from the name. The
While True HUD is separate from logic and retained by logic updates; set it with
`update_property_stack_hud(uid, {"enabled": True, "display": "FRAME", ...})`,
giving `display` and `alignment` together (an alignment the display lacks is
rejected, not moved). A Stack rename may change its Property ID; a widget's stored
source containing the old ID is not rewritten automatically.

A row's `source` can address one element of an array: end `segments` with
`{"kind": "index", "index": 0}`, as in `tool_settings.mesh_select_mode[0]`.
A row that only tests a read-only value such as `context.mode` sets
`"readonly": True` on its source, and its write entry has `"status": "off"` and
`"value": None`. From a Vector Menu, link the
Stack with a Menu slot (`menu_target_uid` = the Stack's UID) or draw its
registered ID as above.

Choose FALSE behavior explicitly. `auto_resolved` Bool writes can invert on FALSE;
fixed manual writes and enum-set Replace do not mean restoring previous values.
Check all involved values, unrelated settings, checked state and direction release.
A smaller existing-row edit should not replace an entire logic definition.
