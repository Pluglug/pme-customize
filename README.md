# PME Customize

An Agent Skill for creating and editing menus, shortcuts, and controls in [Pie Menu Editor Fork](https://docs.pie-menu-editor.com/) for Blender.

Requires Pie Menu Editor Fork 2.4.0 or later. [Get it on Gumroad](https://pluglug.gumroad.com/l/pie-menu-editor-fork).

## Install

```sh
npx skills add Pluglug/pme-customize --skill pme-customize
```

For manual installation, copy the whole [`skills/pme-customize`](skills/pme-customize) folder into your agent's skills directory.

## Use

Enable Pie Menu Editor Fork in Blender, then ask your agent to customize a workflow. For example, “Add a popup menu with my most-used UV tools.” For live edits, the agent needs access to your Blender session. See the [PME Authoring API guide](https://docs.pie-menu-editor.com/reference/authoring/index.html) for editing workflows and examples.

## License

[GPL-3.0-or-later](LICENSE.md)
