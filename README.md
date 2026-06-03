# Arx Libertatis — HUD Opacity Patch

> Personal mod. Pre-built binary (CachyOS x86-64, Wayland) available in [Releases](https://github.com/starfox8439/ArxLibertatis-hud-opacity/releases) — likely won't run on other systems without rebuilding. Windows support is theoretical and untested. Use at your discretion.

> **Fork of [Arx Libertatis 1.2.1](https://github.com/arx/ArxLibertatis)** — open-source reimplementation of Arx Fatalis.
> Upstream: **https://github.com/arx/ArxLibertatis**

---

## Highlights

- **New in-game slider** — Options → Interface → HUD Opacity, directly below HUD Scale
- **Full HUD fade** — health/mana orbs, icons, stealth gauge, spell slots, screen arrows, all indicators
- **Non-trivial render fix** — Arx's OpenGL texture stage defaults `GL_COMBINE_ALPHA` to `GL_REPLACE`, silently discarding vertex alpha. The patch switches to `GL_MODULATE` for the HUD draw block so opacity actually takes effect. See [GL_REPLACE trap](#the-gl_replace-trap-why-vertex-alpha-alone-isnt-enough)
- **Zero-change default** — `hud_opacity = 1.0`, persisted to `cfg.ini`; vanilla behavior until you touch the slider
- **Modder's guide included** — full walkthrough of the config/menu/render pattern for adding any future HUD setting

---

## What this fork adds

A **HUD opacity slider** in Options → Interface, directly beneath the existing HUD Scale slider.

Moving it fades the entire in-game HUD — health/mana orbs, side icons (backpack, book, torch, purse, steal, level-up, change-level), stealth gauge, damaged-equipment indicators, spell slots, screen-edge arrows, hit-strength gauge, memorized rune display, and gold number — from fully opaque (default) down to invisible. Menus, loading screens, cinematics, and the player book interior are unaffected.

The setting persists to `cfg.ini` as `hud_opacity` in the `[interface]` section and defaults to `1.0` (identical to vanilla behavior until you move the slider).

---

## Screenshots

**Options → Interface — HUD Opacity slider visible below HUD Size:**

![Settings menu showing HUD Opacity slider](screenshots/settings-menu.png)

**In-game with HUD opacity reduced — indicators faded:**

![In-game HUD at reduced opacity](screenshots/hud-faded.png)

---

## Quick start — applying the patch

Two options. Both produce the same result.

### Option A — `patch` command

The diff paths are relative to `src/` (e.g. `a/core/Config.h`), so apply from inside `src/`:

```bash
cd ArxLibertatis-1.2.1/src
patch -p1 < /path/to/arx-hud-opacity.patch
```

### Option B — shell script (self-documenting, no patch binary needed)

```bash
bash apply-hud-opacity.sh /path/to/ArxLibertatis-1.2.1
```

The script prints exactly what it is doing at each step and exits non-zero on any failure.

---

## Building

### Linux (Arch / CachyOS)

```bash
sudo pacman -S --needed base-devel git cmake boost glm libepoxy glew freetype2 openal libpng bzip2 ninja
```

Configure and build:
```bash
mkdir build && cd build
cmake ../ArxLibertatis-1.2.1 -DCMAKE_BUILD_TYPE=RelWithDebInfo -G Ninja
ninja arx
```

Run (game data auto-discovered at `~/.local/share/arx/`):
```bash
./build/arx
```

### Windows (theoretical)

> ⚠️ Not tested on Windows. Based on upstream build docs and standard CMake/MSVC practice.

**Prerequisites:**
- [Visual Studio 2019 or 2022](https://visualstudio.microsoft.com/) with the **Desktop development with C++** workload
- [CMake 3.15+](https://cmake.org/download/)
- [vcpkg](https://github.com/microsoft/vcpkg) for dependencies

**Install dependencies via vcpkg:**
```powershell
git clone https://github.com/microsoft/vcpkg
cd vcpkg
.\bootstrap-vcpkg.bat
.\vcpkg install boost glm sdl2 openal-soft freetype zlib libpng
```

**Apply the patch** (from Git Bash — note: apply from inside `src/`):
```bash
cd ArxLibertatis-1.2.1/src
patch -p1 < /path/to/arx-hud-opacity.patch
```

**Configure and build:**
```powershell
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=C:/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake -DCMAKE_BUILD_TYPE=Release
cmake --build . --config Release --target arx
```

**Game data:** place `.pak` files in `%USERPROFILE%\AppData\Local\ArxLibertatis\` or alongside `arx.exe`. Run `arx.exe --list-dirs` to see all searched paths.

---

## How this patch works — guide for modders

This section is a full walkthrough of the approach so future modders can clone the same pattern for other HUD settings.

### The template: HUD scale

Arx Libertatis 1.2 already ships a HUD *scale* setting. That setting has exactly the plumbing any new HUD option needs:

1. **A config field** — `config.interface.hudScale` (float, 0–1, stored in `cfg.ini`)
2. **A menu slider** — on the Interface options page in `src/gui/MainMenu.cpp`
3. **A render-path consumer** — `HudRoot::recalcScale()` reads the value and calls `setScale()` on every HUD element

The opacity patch copies this pattern exactly, adding `hudOpacity` alongside `hudScale` in each of those three places. If you want to add a *tint colour*, a *blur amount*, or any other per-HUD-element setting, follow the same three-file trail.

### File 1 — `src/core/Config.h`

Declares the data. The `Config` class has an `interface` sub-struct. Add your field here:

```cpp
float hudScale;
bool  hudScaleInteger;
float hudOpacity;   // ← new field, lives right next to hudScale
float bookScale;
```

### File 2 — `src/core/Config.cpp`

Three sub-locations inside this one file:

- **`namespace Default`** — the fallback value loaded when the key is absent from `cfg.ini`. Use `1.0f` for opacity so new installs are unchanged.
- **`namespace Key`** — the string that appears in `cfg.ini`. Use something unambiguous like `"hud_opacity"`.
- **`Config::init()`** — reads the key, clamps it, stores it: `glm::clamp(reader.getKey(...), 0.f, 1.f)`.
- **`Config::save()`** — writes it back: `writer.writeKey(Key::hudOpacity, interface.hudOpacity)`.

### File 3 — `src/gui/MainMenu.cpp`

The Interface options page is `InterfaceOptionsMenuPage`. Its constructor builds the widget list top-to-bottom. Find the `SliderWidget` for `hud_scale` and add a second one immediately after:

```cpp
{
    std::string label = getLocalised("system_menus_options_interface_hud_opacity", "HUD Opacity");
    SliderWidget * sld = new SliderWidget(sliderSize(), hFontMenu, label);
    sld->valueChanged = boost::bind(&InterfaceOptionsMenuPage::onChangedHudOpacity, this, arg::_1);
    sld->setValue(int(config.interface.hudOpacity * 10.f));
    addCenter(sld);
}
```

The slider range is 0–10 (integers); multiply/divide by 0.1 to convert to/from a 0–1 float. The callback just writes to config — no recalc needed because `HudRoot::draw()` reads `config.interface.hudOpacity` live on every frame.

The second argument to `getLocalised()` is a fallback string used when the localization key is absent (i.e., always, since the game data `.pak` files aren't modified). This avoids showing the raw key name in the menu.

### File 4 — `src/gui/Hud.cpp`

This is the render path. Two things were added:

**a) A helper function** (file-local, just above the draw functions):

```cpp
static Color hudAlpha(Color c) {
    return Color(c.r, c.g, c.b, u8(float(c.a) * config.interface.hudOpacity));
}
```

This scales the alpha channel of any color by the current opacity. It is applied to every `EERIEDrawBitmap` call inside HUD draw functions.

**b) A texture-stage bracket** inside `HudRoot::draw()`:

```cpp
GRenderer->GetTextureStage(0)->setAlphaOp(TextureStage::OpModulate);
// ... all HUD draw calls ...
GRenderer->GetTextureStage(0)->setAlphaOp(TextureStage::OpSelectArg1);
```

This is the critical non-obvious part. See the **GL_REPLACE trap** section below.

### File 5 — `src/gui/hud/HudCommon.cpp`

`HudIconBase::draw()` is the base-class draw for all side icons (backpack, book, purse, etc.). It lives in a separate file but follows the same pattern: include `core/Config.h`, then apply `Color(255, 255, 255, u8(255.f * config.interface.hudOpacity))` to every `EERIEDrawBitmap` call.

---

### The GL_REPLACE trap (why vertex alpha alone isn't enough)

This is the most important thing this patch discovered, and the thing most likely to trip up future modders.

Arx's OpenGL texture stage is initialized with:

```cpp
setTexEnv(GL_TEXTURE_ENV, GL_COMBINE_ALPHA, GL_REPLACE);
// comment in source: "TODO change the AL default to match OpenGL"
```

`GL_REPLACE` means the fragment's final alpha is taken *entirely* from the texture's alpha channel. The vertex color alpha is **discarded**. This is a known deviation from the OpenGL default (which is `GL_MODULATE`).

Consequence: if you only change the alpha field of the `Color` you pass to `EERIEDrawBitmap`, nothing happens visually. The alpha you wrote is thrown away before the blend equation ever sees it.

The fix is to switch the stage to `GL_MODULATE` for HUD draws:

```
out_alpha = texture_alpha × vertex_alpha
```

With `vertex_alpha = hud_opacity × 255`, this gives:

```
out_alpha = texture_alpha × hud_opacity
```

At `hud_opacity = 1.0` (default), `vertex_alpha = 255`, so `out_alpha = texture_alpha` — identical to vanilla behavior.
At `hud_opacity = 0.5`, every HUD element draws at half its normal alpha.

The `OpModulate` switch is scoped entirely to `HudRoot::draw()` and restored to `OpSelectArg1` at the end, so nothing outside the HUD is affected.

This same fix applies to **any future HUD alpha effect**. You do not need to touch the texture stage code again — just make sure your draw calls happen inside the `HudRoot::draw()` block where `OpModulate` is already active.

---

### Scope of the fade

| Element | Fades? | Why |
|---|---|---|
| Health / mana orbs | ✅ | drawn by `HudRoot::draw()` |
| Side icons (backpack, book, etc.) | ✅ | `HudIconBase::draw()` called from `HudRoot::draw()` |
| Stealth gauge | ✅ | drawn by `HudRoot::draw()` |
| Spell slots (precast / active) | ✅ | drawn by `HudRoot::draw()` |
| Screen-edge arrows | ✅ | drawn by `HudRoot::draw()` |
| Damaged-equipment indicators | ✅ | drawn by `HudRoot::draw()` |
| Memorized runes | ✅ | drawn by `HudRoot::draw()` |
| Hit-strength gauge | ✅ | drawn by `HudRoot::draw()` |
| Quick-save flash icon | ✅ | drawn by `HudRoot::draw()` |
| Gold number (purse) | ✅ | drawn via `ARX_INTERFACE_DrawNumber` inside `HudRoot::draw()` |
| Main menus | ❌ | separate render path |
| Loading screens | ❌ | separate render path |
| Cinematics | ❌ | separate render path |
| Player book interior | ❌ | `g_playerBook.manage()` uses its own render state stack |
| Inventory grids | ❌ | internal draw calls do not go through `hudAlpha()` |

---

## In-depth change log

See [`CHANGES.md`](CHANGES.md) for a line-by-line walkthrough of every modification.

---

## Files in this repo

| File | Purpose |
|---|---|
| `arx-hud-opacity.patch` | Annotated unified diff — apply with `patch -p1` from inside `src/` |
| `apply-hud-opacity.sh` | Self-documenting shell script that applies the same changes |
| `CHANGES.md` | Line-by-line explanation of every modification |
| `src/` | The five modified source files |

---

## Authorship

**Author / maintainer: Dantalian** — specification, code review, validation against engine internals, and correction of issues found in review.

**Implementation:** generated by [Claude Sonnet 4.6](https://www.anthropic.com/claude) via [Claude Code](https://claude.ai/code) under direction, then reviewed and corrected before release. Review caught and fixed a texture-stage alpha bug (`GL_REPLACE` discarding vertex alpha), a broken patch-apply path, and incorrect build dependencies; additive-blend fade behavior was verified against the upstream `blendAdditive()` definition.

---

## License

The patch follows the same license as Arx Libertatis: **GPLv3+**. See `COPYING` and `LICENSE` in the upstream repo.
