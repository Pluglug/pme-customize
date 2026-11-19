---
name: pme-customize
description: Create and refine Blender workflows with Pie Menu Editor menus, shortcuts, conditions, and controls through the public editing API. Use for user customization, not add-on implementation.
---

# Customize Pie Menu Editor

Build around the user's Blender modes, tools, and next action. Prefer the public
editing API for ordinary menu edits; use the normal UI or targeted JSON
Import/Export when the task needs them. An example is not a feature restriction.

## Connect to the intended session

Use the user's intended Blender instance on its main Python thread. Authoring
needs an initialized GUI session with PME Preferences available. Do not reload
scripts, change boot settings, or modify another session just to discover the API.
Use an available connector or the Console/Text Editor; this skill does not require
one particular tool. If no executable connection is available, return a scoped
script and say that it has not run.

Use `import pme` if the optional alias is already enabled; otherwise:

```python
import bpy, importlib
packages = [a.module for a in bpy.context.preferences.addons
            if a.module.rsplit(".", 1)[-1].replace("-", "_") == "pie_menu_editor"]
if len(packages) != 1:
    raise RuntimeError(f"Select the enabled PME package: {packages}")
pme = importlib.import_module(packages[0] + ".api")
print(pme.ADDON_ID, hasattr(pme, "editing"))
```

Check the needed function and its installed contract. Availability follows actual
capabilities, not a tested-version list. The [API reference](https://docs.pie-menu-editor.com/reference/public_api.html)
is the signature/result authority; these recipes explain how to choose and use it.
Stored Command/Poll/Getter/Custom code has its own execution context: `C` and `L`
inside stored code are not globals in the authoring Console script.

## Choose only the reference needed now

| Task | Read |
|---|---|
| Read by UID, patch a slot, create a menu, set shortcuts/tags, duplicate/delete; choose API versus JSON | [Editing recipes](references/editing.md) |
| Decide Keymap/Poll/Router, tool versus operator, visible switches/Property Stacks, hidden versus grayed items, icons/descriptions; adapt a Maya reference | [Menu design](references/menu-design.md) |
| Choose a single operator, compound Command, Macro Operator or Modal Operator; keep Adjust Last Operation; share variables between steps; open an editor with `popup_area` | [Actions](references/actions.md) |
| Add/move/transfer items, rows, columns/subrows, Layout Panels and headers; fit a panel into a narrow slot; button background color | [Layout recipes](references/layouts.md) |
| Reconcile conflicts, verify input/display/action/Undo/save, try a command outside the user's session, assess an older workaround, or offer a report when PME itself is at fault | [Verification and limits](references/verification.md) |

Do not load every reference for a one-slot edit.

## Keep the edit small and observable

1. Identify the intended menu by name if needed, retain its UID, and read the
   target slots/settings. `PMHandle` is metadata, not raw RNA. Do not invent UIDs.
2. Form the smallest requested change. Use the operation's saved-value `expected`,
   digest, and/or dry-run where supported. There is no universal dry-run signature.
   Preserve the user's concurrent edits; a version alone is not a complete guard.
3. Inspect the result, diagnostics and actual readback before the next dependent
   edit. Creation can succeed while refresh fails; use the returned UID instead
   of creating again. Batch slot saves are sequential, not atomic.
4. On stale input, partial application, unknown outcome or a timeout, read what
   actually happened and reconcile. Do not blindly resend. A timeout does not
   cancel Blender's work. Use small batches with a result/progress record.
5. Verify the requested normal input and visible/action result. Save Preferences
   within the user's authorized scope; editing calls do not save to disk or
   supply a general editing Undo history. Verify restart persistence when needed.

Use one operation/control per direction. Put common eligibility in Poll and
branch-specific differences in individual Router conditions; do not build a
Router for a one-to-one entry. Preserve useful reuse of one menu by multiple
real entries. Prefer visible property widgets for state and tools for continuing
work when they fit Blender's behavior.

Report what changed, the relevant result/UIDs, verification actually performed,
and any unfinished route. Ask the user only for normal input that the available
tools cannot deliver; group those checks after agent-side work. Do not export or
publish unrelated personal menus, a whole live snapshot, or private local paths.
