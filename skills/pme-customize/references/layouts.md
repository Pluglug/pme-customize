# Layout recipes

Use observed menu/item UIDs and freshly read `rows`. `row_ref` is a current saved
position, not a permanent row identity. Re-read after each structural change.
The [API reference](https://docs.pie-menu-editor.com/reference/public_api.html#local-menu-editing)
owns full signatures, supported menu types and diagnostics.

## Choose the unit to edit

| Intended change | Operation and important input |
|---|---|
| Add/remove one list item | `add_slot` / `remove_slot`; normal items, not Panel-owning anchors |
| Reorder a list item | `move_slot` with exactly one `before_uid` / `after_uid` |
| Exchange fixed directions | `swap_slots(pm_uid, first_uid, second_uid)`; UIDs move with content |
| Move content to another menu | `transfer_slot`; fixed destination must be empty, retains its UID; list destination gets a new UID |
| Add/remove/reorder a whole row | `add_row` / `remove_row` / `move_row`; a Panel moves/removes as a whole |
| Split/join ordinary rows | `split_row(..., before_uid=...)` / `join_rows(pm_uid, first_row_ref, second_row_ref)` |
| Add a Layout Panel | `add_panel`; configure its returned anchor through `update_slots` |
| Row size/spacing/width/alignment | `update_row`; settings/enums come from current row readback |
| Column/subrow/spacing boundary | `update_spacer`; no raw marker-string authoring |
| Move alignment boundaries | `align_slot` when its supported layout can express the intended placement |

Regular Menu Add kinds are `ITEM`, `LABEL`, `SPACER`, `SEPARATOR`, `COLUMN`.
Popup/Floating Panel Add uses `kind=None` or `"BUTTON"`. Add with no destination
appends, including to an empty menu. It does not add an implicit row marker.
For new item content, read the committed `slots[0]["uid"]`, then use the guarded
`update_slots` recipe in [editing](editing.md). Preview creates no item UID.

Flat-list Move accepts `dry_run=True` but rejects non-`None` `expected`; use its
version and re-read. Flat Add accepts the complete preview snapshot; flat Remove
keeps its target-field subset contract. These are not Popup snapshot contracts.

## Move a Popup item to a row

For Pop-up Dialog/Floating Panel Add/Move/Remove/swap and row/settings operations,
query first, inspect diagnostics and resolved position, then apply the same
inputs with the complete returned `expected` and `version`. A slot readback alone
is not that complete snapshot. This example moves an observed item to another
observed ordinary row's end:

```python
before = pme.editing.read_pm(pm_uid)
print(before["rows"])  # Choose destination_row_ref from this current read.
checked = pme.editing.move_slot(pm_uid, slot_uid,
    row_ref=destination_row_ref, edge="END",
    expected_version=before["version"], dry_run=True)
if checked["status"] == "validated":
    print(checked["insertion_position"], checked["diagnostics"])
    result = pme.editing.move_slot(pm_uid, slot_uid,
        row_ref=destination_row_ref, edge="END",
        expected_version=checked["version"], expected=checked["expected"])
    print(result["status"], result["after_rows"], result["structure_changes"])
else:
    print(checked)
```

Other Popup positions are `before_uid` / `after_uid`, a current ordinary `row_ref`
with `edge="START"` / `"END"` (including empty rows), or `new_row="END"`.
These forms are exclusive. Relative placement uses the immediate saved gap and
does not skip spacers, COLUMN or subrow markers. Self/already-adjacent moves are
unchanged without cleanup or refresh. `use_spacer=False` is independent of
Preferences; True requests the normal automatic spacer. Flat Add rejects True.

`insertion_position` describes the **before** saved order, including markers and
the source item. It is not the final index after source removal/cleanup, and not
a column/subrow nesting tree. Transfer returns it under `to`. An unresolved
position is `None`. Use actual `after_order` / `after_rows` for subsequent work.
`rows` stays complete even when `slot_uids` filters item details.

## Transfer content between menus

A list destination takes before/after, or neither if empty. A Popup also takes
the row/new-row forms above. To enter a Panel header, always qualify its anchor,
even if Header Layout is on:

```python
position = {"header_anchor_uid": panel_anchor_uid, "edge": "END"}
checked = pme.editing.transfer_slot(
    source_pm_uid, source_slot_uid, destination_pm_uid, **position, dry_run=True)
if checked["status"] == "validated":
    result = pme.editing.transfer_slot(
        source_pm_uid, source_slot_uid, destination_pm_uid,
        **position, expected=checked["expected"])
    print(result["status"], result["from"], result["to"], result["diagnostics"])
else:
    print(checked)
```

The full `expected` guards both menus. Transfer clears a fixed source or removes
a list source; content and enabled state move. Markers/Panel-owning anchors are
not transferable content. Cleanup is local; unrelated empty rows/spacers remain.
An OFF header stays OFF (`header_layout_disabled`); taking its last item out
leaves it empty. Inspect both sides and their actual new UIDs/rows.

## Rows, columns and Panel headers

- `add_row` / `add_panel` accepts before/after row, or neither to append.
  Add Row creates a default button. Read the new Panel's actual `anchor_uid`
  from `after_rows`; configure name, `menu_target_uid`,
  `layout_panel_default_closed`, `layout_panel_use_header_layout` with guarded
  `update_slots`. Do not supply a raw `layout_panel` dictionary. An anchor
  without `menu_target_uid` rejects every `update_slots` edit
  (`editor_validation` "Select the item"); set `menu_target_uid` in the same
  edit or first.
- `remove_row` removes the whole row; the last row may be removed. `move_row`
  takes one before/after row destination. A Panel includes inactive header items.
- `split_row` starts a new default right row at the chosen item/spacer, without
  creating a button. It removes one immediately preceding spacer. Splitting
  before the first item leaves an empty left row; a row marker is not a split
  point. `join_rows` requires adjacent rows in that order, keeps first settings,
  removes the second marker and whole COLUMN/ALIGNER spacers, and retains other
  spacers. Panel rows cannot split/join. Inspect `aligner_without_root` instead
  of assuming automatic alignment repair.
- `update_row(..., changes, scope="row")` edits `size`, `vspacer`, `fixed_col`,
  `fixed_but`, saved `align`. Use current enums and Bool fixed-width flags.
  `aligned_rows` edits the spacing group; `all_rows` allows `row_ref=None`.
- `update_spacer(..., slot_uid=..., prop="hsep", value="COLUMN")` changes an
  existing spacer; one before/after content anchor can also create it. Change
  one property per call: `hsep=NONE/SPACER/COLUMN/ALIGNER`,
  `subrow=NONE/BEGIN/END`. Inspect cleanup and new rows/UIDs after applying.
- `align_slot(pm_uid, slot_uid, "LEFT"/"CENTER"/"RIGHT")` moves alignment boundaries;
  `None` clears them. In mixed layouts the item must precede the first COLUMN,
  with no ALIGNER after that COLUMN. When unsupported, use explicit spacer edits
  that express the request; do not silently change the desired placement.

Header Add/Move takes `header_anchor_uid` plus exactly one before/after or edge.
Edges work in empty headers. Moving into another header requires the qualifier;
relative moves inside the same header do not. ON-header relative Add retains
its unqualified form; OFF-header Add requires the qualifier. Cross-menu Transfer
always requires it. Header positions cannot combine with row/new-row positions.
These edits do not switch Header Layout on. Only an explicit OFF-to-ON transition
on an empty header creates its initial button. Do not move an owning anchor into
another header; move its whole row. `enabled` is not an `update_slots` input.

## Arrange a Floating Panel for narrow widths

A Floating Panel expanded in a Vector Menu slot is narrow, and long labels are
cut. Its width belongs to the slot that links it: set `menu_expand_width` on
that Menu slot, not on the panel. Judge the fit from the drawn panel in the
user's UI language and scale; the displayed text can differ from the saved name.

Give each row one topic and one of these shapes:

| Row shape | Build |
|---|---|
| Label on the left, short values or toggles on the right | A Custom label slot (`L.label(text=text, icon=icon, icon_value=icon_value)`), then `update_spacer(pm_uid, after_uid=label_uid, prop="hsep", value="ALIGNER")`, then the value slots |
| Two long toggles filling the width | Two Property slots in one row |
| A numeric field | Alone in its row; its label is drawn inside the field |

An Enum property drawn by a Property slot shows its value without a label; put
a label slot before it. Use the short names Blender's own UI uses.

To regroup an existing panel, take `row_ref` from a fresh `read_pm()["rows"]`
and combine `split_row(pm_uid, row_ref, before_uid=...)`,
`move_slot(pm_uid, slot_uid, after_uid=...)` and
`add_slot(pm_uid, before_uid=...)` followed by `update_slots` for the new
item's content. Read the menu again before every step: each call changes the
version and the row references, and the user may be editing the same panel.

A Command slot in a Floating Panel or Vector Menu can have a background color:
`{"button_background_color_enabled": True, "button_background_color": [r, g, b]}`.

- The three numbers are scene linear, not sRGB. Convert a hex or sRGB channel
  `c` (0-1) first: `c / 12.92 if c <= 0.04045 else ((c + 0.055) / 1.055) ** 2.4`.
  An sRGB value written unchanged is drawn lighter than intended.
- Set both fields. The color is drawn only while
  `button_background_color_enabled` is `True`; read both back.
- The text color does not change. Compare the color with the theme's button
  text (`bpy.context.preferences.themes[0].user_interface.wcol_tool.text`) and
  check the drawn button.
- Some builds show the custom background color only when the Floating Panel opens
  on its own. In those builds, the color is missing when the panel is expanded
  in a Vector Menu slot.

For all structural edits, inspect `structure_changes`, actual row references and
diagnostics. Affected open drafts can reject an operation; unrelated drafts and
selection are retained. Add/Move/Remove have different cleanup behavior. A query
does not predict cleanup or prove rendering. On `popup_readback_failed`, unknown
`after_order` / `after_rows` are `None`, not empty; a missing read does not undo a
commit. `restored_failure` describes saved-data recovery, not host recovery.
Read both sides of a failed transfer before retrying. These APIs neither save
Preferences nor provide editing Undo. See [verification](verification.md).
