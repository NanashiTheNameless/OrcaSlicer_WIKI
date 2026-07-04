# G-code

The `orca.gcode` module exposes `GCodePluginCapabilityBase` for post-processing plugins that
run during G-code export.

| Base class | `get_type()` returns | Required methods | Invoked by |
|---|---|---|---|
| `orca.gcode.GCodePluginCapabilityBase` | `PostProcessing` | `get_name()`, `execute(self, ctx) -> ExecutionResult` | **G-code export / post-processing** during slicing |

`GCodePluginCapabilityBase` runs on a background slicing thread. Lifecycle hooks
`on_load()` / `on_unload()` are optional and available on every capability (defaults do
nothing).

## Context

The G-code context (`orca.gcode.GCodePluginContext`) is passed to `execute` and exposes
read/write fields:

| Field | Meaning |
|---|---|
| `orca_version` | OrcaSlicer version string (inherited from `PluginContext`) |
| `gcode_path` | absolute path to the temporary G-code file being post-processed |
| `host` | target host, when exporting to a network printer |
| `output_name` | the output file name |

> [!IMPORTANT]
> Filesystem access is audited. While `execute()` runs, the audit hook restricts writes to an
> allow-list. G-code plugins additionally get the folder containing `gcode_path` added as a
> scoped writable root, so appending to / rewriting the current G-code file is allowed;
> writing elsewhere outside `data_dir()` is blocked. See
> [Plugin Audit Hook](plugin_audit_hook).

## Minimal Shape

```python
import orca

class EnvironmentReport(orca.gcode.GCodePluginCapabilityBase):
    def get_name(self):
        return "Environment Report"

    def execute(self, ctx):
        with open(ctx.gcode_path, "a", encoding="utf-8") as f:
            f.write(f"\n; processed by Environment Report for {ctx.output_name}\n")
        return orca.ExecutionResult.success("report appended")
```

See [Registry](registry) for `ExecutionResult`, package registration, and capability name
rules.
