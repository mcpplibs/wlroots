# wlroots, for mcpp

wlroots 0.20.2 as an mcpp package, with `upstream/` untouched and everything
this fork adds under `mcpp/`.

```toml
[dependencies]
wlroots.wlroots = "0.20.2"
```

```cpp
import wlroots;

wlr_scene *scene = wlr_scene_create();
```

## Why a fork rather than a descriptor

The criterion in this ecosystem is **generators, not line count**. wlroots has
six:

| upstream | here |
|---|---|
| 45 protocols → `wayland-scanner` | 13 scanned, 32 forwarded from the index |
| `backend/drm/gen_pnpids.sh` | `write_pnpids()` over a **pinned** hwdata |
| `render/gles2/shaders/embed.sh` | `embed_shaders()` |
| `configure_file` → `wlr/config.h` | `write_public_config()` |
| `configure_file` → `config.h` | `write_internal_config()` |
| `configure_file` → `wlr/version.h` | `write_version()` |

Three of those are shell scripts upstream. **There is no `sh` and no `python`
in this tree**: `build.mcpp` is a compiled C++ program, so a generator is just
a function in it.

## `import wlroots;` is not a convenience — it is the only way in

wlroots' 121 public headers contain **not one `extern "C"` block**, and two of
them are not valid C++ at all:

```c
void wlr_scene_rect_set_color(struct wlr_scene_rect *rect,
                              const float color[static 4]);   /* C99 only */
```

A C++ translation unit cannot include those headers under any arrangement of
`extern "C"`. The module is where the adaptation happens: `[static N]` reduced
to `[N]` in copies of the two affected headers, and the three struct members
whose names are C++ keywords given a `#ifdef __cplusplus` spelling at the same
offset.

| upstream C | C++ |
|---|---|
| `wlr_layer_surface_v1::namespace` | `::namespace_` |
| `wlr_input_method_v2::delete` | `::delete_` |

Two things the module cannot do, both inherent:

* **Macros do not cross a module boundary.** `WLR_HAS_*`, `wl_container_of`,
  `wl_list_for_each` come from headers. `#include <wlr/config.h>` alongside the
  import is safe — it is macros only.
* **A header that DECLARES something must not be included alongside the
  module.** `<wlr/version.h>` has no `extern "C"`, so including it gives those
  three names C++ linkage while the module's have C linkage, and the link fails
  with ``undefined reference to `wlr_version_get_major()'`` — the parentheses
  are the tell.

## Features

```toml
wlroots.wlroots = { version = "0.20.2", default-features = false,
                    features = ["session", "drm"] }
```

Default: `drm`, `libinput`, `session`, `gles2`, `gbm`. Also available:
`udmabuf`. `drm` and `libinput` both require `session`, and saying so is an
error at configure time rather than a link failure naming `wlr_session_*`.

Not offered, each for a stated reason rather than an omission:

| | why |
|---|---|
| `vulkan-renderer` | needs `glslang` to compile shaders to SPIR-V; not in the index |
| `x11-backend`, `xwayland` | need xcb; the point of this stack is a compositor that does not drag X11 in |
| `color-management` | needs lcms2; `color_fallback.c` is built, which is upstream's own arrangement |

**The feature selection lives in `build.mcpp`, not in `[features].sources`.**
Feature sources match literal entries, so a file a glob excluded with `!` can
never be added back — and wlroots keeps feature-gated files inside otherwise
unconditional directories (`types/wlr_drm_lease_v1.c` among 108 siblings).
`mcpp::has_feature()` answers inside the build program, so the whole
conditional graph is one function.

## The pinned `pnp.ids`

`mcpp/data/pnp.ids` is hwdata v0.410, checked in. Upstream reads
`/usr/share/hwdata/pnp.ids` **off the build machine**, so the manufacturer a
monitor reports would depend on where the package was built. Same trap
`freedesktop.libdisplay-info` documents, same fix.

The generator is also stricter than upstream's: `PNP_ID` keeps only the low
five bits of each character, so ids differing in case collide — hwdata really
does carry `inu` alongside uppercase ids — and a collision is reported naming
both rather than emitted as two identical `case` labels.

## Layout

```
upstream/            wlroots 0.20.2, byte for byte (CI diffs it)
mcpp/data/pnp.ids    pinned hwdata v0.410
mcpp/wlroots/
  mcpp.toml          manifest; `include/` is FIRST in include_dirs, deliberately
  build.mcpp         every generator, and the feature selection
  include/           GENERATED, not checked in (only .gitkeep)
  src/wlroots.cppm   GENERATED module interface
  tests/wlroots.cpp
```
