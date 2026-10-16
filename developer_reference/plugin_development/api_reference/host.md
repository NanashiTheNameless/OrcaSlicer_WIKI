# Host

`orca.host` (bound in `src/slic3r/plugin/host/`) gives plugins **read-only** access to the running
slicer. It is intended for analysis, reporting, and export plugins; nothing here mutates the
model. The one state-changing exception is [local bundle reload](#local-bundle-reload), which
re-reads a user's local preset bundle from disk and refreshes the preset UI - it does not edit or
create presets itself.

**Script plugins run on the main/UI thread**, so within one `execute()` the model cannot
change under you. **Slicing-pipeline plugins run on a background thread** while the GUI
keeps running, and printer-agent methods may be called on the UI thread or on job threads. Either way, treat everything as a momentary snapshot and
do not stash references across runs.

## Entry Points

Each entry point raises `RuntimeError` if called before the GUI/model is ready:

```python
import orca
model  = orca.host.model()           # the active Model
plater = orca.host.plater()          # the Plater
bundle = orca.host.preset_bundle()   # presets (prints/printers/filaments/...)
lang   = orca.host.app_language()    # UI language code, e.g. "en_US", "ru_RU"
store  = orca.host.plugin.storage()  # this plugin's private storage folder
```

`app_language()` returns the language OrcaSlicer's UI is running in, so a plugin can localize
its own dialogs. Use it instead of reading the app config file, which the audit hook blocks.

`plugin.storage()` returns a folder reserved for the calling plugin, creating it if needed, so
the plugin can keep data across runs and updates. A local plugin's folder is
`orca_plugins/plugin_data/<plugin_key>`; a cloud plugin's is per signed-in user
(`orca_plugins/plugin_data/_subscribed/<user_id>/<plugin_key>`), so it raises `RuntimeError`
when no user is signed in. It must be called from a plugin callback (`execute()`, `on_load()`,
a message handler, ...), not at import time.

The `Plater` returned by `plater()` also exposes `model()`, `is_project_dirty()`,
`is_presets_dirty()` and `inside_snapshot_capture()`.

## Model Graph

`Model.objects()` returns `ModelObject` values. Each object has `volumes()` / `volume(i)`
(returning `ModelVolume`) and `instances()` / `instance(i)` (returning `ModelInstance`).
Bounding boxes are a `host.BoundingBox` value type (`min` / `max` / `size` / `center` as
`(x, y, z)` mm tuples, plus `radius` / `defined`).

## Presets

`orca.host.preset_bundle()` returns the live `PresetBundle`: the collections the GUI is editing,
plus the selected and edited presets. All of it is a read-only view of the loaded presets.

### Collections

`PresetBundle` exposes five `PresetCollection` attributes:

| Attribute | `PresetType` | Presets |
|---|---|---|
| `bundle.prints` | `Print` | FDM print (process) presets |
| `bundle.printers` | `Printer` | printer presets |
| `bundle.filaments` | `Filament` | FDM filament presets |
| `bundle.sla_prints` | `SlaPrint` | SLA print presets |
| `bundle.sla_materials` | `SlaMaterial` | SLA material (resin) presets |

A `PresetCollection` gives read-only access to its presets:

| Call | Returns | Notes |
|---|---|---|
| `size()` | `int` | number of presets in the collection |
| `selected_preset()` | `Preset` | alias `get_selected_preset()` |
| `selected_preset_name()` | `str` | alias `get_selected_preset_name()` |
| `edited_preset()` | `Preset` | alias `get_edited_preset()`; the selected preset with unsaved edits |
| `preset(index)` | `Preset` | by index; raises `IndexError` when out of range |
| `find_preset(name)` | `Preset` or `None` | `None` when no preset has that name |
| `preset_names()` | `list[str]` | names of every preset in the collection |

### Preset

A `Preset` exposes read-only metadata and its own configuration:

| Member | Returns | Notes |
|---|---|---|
| `type` | `PresetType` | the kind of preset |
| `name`, `alias`, `file`, `bundle_id` | `str` | `alias` / `file` may be empty |
| `is_default`, `is_external`, `is_system`, `is_visible`, `is_dirty`, `is_compatible`, `is_project_embedded` | `bool` | read-only flags |
| `is_user()` | `bool` | the preset lives in the user's preset folder |
| `is_from_bundle()` | `bool` | the preset came from a bundle rather than being user-created |
| `label(no_alias=False)` | `str` | display name; `no_alias=True` returns the raw name |
| `config_keys()` | `list[str]` | keys set on this preset's own configuration |
| `config_value(key)` | value or `None` | value for `key` on this preset; `None` when it is not set |

### Current presets and full configuration

| Call | Returns | Notes |
|---|---|---|
| `current_process_preset()` | `Preset` | the edited **print** preset |
| `current_print_preset()` | `Preset` | the same edited print preset as `current_process_preset()` |
| `current_printer_preset()` | `Preset` | the edited printer preset |
| `current_filament_preset_names()` | `list[str]` | names of the currently selected filaments |
| `current_filament_presets()` | `list[Preset or None]` | the selected filament presets; an entry is `None` when its name cannot be resolved |
| `full_config_keys()` | `list[str]` | keys of the bundle's full (flattened) configuration |
| `full_config_value(key)` | value or `None` | value for `key` in the full configuration |

`current_process_preset()` and `current_print_preset()` are two names for the same edited print
preset, and `current_printer_preset()` is the edited printer preset. There is no single "current
filament" call: a multi-extruder setup can select several filaments, so iterate
`current_filament_presets()` (or their names). `config_value()` reads a key as the preset sets it,
while `full_config_value()` reads it as it resolves across the whole bundle.

### PresetType

`orca.host.PresetType` identifies a preset's kind: `Invalid`, `Print`, `SlaPrint`, `Filament`,
`SlaMaterial`, `Printer`, `PhysicalPrinter`, `Plate` and `Model`. Only `Print`, `Printer`,
`Filament`, `SlaPrint` and `SlaMaterial` have a matching `PresetCollection` attribute above.

### Local bundle reload

`orca.host.preset.bundles` is the one **state-changing** part of `orca.host`: it re-reads a user's
local preset bundles from disk and refreshes the preset UI. It does not edit, create, import or
export presets - the plugin API exposes no such calls.

```python
bundles = orca.host.preset.bundles
for bundle_id in bundles.list_ids():
    bundles.reload_local(bundle_id)
```

| Call | Effect |
|---|---|
| `list_ids()` | IDs of the local bundles in the active user's `_local` directory that `reload_local()` accepts, including bundles added after startup |
| `reload_local(bundle_id)` | re-read that bundle from the active user's `_local` directory and refresh the preset UI (side preset list, tab configuration, preset comboboxes) |

Both work on local bundles only and marshal to the UI thread, so they are safe to call from a
background-thread plugin. Each raises `RuntimeError` if the app is closing, or the preset bundle
or main window is not available; `reload_local()` also raises it when the bundle cannot be
reloaded (unknown ID, or a parse error).

> [!NOTE]
> Re-list with `list_ids()` rather than caching it: it includes bundles added after startup.

## Mesh Geometry

`ModelVolume.mesh()` returns `host.TriangleMesh`.

| Member | Returns | Notes |
|---|---|---|
| `vertex_count()` / `triangle_count()` (`facets_count()`) / `is_empty()` | `int` / `bool` | numpy-free |
| `vertex(i)` / `triangle(i)` | `(x, y, z)` / `(a, b, c)` tuple | numpy-free, bounds-checked |
| `vertices()` | `(N, 3)` float32 ndarray | **read-only, zero-copy**, requires numpy |
| `triangles()` | `(M, 3)` int32 ndarray | vertex indices; **read-only, zero-copy**, requires numpy |
| `face_normals()` | `(M, 3)` float32 ndarray | computed copy, requires numpy |
| `volume()` / `bounding_box()` / `is_manifold()` | `float` / `BoundingBox` / `bool` | numpy-free |

Coordinates are **local** (the volume's own frame, in mm). The `vertices()` / `triangles()`
arrays are zero-copy views into the live mesh and are marked read-only; writing to them raises
`ValueError`. Their lifetime is pinned to an immutable mesh snapshot, so they stay valid even
if the volume's mesh is later replaced.

## Worked Example

Declare numpy in the PEP 723 block so the bundled `uv` installs it:

```python
# /// script
# dependencies = ["numpy"]
# ///
import orca, numpy as np

class MeshReport(orca.script.ScriptPluginCapabilityBase):
    def get_name(self): return "Mesh Report"
    def execute(self):
        model = orca.host.model()
        for obj in model.objects():
            for vol in obj.volumes():
                mesh = vol.mesh()
                V = np.asarray(mesh.vertices())          # (N, 3) float32, read-only
                T = np.asarray(mesh.triangles())         # (M, 3) int32
                # World-space coordinates for the first instance (row-vector convention):
                M = obj.instance(0).matrix() @ vol.matrix()      # 4x4 float64
                world = (np.c_[V.astype(np.float64), np.ones(len(V))] @ M.T)[:, :3]
                print(vol.name, V.shape, T.shape, world.min(0), world.max(0))
        return orca.ExecutionResult.success()

@orca.plugin
class MeshReportPlugin(orca.base):
    def register_capabilities(self):
        orca.register_capability(MeshReport)
```

> [!NOTE]
> If the instance is mirrored (`instance.is_left_handed()` is `True`, i.e. `det(M) < 0`),
> flip triangle winding / negate face normals when computing outward-facing normals in world
> space.

## numpy Requirement

`vertices()`, `triangles()`, `face_normals()`, and the `matrix()` accessors on
`ModelVolume` / `ModelInstance` require numpy and raise a clear `ImportError` if it is not
installed (declare `dependencies = ["numpy"]`). Everything else - counts, `vertex(i)` /
`triangle(i)`, `volume()`, `bounding_box()`, `is_manifold()`, and the `offset` / `rotation` /
`scaling_factor` / `mirror` tuple accessors - works without numpy.
