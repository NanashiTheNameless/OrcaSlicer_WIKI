# Plugin Types

A plugin can provide one or more features. Expand a plugin row to see its plugin capabilities.

![local-plugin-activated-capabilities](https://github.com/NanashiTheNameless/OrcaSlicer_WIKI/blob/main/images/plugins/local-plugin-activated-capabilities.png?raw=true)

## Script

Script plugins are run manually from the Plugins window.

1. Open **File** > **Plugins**.
2. Expand the plugin.
3. Find the script feature.
4. Click the **Run** button.

![script-plugin-run-result](https://github.com/NanashiTheNameless/OrcaSlicer_WIKI/blob/main/images/plugins/script-plugin-run-result.png?raw=true)

## Slicing Pipeline

[Mode](option_mode): `Advanced`.  
[Variable](built_in_placeholders_variables): `slicing_pipeline_plugin`.  
[Type](option_type#list-types): `Text list`.  
[CLI Example](cli_mode#setting-overrides): `--slicing-pipeline-plugin=value`.  
Slicing-pipeline plugins run at selected points in the slicing workflow. They can inspect or modify
the live slicing graph, or edit the exported G-code at the post-processing step. This capability is
experimental and intended for plugin developers and testers.

The plugin's capability appears in the appropriate plugin setting or workflow when it is enabled.
The `psGCodePostProcess` step runs after the classic post-processing scripts and receives the
working G-code path; geometry steps receive the active slicing context instead.

> [!WARNING]
> Slicing-pipeline plugins can change generated geometry or G-code. Review and test plugins from
> sources you trust before using them for a print.

## Pages

Page plugins add their own tab to the main window, after OrcaSlicer's built-in tabs. The tab
appears while the plugin feature is activated and disappears when it is deactivated.

If many page plugins are activated, set how many tabs are shown side by side in **Preferences** >
**General** > **Plugins** > **Visible plugin pages** (1 to 10, default 5). The remaining pages
are listed in a dropdown on the last plugin tab.

## Resource Textures

Resource-texture plugins contribute height-map images as a resource pack. Each plugin provides one pack of images, and the pack's name is how it appears in the app.

These plugins install as wheels (`.whl`) only, because a pack ships image files alongside its code.

Enabling, disabling, or uninstalling a pack adds or removes its textures, and the change shows up without a restart. A project that has already used a pack texture keeps working, because it stores its own copy of the image.

## Plugin Configuration

Select a plugin, open the **Config** tab, and select one of its capabilities. Edit the JSON
configuration and click **Save**. **Restore defaults** removes the saved configuration for that
capability and uses the plugin's default configuration.

When a preset uses a plugin capability, its configuration can also be overridden for that preset.
The override is stored with the preset when you save it; otherwise the capability uses its global
configuration.

## Speed Dial

The [Speed Dial](speed_dial) allows quick access to installed plugins.

## Printer Connection

Printer connection plugins add printer communication features. The Printer Agent workflow is still WIP.

For printers that are not Bambu Lab printers, OrcaSlicer only sends print jobs through a printer connection plugin when **Preferences** > **Developer** > **Experimental Features** > **Use printer agents instead of print hosts** is enabled. It is off by default, so the classic print host upload is used.

Each printer connection has an ID. If a plugin's printer connection uses an ID that another plugin or a built-in connection already uses, OrcaSlicer shows a warning, turns that feature off, and marks the plugin with a **RuntimeError** status. See [Managing Plugins](plugins_management#plugin-status).

If the plugin has its own setup instructions, check the **Description** or **Diagnostics** tab in the Plugins window.
