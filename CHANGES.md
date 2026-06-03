# In-depth change log — HUD opacity patch

This document explains every modification made to the Arx Libertatis 1.2.1 source tree, line by line. It is intended for modders who want to understand the reasoning behind each decision, extend the patch, or port it to a later version.

---

## Overview

The patch touches five files and follows a single design principle: **clone the existing HUD scale pattern**. Arx 1.2 already ships a working HUD *scale* slider with full config/menu/render plumbing. Every new HUD setting should use the same three-location structure.

The one non-obvious addition is a texture stage mode change in the render path — explained in full detail below.

---

## File 1 — `src/core/Config.h`

### What changed

One line added to the `interface` sub-struct of the `Config` class:

```cpp
// BEFORE
float hudScale;
bool  hudScaleInteger;
float bookScale;

// AFTER
float hudScale;
bool  hudScaleInteger;
float hudOpacity;   // ← added
float bookScale;
```

### Why here

`Config.h` declares the in-memory representation of all settings. Every value that needs to persist to `cfg.ini` must have a field in this struct. The field is placed immediately after `hudScaleInteger` to keep all HUD-specific settings grouped together.

### Why `float` and why 0–1

The range 0.0–1.0 is the natural range for an opacity/alpha value and matches the convention used everywhere in the renderer (`Color4<float>`, blend factors, etc.). The slider in the menu maps 0–10 integers to this range by dividing by 10.

---

## File 2 — `src/core/Config.cpp`

This file has three separate locations that all need to be updated for any new config key.

### Change 2a — Default value

```cpp
// BEFORE
hudScale = 0.5f,
bookScale = 1.f,

// AFTER
hudScale = 0.5f,
hudOpacity = 1.f,   // ← added
bookScale = 1.f,
```

**Why `1.0f`:** The default must be fully opaque. If it were anything less, every existing installation would see a different HUD the moment they updated the binary, even without touching the slider. A default of `1.0f` means the patch is a zero-change no-op until the user interacts with it.

The `Default` namespace is used as the fallback value in `Config::init()` when the key is absent from `cfg.ini`. On first run with the patched binary, `hud_opacity` will not exist in the config file and this default will be used.

### Change 2b — Config key string

```cpp
// BEFORE
hudScale    = "hud_scale",
hudScaleInteger = "hud_scale_integer",
bookScale   = "book_scale",

// AFTER
hudScale    = "hud_scale",
hudScaleInteger = "hud_scale_integer",
hudOpacity  = "hud_opacity",   // ← added
bookScale   = "book_scale",
```

This is the string that appears in `cfg.ini` under the `[interface]` section:

```ini
[interface]
hud_scale = 5
hud_scale_integer = 1
hud_opacity = 10       ← written by Config::save()
```

The naming convention is `snake_case`, consistent with every other key in the file.

### Change 2c — Writer (`Config::save()`)

```cpp
// BEFORE
writer.writeKey(Key::hudScale,        interface.hudScale);
writer.writeKey(Key::hudScaleInteger, interface.hudScaleInteger);
writer.writeKey(Key::bookScale,       interface.bookScale);

// AFTER
writer.writeKey(Key::hudScale,        interface.hudScale);
writer.writeKey(Key::hudScaleInteger, interface.hudScaleInteger);
writer.writeKey(Key::hudOpacity,      interface.hudOpacity);   // ← added
writer.writeKey(Key::bookScale,       interface.bookScale);
```

`Config::save()` serialises every field to the config file. If this line is omitted, the setting loads correctly (from the default or from a previously written value) but is never saved — it resets to the default every session.

### Change 2d — Reader (`Config::init()`)

```cpp
// BEFORE
interface.hudScaleInteger = reader.getKey(Section::Interface, Key::hudScaleInteger, Default::hudScaleInteger);
float bookScale = reader.getKey(...);

// AFTER
interface.hudScaleInteger = reader.getKey(Section::Interface, Key::hudScaleInteger, Default::hudScaleInteger);
float hudOpacity = reader.getKey(Section::Interface, Key::hudOpacity, Default::hudOpacity);   // ← added
interface.hudOpacity = glm::clamp(hudOpacity, 0.f, 1.f);                                      // ← added
float bookScale = reader.getKey(...);
```

`reader.getKey()` returns the value from the file, or the third argument (the default) if the key is absent.

`glm::clamp(hudOpacity, 0.f, 1.f)` enforces the valid range. Without this, a hand-edited `cfg.ini` with `hud_opacity = 5` would produce undefined rendering behavior. Every float config value in this file is clamped; this follows the same pattern.

---

## File 3 — `src/gui/MainMenu.cpp`

### Change 3a — Slider widget

The Interface options page (`InterfaceOptionsMenuPage`) builds its widget list in its constructor. Widgets are added top-to-bottom via `addCenter()`. The new slider is inserted between the HUD Scale slider and the HUD Scale Integer checkbox:

```cpp
// New block added after the hudScale slider:
{
    std::string label = getLocalised("system_menus_options_interface_hud_opacity", "HUD Opacity");
    SliderWidget * sld = new SliderWidget(sliderSize(), hFontMenu, label);
    sld->valueChanged = boost::bind(&InterfaceOptionsMenuPage::onChangedHudOpacity, this, arg::_1);
    sld->setValue(int(config.interface.hudOpacity * 10.f));
    addCenter(sld);
}
```

**`getLocalised(..., "HUD Opacity")`:** The two-argument overload of `getLocalised` returns its second argument when the localization key is not found. Since we are not modifying the game data `.pak` files (a hard constraint), the key `system_menus_options_interface_hud_opacity` will never exist in the localization tables and the fallback `"HUD Opacity"` is always used. If the upstream project ever adds an official translation for this key, the fallback would be superseded automatically.

**`sld->setValue(int(config.interface.hudOpacity * 10.f))`:** The `SliderWidget` works in integer steps 0–10. Multiplying the 0–1 float by 10 gives the correct initial position. At the default `1.0f`, this sets the slider to position 10 (full right).

**`boost::bind`:** Arx uses Boost.Bind extensively for widget callbacks. The pattern `boost::bind(&Class::method, this, arg::_1)` is identical to the lambdas used for every other slider on the same page.

### Change 3b — Callback

```cpp
void onChangedHudOpacity(int state) {
    config.interface.hudOpacity = float(state) * 0.1f;
}
```

`state` is the integer slider position (0–10). Multiplying by 0.1 converts it back to the 0–1 float range.

**No recalc call:** Unlike `onChangedHudScale`, which calls `g_hudRoot.recalcScale()` to recompute cached pixel sizes, opacity does not need a recalc. `HudRoot::draw()` reads `config.interface.hudOpacity` directly on every frame, so changes take effect immediately on the next rendered frame.

---

## File 4 — `src/gui/Hud.cpp`

This is the largest and most technically involved change.

### Change 4a — `hudAlpha()` helper function

```cpp
static Color hudAlpha(Color c) {
    return Color(c.r, c.g, c.b, u8(float(c.a) * config.interface.hudOpacity));
}
```

`Color` is `Color4<u8>` — four unsigned bytes, RGBA, range 0–255.

This function takes any color and returns a version with its alpha channel scaled by `hud_opacity`. At opacity `1.0`, `u8(float(255) * 1.0) = 255` — unchanged. At opacity `0.5`, `u8(float(255) * 0.5) = 127`.

It is declared `static` (file-local) because nothing outside `Hud.cpp` needs it.

It reads `config.interface.hudOpacity` directly rather than caching it, because config reads are trivially cheap (a float field access) and caching would require invalidation logic.

**Why a single helper for both blend modes:** Arx HUD elements use two blend modes:
- Standard alpha blend (`render2D()` = `BlendSrcAlpha, BlendInvSrcAlpha`): alpha controls opacity directly
- Additive blend (`render2D().blendAdditive()` = `BlendSrcAlpha, BlendOne`): alpha also participates as the source factor

In both cases, reducing vertex alpha reduces the element's contribution to the final pixel. `hudAlpha()` is therefore correct for both — no separate "additive" variant is needed, provided the texture stage is in `OpModulate` mode (see below).

### Change 4b — The `OpModulate` bracket

```cpp
// At the start of the draw block in HudRoot::draw():
GRenderer->GetTextureStage(0)->setAlphaOp(TextureStage::OpModulate);

// ... all HUD draw calls happen here ...

// At the very end of HudRoot::draw():
GRenderer->GetTextureStage(0)->setAlphaOp(TextureStage::OpSelectArg1);
```

**This is the critical change. Without it, nothing works.**

#### Why vertex alpha is discarded by default

Arx's OpenGL texture stage is initialized in `GLTextureStage::GLTextureStage()`:

```cpp
ops[AlphaOp] = OpSelectArg1;
setTexEnv(GL_TEXTURE_ENV, GL_COMBINE_ALPHA, GL_REPLACE);
// comment: "TODO change the AL default to match OpenGL"
```

`GL_COMBINE_ALPHA = GL_REPLACE` means:

```
fragment_alpha = SOURCE0_ALPHA = GL_TEXTURE (the texture's alpha channel)
```

The vertex color alpha is not SOURCE0 here — it is effectively ignored. No matter what alpha value you embed in the `Color` argument to `EERIEDrawBitmap`, it is thrown away before the blend equation runs.

This is a deliberate (though flagged-as-TODO) divergence from the OpenGL default. It means that scaling vertex alpha — the obvious approach — produces no visible effect.

#### The fix: `OpModulate`

`OpModulate` switches the formula to:

```
fragment_alpha = SOURCE0_ALPHA × SOURCE1_ALPHA
               = texture_alpha × vertex_alpha
```

`SOURCE1_ALPHA` defaults to `GL_PREVIOUS`, which for texture stage 0 equals the primary vertex color alpha.

With `vertex_alpha = hud_opacity × 255`:

```
fragment_alpha = texture_alpha × hud_opacity
```

At `hud_opacity = 1.0`: `fragment_alpha = texture_alpha` — unchanged from vanilla.
At `hud_opacity = 0.5`: `fragment_alpha = texture_alpha × 0.5` — half opacity.

#### Scope control

The `setAlphaOp` calls are placed at the very start and very end of the draw block in `HudRoot::draw()`, bracketing all HUD element draws. The `UseTextureState` RAII object that `HudRoot::draw()` already creates does *not* save/restore the alpha op (it only handles filter and wrap mode), so the explicit restore at the end is necessary.

Nothing outside `HudRoot::draw()` — menus, world geometry, cinematics, loading screens — ever runs between the two `setAlphaOp` calls, so there is no risk of inadvertently fading anything other than the HUD.

### Change 4c — Draw call modifications

Every `EERIEDrawBitmap` (and `EERIEDrawBitmap2DecalY`) call within HUD draw functions has its color argument wrapped in `hudAlpha()`:

```cpp
// BEFORE
EERIEDrawBitmap(m_rect, 0.001f, m_emptyTex, Color::white);

// AFTER
EERIEDrawBitmap(m_rect, 0.001f, m_emptyTex, hudAlpha(Color::white));
```

For colors that already have meaningful RGB (e.g., `Color::gray(m_intensity)` for the hit gauge, `Color::rgb(...)` for the flash), `hudAlpha()` preserves the RGB and only scales the alpha:

```cpp
EERIEDrawBitmap(m_rect, 0.0001f, m_fullTex, hudAlpha(Color::gray(m_intensity)));
//  Color::gray(0.5f) = Color(127, 127, 127, 255)
//  hudAlpha at 0.5   = Color(127, 127, 127, 127)
//  Result: same color, half alpha → half opacity
```

The `QuickSaveIconGui` uses an unusual blend mode (`BlendSrcColor, BlendOne` — a "screen" blend where the source color, not source alpha, is the blend factor). For this element the opacity is applied by scaling the gray value directly:

```cpp
EERIEDrawBitmap(..., Color::gray(alpha * config.interface.hudOpacity));
```

This is the only element where `hudAlpha()` is not used, because the blend factor is RGB-based rather than alpha-based.

---

## File 5 — `src/gui/hud/HudCommon.cpp`

`HudIconBase` is the base class for the six side icons (backpack, book, purse, torch-slot, steal, level-up). Its `draw()` method is in a separate compilation unit.

### Change 5a — Include

```cpp
#include "core/Config.h"   // ← added
```

Required to access `config.interface.hudOpacity`.

### Change 5b — Draw calls

```cpp
// BEFORE
EERIEDrawBitmap(m_rect, 0.001f, m_tex, Color::white);

if(m_isSelected) {
    UseRenderState state(render2D().blendAdditive());
    EERIEDrawBitmap(m_rect, 0.001f, m_tex, Color::white);
}

// AFTER
EERIEDrawBitmap(m_rect, 0.001f, m_tex,
    Color(255, 255, 255, u8(255.f * config.interface.hudOpacity)));

if(m_isSelected) {
    UseRenderState state(render2D().blendAdditive());
    EERIEDrawBitmap(m_rect, 0.001f, m_tex,
        Color(255, 255, 255, u8(255.f * config.interface.hudOpacity)));
}
```

The expression `Color(255, 255, 255, u8(255.f * config.interface.hudOpacity))` is the inline equivalent of `hudAlpha(Color::white)`. The helper function from `Hud.cpp` is not available here because it is `static` (file-local). Rather than adding a shared header for a one-liner, the expression is inlined directly.

The `OpModulate` texture stage mode set in `HudRoot::draw()` is still active when `HudIconBase::draw()` is called (since `HudIconBase::draw()` is called from `HudRoot::draw()`), so the vertex alpha set here does take effect.

---

## What was intentionally not changed

- **Save files** — `SaveGame.cpp` and the save format are not touched. Opacity is a rendering setting with no gameplay significance.
- **Localization `.pak` files** — game data files are not modified. The slider label falls back to the hardcoded English string `"HUD Opacity"`.
- **Player book interior** — `g_playerBook.manage()` is called inside `HudRoot::draw()` but uses its own render state stack and draw calls that do not go through `EERIEDrawBitmap` with the HUD helpers.
- **Cursor and book/cursor scale sliders** — unrelated settings, not touched.
- **Minimap** — the minimap is drawn from within the player book path, not `HudRoot::draw()`, so it is outside scope.

---

## Porting to a future Arx Libertatis version

The three-location config pattern (`Config.h` field → `Config.cpp` default/key/reader/writer → `MainMenu.cpp` slider) is stable across versions. The render path (`Hud.cpp`) is more likely to change as the HUD is refactored.

Key things to verify when rebasing:

1. `GLTextureStage` still uses `GL_REPLACE` as its alpha default (check `GLTextureStage.cpp` constructor). If upstream ever fixes the TODO and switches to `GL_MODULATE` by default, the `setAlphaOp` bracket becomes unnecessary but harmless.
2. `HudRoot::draw()` still exists as the single entry point for all HUD rendering.
3. `HudIconBase::draw()` in `HudCommon.cpp` still handles the base icon draw.
