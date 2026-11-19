# Verification and limits

Verify the requested user result at the relevant boundary. A small label edit
does not need an unrelated full regression exercise. Input changes need normal
input; state controls need observed values; persistence changes need save/reopen.
Use only the intended session and the user's authorized scope.

## A minimal check for a menu edit

1. Retain the relevant before definition, UID, version/digest and intended patch.
   Read only the affected menu/settings; do not publish a whole private snapshot.
2. Inspect the operation result and diagnostics, then read the affected item or
   settings again. Check intended values and relevant side effects. For a batch,
   distinguish committed, rejected and pending items.
3. Open the menu through the requested normal shortcut in its intended area/mode.
   Check placement and visible state, choose the changed item, and observe the
   requested result. If conditions changed, also check a nonmatching context and
   Pass Through. A registration report or synthetic invoke is not this evidence.
4. If the action changes scene data, complete it and check relevant F9/Adjust Last
   Operation, Undo and Redo. Tool activation needs the tool to become active;
   a geometry result needs the geometry result. Check a property control's actual
   values and checked state. For VMENU Custom, test click and direction release
   separately; displaying one or several widgets does not prove either action.
5. Save Preferences when authorized/needed, and verify restart persistence when
   the request requires it. API edits do not save to disk. Avoid restarting the
   user's active work merely to repeat unrelated established evidence.

For example, after changing the right Mesh Tools direction to Knife: inspect the
saved command; hold its assigned key in Mesh Edit Mode; observe the right item;
release toward it and confirm Knife is active. Save/reopen only as required by
the task. This tests that specific path, not every menu action or Blender mode.

If an available tool cannot send a particular hold/move/release or rapid key
sequence, finish agent-side preparation and batch only those remaining checks
for the user. Give the exact window/context, input, expected visible result and
end state. Record the returned result's scope; "opened and Esc closed it" is not
evidence that its commands, Undo or restart persistence work.

## Try commands outside the user's session

Keep test data out of the user's Blender. To learn what an operator or a
Command does, start a separate Blender with factory settings and its own
profile folder, run the code there, and read the result. Slot code that uses
only `bpy` and `C` needs no PME for this: execute it with
`exec(code, {"bpy": bpy, "C": bpy.context})` inside a
`bpy.context.temp_override(window=..., area=..., region=...)` from a timer. For
Macro or Modal slots that share variables, run the slot bodies in order in one
dictionary. Many commands fit one run.

- Start from uneven states as well as the clean one: an object that already has
  a parent, a hidden object, Edit Mode, a selection of one element. An operator
  that narrows the current selection or keeps an existing parent behaves
  differently there.
- When a stand-in operator or script replaces a PME part, call it the way PME
  does, including positional arguments such as the Undo flag. A stand-in called
  differently can report the opposite result.
- Read an unfamiliar operator's poll and invoke in Blender's source or `bl_ui`
  before placing it. Operators with the same name in two categories can differ
  in what they select or reset.
- Check that an operator is Blender's own before using it in a preset meant for
  other users: `bpy.types.<CATEGORY>_OT_<name>.__module__` is `None` for a
  built-in operator, `bl_operators.*` or `bl_ui.*` for bundled Python, and the
  add-on's module for an add-on operator.
- This run proves the code path only. Gesture release, a Modal's mouse and
  wheel input, on-screen values and Adjust Last Operation still need normal
  input in the user's session.

Before asking the user to check something in their Blender, confirm what that
Blender is running: the PME build it loaded, and whether the edits are in
memory only. A check against an older build reports a problem that is already
fixed. After a restart without Save Preferences, read which menus exist before
reapplying; write the applying script so that running it again is harmless.

## Reconcile before retrying

| Observation | Next action |
|---|---|
| Stale expected/version/digest | Read again and reconcile the intended change with human edits; do not replace their values just to make the call pass |
| Partial slot batch | Read every affected/pending item and reported side effects; resume only the changes still needed |
| Accepted creation with refresh failure | Inspect the returned menu UID and diagnostics; do not create a second menu |
| `unknown`, exception during save, unavailable after-read, or transport timeout | Read actual state before deciding whether to retry; timeout is not cancellation |
| Dry-run validated/ready | Apply using that operation's contract, then inspect the real result; no execution/rendering proof yet |

Slot updates save sequentially and stop on rejection; they are not atomic.
`pending` means its explicit save was not attempted, not necessarily unchanged:
earlier callbacks may change it. A `TypeError`/`ValueError` during preflight differs
from the same type inside a saving callback; inspect notes/diagnostics and state.
Small batches with recorded outcomes are easier to reconcile than a timed-out
whole-catalog rewrite. Do not replace saved-value guards with fixed sleep/retry.

## Code fields and Undo

Command executes statements; Poll and Getter return values; Custom draws UI
repeatedly. Use their actual execution forms and exposed globals. Avoid accidental
leading whitespace in a one-line body while retaining needed multiline indentation.
Do not blanket-strip or dedent saved scripts to repair one diagnostic. Import or
syntax validation proves neither actual poll nor interaction success.

There are two different edits to verify: editing a **menu definition**, and running
its command to edit **Blender data**. The editing API supplies no general Undo/Redo
history for menu definitions. A menu's `exec_undo` setting and a Python operator
call's Undo argument concern command execution; neither is proof of menu-edit Undo.

Builds with the command Undo fix forward the resolved Undo flag through the
VMENU COMMAND fallback and Context Router COMMAND paths. Do not add
`bpy.ops.ed.undo_push()` to every script as a workaround, or automatically remove
it from an existing compound action.
Verify the user's actual action sequence, including whether one Undo restores
the intended state and Redo reapplies it. Design multi-step behavior deliberately.
Which form keeps Adjust Last Operation, and where a Modal Operator needs its own
Undo step, is in [actions](actions.md).

## Offer a report when PME is at fault

Offer this when the installed PME did not do what its API reference or
documentation says, raised a traceback from its own code, or made you work
around a missing capability. A mistake in the requested design or in a script
you wrote is fixed in that script instead.

1. Tell the user what happened and what you did about it, and ask whether they
   want to report it to PME support. Continue the requested work either way.
2. With their agreement, draft the report for them: a short summary, the steps
   or the smallest call that shows the problem, the exact error text, and the
   Blender version, PME version and OS. The
   [support page](https://docs.pie-menu-editor.com/support_community/get_support.html)
   lists what helps.
3. Keep the example to the smallest menu that shows the problem. Attach a menu
   export or a local path only when the user says to include it.
4. The user sends the report: by email to `support@pie-menu-editor.com`, or
   with the Report a Bug button on the support page.

## Current boundaries and evidence

Use the installed API contract and actual capabilities. A tested Blender version
records evidence, not a feature or input restriction. Ordinary tags, shortcuts,
Router conditions, Stack logic/rows and Popup layout now have public editing
operations; older JSON workarounds for them are not current limitations.
Vector Menu start inputs (`update_settings`), PROPERTY callbacks
(`update_property`) and the Stack While True HUD (`update_property_stack_hud`)
have public operations too. See [editing](editing.md) and [menu design](menu-design.md).

Evidence reused for these recipes is deliberately scoped:

- The original API customization trial created the four-direction VMENU, changed
  Knife, and created/edited Router conditions through public calls. User normal
  input confirmed hold/release to Knife and Move, and a separate Object menu's
  display/Esc. That does not validate every new composition in this edition.
- A later customization trial confirmed Components, UV Tools and Sculpt Options
  opening through their requested normal shortcuts and closing with Esc. Item
  actions, all condition combinations and restart were not part of that report.
- The maintainer checked the command Undo fix in Blender 5.2 with the packaged
  extension. The check covered command Undo. Individual before/after, Undo-OFF
  and Redo outcomes were not all reported.
- This edition reorganizes recipes and checks them against current contracts.
  Newly composed snippets are not claimed as new foreground runs. In particular,
  the UV Sync Router example and the full Stack authoring example still need the
  requested user's end-to-end check when used. Prior Stack widget/preset evidence
  alone does not prove its current API authoring route.

- The [actions](actions.md) guidance and the state-control, narrow-panel and
  background-color notes come from one mode-switch preset built through the
  public API on Blender 5.2. User normal input confirmed a Modal Operator with
  Mouse Move and Mouse Wheel properties, `depress` operator buttons on Vector
  Menu directions, a PME Property Setter reading `E`, and a Property Stack with
  its While True HUD. The Macro example's steps were run in order in a separate
  Blender; its Adjust Last Operation behavior from a menu was not reported.
  The draw-only-when-available Custom slot and the regrouped panel rows were
  saved and read back; their display checks were not reported.

Report the edit, actual readback and checks performed, then the specific remaining
route. Keep source/runtime problems visible; absence of an exception or
`invoke_pm() == True` does not prove a usable menu.
