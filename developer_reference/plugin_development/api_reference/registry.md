# Registry

The embedded plugin API is exposed through a single Python module named `orca`
(`PYBIND11_EMBEDDED_MODULE(orca, ...)` in `PythonPluginBridge.cpp`). It provides the
capability base classes, the package base, result types, and capability registration.

| Symbol | Kind | Members / purpose |
|---|---|---|
| `orca.PluginType` | enum | `PrinterConnection`, `Pages`, `Analysis`, `Importer`, `Exporter`, `Visualization`, `Script`, `SlicingPipeline`, `ResourceTexture`, `Unknown` |
| `orca.PluginResult` | enum | `Success`, `Skipped`, `RecoverableError`, `FatalError` |
| `orca.PluginContext` | class | base context, field `orca_version: str` |
| `orca.ExecutionResult` | class | fields `status`, `message`, `data`; factories below |
| `orca.LifecycleEvent` | enum | [Lifecycle Events](lifecycle_events): the application moments passed to `on_lifecycle_event` |
| `orca.LifecycleEvtCode` | enum | `Ok`, `Error`, `Warn` - the outcome carried by a lifecycle event |
| `orca.LifecycleEventContext` | class | the read-only payload of a lifecycle event |
| `orca.PythonPluginBase` | class | the root **capability** base; subclasses must implement `get_name()` |
| `orca.base` | class | the **package** base; subclass it and override `register_capabilities()` |
| `orca.plugin` | decorator | marks the single package class for the file (exactly one per file) |
| `orca.register_capability(cls)` | function | register one capability class; call it inside `register_capabilities()` |
| `orca.request_permissions(...)` | function | declare filesystem, network, process, and thread permissions up front; call it inside `register_capabilities()` (see [Permissions](#permissions)) |
| `orca.slicing` | submodule | [Slicing Pipeline](slicing): `Step`, `SlicingPipelineContext`, `SlicingPipelineCapabilityBase` |
| `orca.script` | submodule | [Script](script): `ScriptPluginCapabilityBase` |
| `orca.pages` | submodule | [Pages](pages): `PagesPluginCapabilityBase`, a plugin tab in the main window |
| `orca.resource_texture` | submodule | [Resource Texture](resource_texture): `ResourceTexturePluginCapabilityBase` |
| `orca.printer_agent` | submodule | [Printer Agent](printer_agent): `PrinterAgentBase` and its data types |
| `orca.host` | submodule | [Host](host): read-only host access to the live model, presets, and mesh geometry |
| `orca.host.ui` | submodule | [Host UI](host_ui): dialogs, progress dialogs, interactive windows, docked panels, and notifications |

Every capability inherits the optional hooks `on_load()`, `on_unload()` and
`on_lifecycle_event(event, ctx)` (see [Lifecycle Events](lifecycle_events)) from
`orca.PythonPluginBase`, along with these configuration methods:

| Method | Purpose |
|---|---|
| `has_config_ui()` | Select a custom HTML editor instead of the host JSON editor. |
| `get_config_ui()` | Return the custom HTML editor page. |
| `get_default_config()` | Return a Python dict used by **Restore defaults**; the Python call returns its JSON string. |
| `get_config()` | Return the effective configuration as a JSON string. |
| `get_config_version()` | Return the version that last saved the configuration. |
| `save_config(config)` | Persist a JSON string for this capability; returns `False` on invalid JSON or write failure. |

Use `json.loads()` and `json.dumps()` at this boundary. See
[Capability configuration](plugin_development#capability-configuration) for storage layers and
custom UI messaging.

## Execution Results

`ExecutionResult` is how a plugin reports the outcome of a run:

```python
orca.ExecutionResult.success(message="", data="")
orca.ExecutionResult.skipped(message="")
orca.ExecutionResult.failure(status, message, data="")   # status is an orca.PluginResult
```

- `status` - an `orca.PluginResult`.
- `message` - human-readable text; this is what surfaces in error/result dialogs.
- `data` - a free-form string whose meaning is defined by the plugin/workflow (not
  interpreted by the framework).

## Registration

Registration has two parts, both resolved at **module import / load time**.

**Capabilities** - each capability is a class that subclasses a typed base and implements
`get_name(self) -> str`. The name is how the capability appears in the UI and how presets
refer to it, so it must be **unique within the plugin** and **must not contain a `;`**. That
character is reserved as a separator in preset references, and a `;` in a capability name
fails the load.

**The package** - exactly one class per file is decorated with `@orca.plugin` and subclasses
`orca.base`. Its `register_capabilities(self)` method calls `orca.register_capability(Cls)`
once for each capability class you want to expose:

```python
@orca.plugin
class SamplePlugin(orca.base):
    def register_capabilities(self):
        orca.register_capability(GCodeBenchmark)
        orca.register_capability(EnvironmentReport)
```

OrcaSlicer instantiates the package class (it must be callable as `SamplePlugin()` with no
arguments), calls `register_capabilities()`, then instantiates each registered capability.

Rules enforced when a plugin loads (most in `PythonPluginBridge.cpp`):

- The `@orca.plugin` class **must** subclass `orca.base`, and there must be **exactly one**
  per file. A second `@orca.plugin` fails the load.
- Each class passed to `orca.register_capability` must subclass a capability base (ultimately
  `orca.PythonPluginBase`); otherwise it raises `value_error`.
- Every capability must resolve `get_name()` (checked in the bridge); the loader
  (`PluginLoader.cpp`) additionally rejects the plugin if the resulting `(type, name)` pair is
  not unique across it.
- A capability class you never pass to `orca.register_capability` is **invisible** to
  OrcaSlicer, even if it is defined in the file.
- If the plugin calls `orca.request_permissions(...)`, the user must accept the combined
  dialog that follows registration; declining it fails the load.

## Permissions

`orca.request_permissions(...)` declares, from `register_capabilities()`, the filesystem,
network, process, and thread access the plugin expects. All parameters are optional:

```python
orca.request_permissions(
    fs_read=[],           # filesystem paths to read
    fs_readwrite=[],      # filesystem paths to read/write
    network_http=[],      # URLs
    network_socket=[],    # socket addresses, as str() renders them
    process=[],           # commands / argv
    threading=False)      # True requests the fixed "thread" target
```

- Each list takes the target strings the corresponding audit event reports: paths for the
  filesystem buckets, a URL for `network_http`, an address string such as
  `"('example.com', 443)"` for `network_socket`, and a command for `process`.
- `threading=True` requests permission to create one thread; thread creation is granted once
  per plugin, and the new thread's own work is not audited.
- OrcaSlicer collects the declarations while `register_capabilities()` runs and shows **one
  combined Yes/No dialog** after it returns and before any capability instance is created.
  Targets already granted to the plugin are skipped, so declaring them again does not prompt.
- Declining the dialog aborts the plugin load. An empty string in any list, or calling the
  function outside plugin discovery, raises `ValueError`.
- A declaration pre-approves only the exact target strings listed; any other access still
  prompts on first use. See [Plugin Audit Hook](plugin_audit_hook) for the audit categories,
  persistence, and off-thread prompt behavior.

## Capability Names and Types

`get_name()` is required; `get_type()` usually is not. Every capability must implement
`get_name()`. It is pure virtual on the root base, and a missing override fails the load.
The typed C++ bases already implement `get_type()` (for example,
`ScriptPluginCapability::get_type()` returns `Script`), so a subclass of a *typed* base does
not need to override it. Only a capability that subclasses the **root**
`orca.PythonPluginBase` directly must set its own `get_type()`.
