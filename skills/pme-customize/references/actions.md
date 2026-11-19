# Actions: Command, Macro and Modal

Choose the PME type from what the user must be able to do after the action
starts: adjust it, undo it, cancel it, or start it from another mode. Use
[editing](editing.md) for the calls that save slots and [menu design](menu-design.md)
for controls that show state. Check each operator's own requirements in the
user's Blender; an example here is not a list of supported operators.

## Choose from the need

| Need | Use | Check first |
|---|---|---|
| Run one operator and adjust it in Adjust Last Operation | Command slot holding exactly that one `bpy.ops` call | Nothing stands before or after the call. The operator's poll decides when it is available |
| Run several steps, then adjust the main operator in Adjust Last Operation | Macro Operator with one operator per step, linked from a Menu slot | Every operator step's poll passes in the state the Macro starts from |
| Remember the current state, run a native operator, then clean up | Macro Operator: a code step, the operator step, then code steps | The code steps share variables during one run |
| Run several steps from any starting mode; no adjustment afterwards | Macro Operator made of code steps, or one compound Command | Give it a description |
| Move the mouse or scroll to change a value and watch the result before confirming | Modal Operator linked from a Menu slot | On Invoke prepares, On Cancel restores |
| Open a Blender editor in its own window | Command calling `bpy.ops.pme.popup_area(...)` | Per-window setup goes in `cmd` |
| Show whether a setting is on, or switch it | A state control, see [menu design](menu-design.md#show-state-and-choose-an-action) | — |

Switching Blender's mode needs an operator such as `object.mode_set`:
`context.mode` and `object.mode` are read-only, so a Property slot, a PME
Property Setter writing them, or a Property Stack cannot enter a mode.

## How a Command runs

A Command that is exactly one `bpy.ops.<category>.<name>(...)` or
`O.<category>.<name>(...)` call is handed to Blender as that operator. Written
without positional arguments, it runs as `'INVOKE_DEFAULT', True`. Blender
records it, so Adjust Last Operation and Undo follow the operator. Vector Menu
and Floating Panel draw such a slot disabled while the operator's poll fails;
see [Buttons that cannot run now](https://docs.pie-menu-editor.com/reference/command_text.html#code-operator-poll-display).

Any other Command is Python run by PME's command operator: several statements
joined with `;`, a condition, an assignment, or a helper call.

- Operators called inside run as written. `bpy.ops.x.y()` there is
  `'EXEC_DEFAULT'`; PME adds no arguments. Write `'INVOKE_DEFAULT'` yourself
  for an operator whose work happens in its invoke step: one that opens a menu,
  popup or file browser, or starts a modal interaction. `pme.popup_area` is one
  of them.
- In a Vector Menu, an operator called inside is not recorded for Adjust Last
  Operation. The slot's Undo setting and a `True` argument do not change that.
  When the user must adjust the result, keep the slot to the one operator, or
  use a Macro Operator.
- A prefix such as "enter Edit Mode first" turns a single operator into a
  compound Command. Decide which the user needs: the adjustable operator that
  is available only in its mode, or the sequence that starts anywhere.
- The tooltip of a compound Command without a description is the generic
  "Execute python code". Add the description in the same edit.

Classify a saved Command from its text: one statement that is one `bpy.ops` /
`O` operator call is the single-operator form; everything else is compound.

## Macro Operator

A new Macro Operator has one empty Command step. Add steps with `add_slot`,
then fill them with `update_slots`:

```python
created = pme.create_pm(pme.PmCreationSpec("MACRO", name="Lattice Deformer and Parent"))
if not created.accepted:
    raise RuntimeError(created.diagnostics)
macro_uid = created.pm_uid
for _ in range(3):
    version = pme.editing.read_pm(macro_uid)["version"]
    pme.editing.add_slot(macro_uid, expected_version=version)
before = pme.editing.read_pm(macro_uid)
steps = [
    ("Remember Selection", "sel = list(C.selected_objects)"),
    ("Add Lattice", "bpy.ops.object.lattice_add_to_selected()"),
    ("Parent to Lattice", "[o.select_set(True) for o in sel]; bpy.ops.object.parent_set('EXEC_DEFAULT', type='OBJECT', keep_transform=True)"),
    ("Select Lattice Only", "[o.select_set(False) for o in sel]"),
]
edits = {slot["uid"]: {"mode": "COMMAND", "name": name, "cmd": command}
         for slot, (name, command) in zip(before["slots"], steps)}
result = pme.editing.update_slots(macro_uid, edits,
    expected_version=before["version"],
    expected={slot["uid"]: slot for slot in before["slots"]})
print(result["status"], [s["icon"] for s in pme.editing.read_pm(macro_uid)["slots"]])
```

The read-back icons are `TEXT`, `BLENDER`, `TEXT`, `TEXT`: only the second
step is a native step. Link the Macro with
`{"mode": "MENU", "menu_target_uid": macro_uid}`. A direct Menu slot link keeps
the route that Adjust Last Operation uses; calling the Macro from a Command's
code makes it part of a compound Command.

How a step runs is decided from its text, and the saved icon reports it:

| Step | Runs as | Icon after saving |
|---|---|---|
| One operator call, invoke context (the default), keyword values written as literals | A native step of Blender's Macro. Its options appear in Adjust Last Operation | `BLENDER` |
| Several statements, an explicit `'EXEC_DEFAULT'`, or a keyword value that is an expression | Code | `TEXT` |
| Menu slot linking a Macro Operator, Modal Operator or Sticky Key | That operator | — |

The editor owns the icon of Command steps; an icon you set is replaced.

- **Blender checks every native step's poll before the first step runs.** A
  Macro of `object.mode_set(mode='EDIT')` followed by an Edit Mode operator is
  refused from Object Mode, although the first step would have entered Edit
  Mode. Read the poll of each operator you add. When a poll cannot pass at the
  start, choose one: offer the single operator only in its mode; write that
  step as code (`bpy.ops.object.convert('EXEC_DEFAULT', target='MESH')`), which
  is not checked beforehand and is not adjustable afterwards; or use a Modal
  Operator, whose On Invoke is your code.
- **Keep an operator whose invoke opens a menu or popup out of the native
  steps.** `object.parent_set` asks "Set Parent To"; a step that does not
  finish ends the Macro. Call it from a code step with `'EXEC_DEFAULT'` and
  explicit arguments.
- **Write cleanup steps with the data API.** After a native step, Adjust Last
  Operation reruns the Macro with changed options, and those can change the
  mode. `[o.select_set(False) for o in sel]` still works; a `bpy.ops` call
  fails its poll.
- A cleanup step is not `finally`: a cancelled or failed step stops the Macro.

Code steps of one run share variables, as do Sticky Key, Stack Key and Modal
Operator slots. The documentation owns the lifetime rules:

| Type | Section |
|---|---|
| Macro Operator | [Separate save, process, restore into slots](https://docs.pie-menu-editor.com/editors/macro_operator_editor.html#macro-shared-variables) |
| Sticky Key | [Shared variables and restoration](https://docs.pie-menu-editor.com/editors/sticky_key_editor.html#sticky-shared-variables) |
| Stack Key | [Variable sharing and reset conditions](https://docs.pie-menu-editor.com/editors/stack_key_editor.html#stack-shared-variables) |
| Modal Operator | [Share the starting value across slots](https://docs.pie-menu-editor.com/editors/modal_operator_editor.html#modal-shared-variables) |

Use `U` only to pass a value to a later, separate invocation.

## Modal Operator

A new Modal Operator has one On Invoke slot. `update_slots` takes `mode`
(`INVOKE`, `UPDATE`, `FINISH`, `CANCEL`, `COMMAND`, `PROP`), the code in `cmd`
or the property path in `prop`, and for Command and Property slots a
`modal_input` mapping:

```python
pme.editing.update_slots(modal_uid, {
    invoke_uid: {"mode": "INVOKE", "cmd": "view = C.space_data; start = view.lens"},
    lens_uid: {"mode": "PROP", "name": "Lens", "prop": "view.lens",
               "modal_input": {"property_mode": "WHEEL", "step": 1.0}},
    cancel_uid: {"mode": "CANCEL", "cmd": "view.lens = start"},
})
```

- The saved `icon` of a Command or Property slot holds its modal input, for
  example `MOUSEMOVE;0.0;1.0;0.01;`. Read it as `modal_input` with
  `read_slot_inputs`.
- A Property path may start from a variable set in On Invoke, as `view.lens`
  does. To adjust an operator's arguments, keep them in
  `p = C.window_manager.operator_properties_last('<category.name>')`, point
  Property slots at `p.<argument>`, and rerun the operator in an On Update
  slot placed above the inputs. Restore the starting state at the top of that
  On Update so repeated runs do not stack.
- Left mouse button, Enter and Numpad Enter confirm; right mouse button and
  Esc cancel. A Mouse Wheel property works beside a Mouse Move property. While
  the sub-hotkey of a Property slot is held, that slot takes the mouse movement
  instead of the Mouse Move slot. Ctrl makes a float property move in fine
  steps.
- The Modal Operator adds no Undo step when it ends. When the adjustment should
  be one Undo step, add it in On Confirm with
  `bpy.ops.ed.undo_push(message='...')`, and restore in On Cancel.
- A slot that is exactly one `bpy.ops` call receives `'INVOKE_DEFAULT', True`,
  as a Command does. Several statements run as written.

Mouse movement, wheel input and the value display need the user's normal input;
see [verification](verification.md).

## `pme.popup_area`

`bpy.ops.pme.popup_area(area='PROPERTIES', width=440, height=760)` opens that
editor in a new window. Arguments: `area`, `width`, `height`, `center`,
`auto_close`, `header` (`DEFAULT`, `TOP`, `BOTTOM`, `TOP_HIDE`, `BOTTOM_HIDE`)
and `cmd`.

- Every call opens its own window and Screen. `cmd` runs each time, with
  `C.area` and `C.space_data` pointing at the new editor. Put per-window setup
  there: `cmd="C.space_data.context = 'MODIFIER'"`.
- Reach the popup through `cmd`, not by searching Screens for a name. The
  Screen name follows the UI language, and a name search reaches every open
  popup.
- All of its work is in invoke. As the only statement of a Command it is
  invoked; inside a compound Command, write
  `bpy.ops.pme.popup_area('INVOKE_DEFAULT', area=...)`.
- `auto_close=False` keeps the window until the user closes it.
