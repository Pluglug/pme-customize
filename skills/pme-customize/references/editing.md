# Editing recipes

Connect as described in [SKILL.md](../SKILL.md). In these examples `pm_uid` and
`slot_uid` are observed identities chosen for the request, not placeholders to
guess. Use [layout recipes](layouts.md) for structural edits. The
[API reference](https://docs.pie-menu-editor.com/reference/public_api.html#local-menu-editing)
owns complete signatures and results.

## Read and patch one item

Discover once by name, then keep the UID:

```python
menu = pme.find_pm("Mesh Tools")
if menu is None:
    raise ValueError("Mesh Tools was not found")
pm_uid = menu.uid
before = pme.editing.read_pm(pm_uid)
print([(s["uid"], s["index"], s["name"]) for s in before["slots"]])
```

Select the intended `slot_uid` from that read. This changes only its tool command
and Auto Icon, while guarding its observed saved fields:

```python
before = pme.editing.read_pm(pm_uid, slot_uids=[slot_uid])
result = pme.editing.update_slots(
    pm_uid, {slot_uid: {"mode": "COMMAND", "auto_icon": True,
                       "cmd": "bpy.ops.wm.tool_set_by_id(name='builtin.knife')"}},
    expected_version=before["version"],
    expected={slot_uid: before["slots"][0]},
)
print(result)
after = pme.editing.read_pm(pm_uid, slot_uids=[slot_uid])
```

Readback has saved `text`; write through `cmd`, `prop`, or `custom`.
`slot_fields()` describes accepted types/enums. Unspecified fields start from
current values, but normal saves can normalize fields and affect other items:
inspect `slots`, `side_effects`, `structure_changes`, `pending`, and diagnostics.
An EMPTY slot loads as COMMAND; omitting `mode` may turn it into a Command slot.
An empty name may receive a suggested name. Keep required logic even if a literal
tool call would make Auto Icon easier. These calls do not run the command.

## Create a small menu

`create_pm` creates the editor's default shape; it does not accept explicit
`slots` or `data`. This four-direction recipe starts a new VMENU:

```python
created = pme.create_pm(pme.PmCreationSpec("VMENU", name="Mesh Tools"))
if not created.accepted:
    raise RuntimeError(created.diagnostics)
pm_uid = created.pm_uid
before = pme.editing.read_pm(pm_uid)
# Only a fresh VMENU: left=0, right=1, down=2, up=3.
actions = {
    0: ("Select More", "bpy.ops.mesh.select_more()", False),
    1: ("Knife", "bpy.ops.wm.tool_set_by_id(name='builtin.knife')", True),
    2: ("Move", "bpy.ops.wm.tool_set_by_id(name='builtin.move')", True),
    3: ("Select All", "bpy.ops.mesh.select_all(action='SELECT')", False),
}
edits = {before["slots"][i]["uid"]: {"mode": "COMMAND", "name": name,
         "cmd": command, "auto_icon": auto_icon}
         for i, (name, command, auto_icon) in actions.items()}
result = pme.editing.update_slots(pm_uid, edits,
    expected_version=before["version"],
    expected={s["uid"]: s for s in before["slots"] if s["uid"] in edits})
print(result)
```

Use this menu in the intended Mesh Edit context; give it the corresponding
Keymap/Poll using the next recipes. Keep other directions empty unless requested.
For an existing menu, select items by their observed meaning/UID, not this fresh
shape's indexes. If creation is accepted with `refresh_failed`, inspect that UID;
do not recreate it. Inspect metadata and tags after creation rather than assuming
that current UI filters or another menu's tag membership were inherited.
Creation follows New in the editor, so Auto Tag can give the menu the tag that
is selected in the editor's list. Compare the whole `tags` list with the
intended one and remove extras with `update_tags`.

The default shape depends on the type. Read it instead of assuming indexes:

| Type | A new menu contains |
|---|---|
| `PMENU`, `VMENU` | Ten empty fixed slots |
| `RMENU` | One empty item; add more with `add_slot(pm_uid, kind="ITEM")` |
| `FPANEL` | One row: its marker and one default button |
| `MACRO` | One empty Command step |
| `MODAL` | One On Invoke slot |
| `PROPERTY` | A stored Boolean; edit it with `update_property` |
| `CONDITION`, `ROUTER` | Empty logic; edit it with `update_property_stack` / `update_router` |

Macro Operator and Modal Operator slots are covered in [actions](actions.md).

## Metadata, shortcuts and tags

For descriptions or Poll, `update_pm` accepts `name`, `description`,
`description_is_expr`, and `poll_cmd`. Guard the metadata, not just a slot version:

```python
before = pme.editing.read_pm(pm_uid)
metadata = ("name", "description", "description_is_expr", "poll_cmd")
result = pme.editing.update_pm(pm_uid,
    {"description_is_expr": False, "description": "Mesh editing tools.",
     "poll_cmd": "return context.mode == 'EDIT_MESH'"},
    expected={key: before[key] for key in metadata})
print(result)
```

A single-line Poll needs no leading newline/indent. Multiline bodies do.
Invalid Poll may save with `poll_compile_error`; committed is not evaluation proof.
Rename updates normal Menu links and normal registration, but not names embedded
in Command/Custom/Poll source or Router `fallback_name`. An unlocked Property
Stack rename may change its Property ID: read `registered_property_id` again;
widget source containing the old ID is not automatically rewritten.

Choose keys in the user's key configuration; do not silently replace a known
binding. For a requested Mesh Edit shortcut, for example:

```python
before = pme.editing.read_pm(pm_uid)["hotkey"]
changes = {"km_name": "Mesh", "key": "F8", "open_mode": "HOLD"}
checked = pme.editing.update_hotkey(pm_uid, changes, expected=before, dry_run=True)
if checked["status"] == "ready":
    result = pme.editing.update_hotkey(pm_uid, changes, expected=before)
    print(result["after"], result["registration"])
else:
    print(checked)
```

Pass the complete 13-field `hotkey` snapshot as expected. Partial changes retain
other modifiers: explicitly change them if the requested binding needs that.
`proposed`, saved `after`, registration and actual input delivery are separate.
Collision warnings are not permission to overwrite unrelated bindings. See
[menu design](menu-design.md) for Keymap/Router and key-modifier choices.

For tags, use the current API instead of the older JSON workaround:

```python
options = {"add": ["Review"], "remove": ["Draft"]}
checked = pme.editing.update_tags([pm_uid], **options, dry_run=True)
if checked["status"] == "validated":
    result = pme.editing.update_tags([pm_uid], **options, expected=checked["expected"])
    print(result)
print(pme.editing.read_pm(pm_uid)["tags"])
```

`read_pm.tags` reads saved names without synchronizing. Add trims and follows
existing comma semantics; remove uses exact names. Duplicate effective additions
and add/remove overlap reject before writing. Inspect `synchronized` separately
from requested `changed_uids`: normal synchronization can affect other menus.
An inactive registry does not remove legacy name-based tag editing.

## Links, clear, duplicate and delete

- Set a child with `update_slots(..., {slot_uid: {"mode": "MENU",
  "menu_target_uid": child_uid, ...}})`. Observe the child UID first. Use a category
  icon; VMENU supplies the Nest arrow. For an icon-only change, omit mode/target.
- Preserve icon prefixes `#` (icon only), `!` (hidden), `^` (checkbox style) before
  replacing the Blender icon ID or `@filename`. Unknown icon IDs may save with a
  fallback; check the display. RMENU Submenu/Inline and Nest gestures differ.
- `clear_slots(pm_uid, [slot_uid], expected_version=..., expected={slot_uid: saved})`
  retains UID/position. Do not write `mode="EMPTY"`. Clear rejects structural
  items and needs an editor supporting EMPTY; a cleared RMENU item is a separator.
  Use `remove_slot` when removal is intended.
- `duplicate_pm(source_uid, name="Working copy", dry_run=True)` returns an
  `expected` to pass to the same operation. Inspect `created` for the actual new
  UID/name/index. The copy is directly after its source, receives new PM/item
  UIDs, and copies tags and normal item content. Menu keys/modifiers are not inherited and
  the copy starts enabled. Public duplication retains selection/drafts; UI selects
  the copy. Do not rename as a second step to supply a creation name.
  To split one page into two (for example an Object Mode page and an Edit Mode
  page), duplicate it, replace only the slots that differ, and give each copy
  its own Router branch.
- Read a slot that serves as a template immediately before writing from it.
  The user may have rearranged the menu since the last read; an empty
  `read_slot_inputs` `edit` (`None`) means there is nothing to copy.
- `delete_pms([pm_uid], dry_run=True)` exposes saved references and diagnostics.
  Inspect these before applying with the complete returned `expected`. Surviving
  references remain and may be broken. A target with an open draft rejects;
  unrelated drafts are retained. Saved-data recovery is not host recovery.
- `set_enabled(pm_uids, enabled, expected=..., dry_run=True)` edits menu enabled
  using its own snapshot contract. It is not a slot `update_slots` field.
- `find_references(pm_uids)` lists the saved slots, Router targets/conditions and
  Property Stack sources that refer to the menus, without preparing a deletion.
  Check `unscannable` too before calling a menu unlinked.
- `read_slot_inputs(pm_uid)["slots"][i]["edit"]` is the slot under `update_slots`
  input names (`cmd` / `prop` / `custom` / `menu_target_uid`, `expand_menu`,
  `use_frame`); pass it back unchanged or to another slot. It is `None` for
  layout entries and empty slots. `read_pm` keeps returning saved fields.
  `update_slots` raises `ValueError` for `any=True` (no slot saves it; set it
  to `False`) and for `use_frame=True` without `expand_menu=True`.
- A `committed` slot entry can carry the diagnostic `slot_value_not_retained`
  (`field`, `requested`, `actual`): the slot was saved but that input did not
  stay, for example `expand_menu=True` on a link to a Regular Menu. Read the
  `diagnostics` of every committed slot.
- `update_slots`, `update_pm`, `update_router` and `update_property_stack`
  return `status == "unchanged"` when the request equals what is saved. Treat
  it as "nothing to do", not as a failure; only `committed` means a write.
- `resolve_auto_icons(pm_uid)` reports per slot whether Auto Icon finds an icon
  (`resolved`) and what the command names (`source`).
- `reorder_router_branches(pm_uid, branch_uids, expected_digest=...)` puts every
  Router branch in the given order with one save; list each branch UID once.
  `move_router_branch(pm_uid, branch_uid, after_uid=...)` moves one branch.
  `update_router` is a patch: an entry with `branch_uid` updates that branch,
  an entry without one appends a branch, and branches not named stay.
- A link from a menu slot to a PME Property or a Property Stack is a Menu slot
  (`"mode": "MENU", "menu_target_uid": uid`), not a Property slot.
- `add_row()` returns the new row marker and button in `created_uids`;
  `add_slot()` returns the new item in `slots[0]["uid"]`.

None of these operations saves Preferences to disk. Use each operation's result
contract, not one shared "success" predicate or generic retry wrapper.

## When to use JSON or the normal UI

| Need | Current route |
|---|---|
| Ordinary item/menu/Router/Stack/layout/metadata edits | Public `editing` operations; do not regenerate a whole menu JSON for one change |
| Transfer presets between installations, selected-menu Import/Export, or a deliberate bulk replacement | Normal JSON workflow; `pme.export_menus(pm_uids, filepath=...)` writes the same document as Export Menus without touching the selection. Validate against the installed schema and inspect the chosen import mode/UID/reference effects |
| A menu type's own settings (Popup Dialog Mode/width, Pie radius, Panel Group placement, extend target, Wait for Pointer Press / start inputs, ...) | `read_pm()["settings"]`, `setting_fields(pm_uid)` and `update_settings(pm_uid, changes, expected=..., dry_run=...)`. They are not `update_pm` or `update_hotkey` inputs, and `create_pm` does not take them |
| PROPERTY definitions (type, arguments, Enum items, Getter/Setter/Update/Init, storage, Property ID) | `read_pm()["property"]` and `update_property(pm_uid, changes, expected_digest=..., dry_run=...)`; create the default Boolean with `create_pm` first. Callbacks are not executed by the edit |
| Multiple independent shortcuts for one menu | One direct saved binding is available, possibly in multiple Keymaps. JSON does not add independent bindings. Reuse the menu through another real entry where needed |

Prefer the normal UI for a small unsupported-field change. When JSON fits the
request, start from the current selected definitions, retain required identities
and references, validate, import with explicit intended semantics, then read back
both targets and unrelated definitions. Do not import a stale whole-preferences
snapshot over the user's newer work. A validating JSON or successful Import does
not prove runtime behavior. Never bundle personal snapshots with a shared skill.
