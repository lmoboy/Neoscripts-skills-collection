---
name: neoscripts
description: >
  Use this skill whenever the user wants to write, debug, or understand scripts for NeoScripts,
  a Minecraft mod with a Lua 5.3 runtime. Triggers include: any mention of "neoscripts", "neo script",
  writing Lua for Minecraft mods, imgui in Minecraft, registerClientTick, registerImGuiRenderEvent,
  register2DRenderer, registerWorldRenderer, or any Lua scripting within a Minecraft mod context.
  Also trigger when the user asks about modules like player, http, json, imgui, or packets in a
  Minecraft Lua scripting context. Always consult this skill before generating any NeoScripts Lua code.
---

# NeoScripts Skill

NeoScripts is a Minecraft mod providing a **Lua 5.3 runtime** for client-side scripting. Scripts can render ImGui UIs, draw in the world or HUD, send packets, and interact with the player/game state.

---

## Avoidable patterns

At all costs, avoid adding unnecessary comment like separators for functions, imports, helpers etc, if the code is not properly structured or self explanatory you are doing something wrong and should consult the user for clarification.

---

## Module System

All built-in modules and other scripts are imported via `require`:

```lua
local http   = require("http")
local json   = require("json")
local player = require("player")
local imgui  = require("imgui")
-- other scripts in the same folder:
local utils  = require("utils")  -- loads utils.lua from the same directory as the script
local newlib  = require("newlib")  -- loads newlib.lua from the libs directory, the mod allows this behaviour for simplicity
```

Some modules are submodules of other modules, e.g. `network` is a submodule of `player` same as `input` or `inventory` which would be accessed as follows.

```lua
local player = require("player")
local network = player.network 
```

---

## Property & Method Access

Properties are accessed via dot-chaining. If a property exposes functions, call them directly:

```lua
-- property chain
local box = player.entity.box

-- method on a property
local hit = player.entity.box.intersects(otherBox)
```

---

## Event Hooks

### `registerClientTick` — Inputs, Packets, Game Logic

**All input checks and packet sends MUST be inside `registerClientTick`.**
Doing this outside the tick hook causes packet spam and will get the player kicked or game to crash.

```lua
-- assume imports are already done
registerClientTick(function()
    if input.isKeyDown(key.KEY_F) then
        packets.sendCustomPayload("mymod:action", data)
    end
end)
```

### `registerImGuiRenderEvent` — ImGui Windows & Draw List

```lua
registerImGuiRenderEvent(function()
    -- imgui and imgui.dl calls go here
end)
```

### `register2DRenderer` — HUD / Screen-Space Rendering

The hook provides a `ctx` object with 2D drawing methods:

```lua
register2DRenderer(function(ctx)
    ctx.renderLine(x1, y1, x2, y2, color)
    ctx.renderRect(x, y, w, h, color)
    -- etc.
end)
```

### `registerWorldRenderer` — 3D World Rendering

The hook provides a `ctx` object with world-space drawing methods:

```lua
registerWorldRenderer(function(ctx)
    ctx.renderBeacon(x, y, z, color)
    ctx.renderBox(box, color)
    -- etc.
end)
```

### Available context methods for 2D renderer

You can safely assume that when a function argument calls for points it is an array of tuple {x,y} 

- `renderText(x, y, text, red, green, blue, alpha, shadow, scale)` — Draw text on screen
- `renderImage(path, x, y, width, height, u, v, regionWidth, regionHeight)` — Draw an image
- `renderRect(x, y, width, height, red, green, blue, alpha)` — Draw a filled rectangle
- `renderLine(x1, y1, x2, y2, red, green, blue, alpha, thickness)` — Draw a line
- `renderPolygon(points, red, green, blue, alpha)` — Draw a filled polygon
- `renderItemStack(x, y, itemStack, scale)` — Draw an item stack icon
- `getWindowScale()` — Returns `{width, height}` of the window
- `getTextWidth(text)` — Returns the rendered width of a string

### Available context methods for 3D renderer

All `box` parameters in these methods MUST be created via `creator.createBox()` — raw Lua tables with `minX/minY/minZ/maxX/maxY/maxZ` fields will NOT be accepted by the Java backend.

```lua
local creator = require("creator")
local box = creator.createBox(minX, minY, minZ, maxX, maxY, maxZ)
ctx.renderFilled(box, red, green, blue, alpha, throughWalls)
ctx.renderOutline(box, red, green, blue, alpha, lineWidth, throughWalls)
```

You can safely assume that when a function argument calls for points it is an array of truple {x,y,z} 

- `renderFilled(box, red, green, blue, alpha, throughWalls)` — Filled box
- `renderFilledCircle(x, y, z, radius, segments, red, green, blue, alpha, throughWalls)` — Filled circle
- `renderOutline(box, red, green, blue, alpha, lineWidth, throughWalls)` — Outlined box
- `renderOutlineCircle(x, y, z, radius, segments, thickness, red, green, blue, alpha, throughWalls)` — Outlined circle
- `renderCylinder(x, y, z, radius, height, segments, red, green, blue, alpha, throughWalls)` — Cylinder
- `renderSphere(x, y, z, radius, segments, rings, red, green, blue, alpha, throughWalls)` — Sphere
- `renderText(x, y, z, text, scale, red, green, blue, throughWalls)` — 3D text label
- `renderLinesFromPoints(points, red, green, blue, alpha, lineWidth, throughWalls)` — Polyline from point list
- `renderLineFromCursor(x, y, z, red, green, blue, alpha, lineWidth)` — Line from camera to point
- `renderImage(path, x, y, z, width, height, regionWidth, regionHeight, offsetX, offsetY, offsetZ, red, green, blue, alpha, throughWalls)` — 3D image billboard
- `renderBeaconBeam(x, y, z, red, green, blue)` — Beacon beam effect
- `renderQuad(points, red, green, blue, alpha, throughWalls)` — Filled quad from 4 points
- `renderHologramBlock(x, y, z, id)` — Ghost/hologram block
- `renderBlock(x, y, z, id)` — Solid block render

---

## Threading & World Access

**Run `world.getBlock` (and other world/entity queries) on the main/client
thread.** Off-thread they are ~10x slower — not the lookup itself, but the
thread-safety machinery around it:

- Minecraft's world/chunk storage isn't built for concurrent reads. Off-thread
  calls either get marshaled back onto the client thread (blocking until it runs)
  or take the chunk `PalettedContainer` lock and contend with the main thread.
- On the main thread there's no hop and no contention, so it's cheap.

Safe places to query the world (all run on the main thread): `registerClientTick`,
`registerClientTickPre`, `registerWorldRenderer`.

If you need block data from a coroutine / async path, **snapshot the blocks you
need during a tick event and read from your own cached table off-thread** — never
loop `world.getBlock` off the main thread. The same caution applies to any other
heavy world/entity query.

---

## ImGui Rules

### Window / Container Pattern

Every ImGui container (`begin`, `beginChild`, `beginTabBar`, `beginTabItem`, `beginCombo`, etc.) follows the **same pattern**:

```lua
if imgui.begin("Window Title") then
    -- widgets go here
end
imgui.endBegin()   -- ALWAYS called after the if/end, even if begin returned false
```

> ⚠️ **Critical**: `imgui.endXxx()` must **always** appear after the closing `end` of the `if` block — never inside it. Skipping it when `begin` returns false will corrupt ImGui's stack.

Examples:

```lua
-- Tab bar
if imgui.beginTabBar("tabs") then
    if imgui.beginTabItem("Config") then
        imgui.text("Settings here")
        imgui.endTabItem()
    end
end
imgui.endTabBar()

-- Combo box
if imgui.beginCombo("Mode", current) then
    imgui.selectable("Option A")
    imgui.endCombo()
end
```

### ImGui Draw List (`imgui.dl`)

Draw list calls are made through `imgui.dl` inside `registerImGuiRenderEvent`:

```lua
registerImGuiRenderEvent(function()
    imgui.dl.addLine(x1, y1, x2, y2, color, thickness)
    imgui.dl.addRectFilled(x1, y1, x2, y2, color)
    imgui.dl.addText(x, y, color, "text")
end)
```

### Available context methods for imgui.dl

You can safely assume that when a function argument calls for points it is an array of tuples {x,y} 

- `renderText(x, y, text, red?, green?, blue?, alpha?)` — Draw text
- `renderImage(textureID, x, y, width, height, uvMinX?, uvMinY?, uvMaxX?, uvMaxY?)` — Draw image
- `renderImageQuad(textureID, p1, p2, p3, p4, uvMinX?, uvMinY?, uvMaxX?, uvMaxY?, red?, green?, blue?, alpha?)` — Draw image quad
- `renderLine(x1, y1, x2, y2, red?, green?, blue?, alpha?, thickness?)` — Draw line
- `renderFilledRect(x1, y1, x2, y2, red?, green?, blue?, alpha?, rounding?)` — Draw filled rectangle
- `renderFilledRectMultiColor(x1, y1, x2, y2, rUL, gUL, bUL, aUL, rUR, gUR, bUR, aUR, rBR, gBR, bBR, aBR, rBL, gBL, bBL, aBL)` — Multi-color filled rect
- `renderRect(x1, y1, x2, y2, red?, green?, blue?, alpha?, rounding?)` — Draw outlined rectangle
- `renderQuad(p1x, p1y, p2x, p2y, p3x, p3y, p4x, p4y, red?, green?, blue?, alpha?, thickness?)` — Draw quad outline
- `renderFilledQuad(p1x, p1y, p2x, p2y, p3x, p3y, p4x, p4y, red?, green?, blue?, alpha?)` — Draw filled quad
- `renderTriangle(p1x, p1y, p2x, p2y, p3x, p3y, red?, green?, blue?, alpha?, thickness?)` — Draw triangle outline
- `renderFilledTriangle(p1x, p1y, p2x, p2y, p3x, p3y, red?, green?, blue?, alpha?)` — Draw filled triangle
- `renderCircle(cx, cy, radius, red?, green?, blue?, alpha?, numSegments?, thickness?)` — Draw circle outline
- `renderFilledCircle(cx, cy, radius, red?, green?, blue?, alpha?, numSegments?)` — Draw filled circle
- `renderNgon(cx, cy, radius, numSegments, red?, green?, blue?, alpha?, thickness?)` — Draw n-gon outline
- `renderFilledNgon(cx, cy, radius, numSegments, red?, green?, blue?, alpha?)` — Draw filled n-gon
- `renderEllipse(cx, cy, rx, ry, red?, green?, blue?, alpha?, rot?, numSegments?, thickness?)` — Draw ellipse outline
- `renderFilledEllipse(cx, cy, rx, ry, red?, green?, blue?, alpha?, rot?, numSegments?)` — Draw filled ellipse
- `renderBezierCubic(p1x, p1y, p2x, p2y, p3x, p3y, p4x, p4y, red?, green?, blue?, alpha?, thickness?, numSegments?)` — Cubic bezier curve
- `renderBezierQuadratic(p1x, p1y, p2x, p2y, p3x, p3y, red?, green?, blue?, alpha?, thickness?, numSegments?)` — Quadratic bezier curve
- `renderPolyline(points, red?, green?, blue?, alpha?, flags?, thickness?)` — Polyline from point list
- `renderPolygon(points, red?, green?, blue?, alpha?)` — Draw polygon
- `renderFilledConvexPolygon(points, red?, green?, blue?, alpha?)` — Draw filled convex polygon
- `pushClipRect(x1, y1, x2, y2, intersect?)` — Push clipping rectangle
- `pushClipRectFullScreen()` — Push full-screen clip rect
- `popClipRect()` — Pop clipping rectangle
- `pushTextureID(textureID)` — Push texture ID
- `popTextureID()` — Pop texture ID

### ImGui Gotchas (native asserts — these crash the game, not just Lua)

ImGui here is a native (imgui-java) binding. Some failures throw a **native C++
assertion that hard-crashes Minecraft** — a Lua `pcall` **cannot** catch those.
The `begin`/`endBegin` pairing rule above also matters here; additionally wrap
window content in a `pcall` and call `endBegin()` unconditionally after it, so a
Lua error inside can't leave the window open.

**Do NOT use `pushStyleVar` for vec2 style vars.** Passing `WindowPadding`,
`FramePadding`, `ItemSpacing`, `ItemInnerSpacing`, `CellPadding`, etc. asserts
`"Calling PushStyleVar() variant with wrong type!"` and crashes — regardless of
whether you pass `(idx, x, y)` or `(idx, val)`. Avoid `pushStyleVar` unless you
have confirmed a specific scalar var works.

**Do NOT feature-detect methods by indexing.** On the `imgui` object, reading a
**non-existent key throws** ("attempt to index ? (a imgui value) with key '…'")
instead of returning `nil`, so `if imgui.pushItemWidth then` itself errors. Only
call methods you know exist. Notably `pushItemWidth` / `setNextItemWidth` are
**not** available.

**Font scaling (e.g. readable on 4K): there is no runtime font-scale setter.**
Pre-build the font at each size you want in `registerImGuiInitEvent` via
`imgui.createFontObject(path, false, true, sizePx)`, keep them in a table, and
`pushFont` the chosen one each frame (pop it outside the content `pcall` so it
always runs). Widget frames (buttons, checkboxes, sliders) derive their height
from the font, so they scale automatically. Load from a real TTF (e.g.
`C:/Windows/Fonts/segoeui.ttf`); guard with `pcall` since a bad path returns nil.

**Fix label clipping without `pushItemWidth`.** Framed widgets (slider/combo/
input) auto-expand to full width and shove a right-side label off screen. Draw
the label on its own line and give the widget a hidden id instead:

```lua
imgui.text("Yaw")
imgui.sliderInt("##yaw", v, min, max)
```

**Auto-size buttons** with `imgui.button(label, 0, 0)` so the text fits at any
font size — hardcoded pixel sizes clip when the font is enlarged.

---

## Library / Module Scripts

When writing a script intended to be used as a library (`require`d by others), follow this pattern exactly:

```lua
local ModuleName = {}

--- Brief description of what this function does.
--- @param argName argType Description of the argument.
--- @param argName2 argType2 Description of the second argument.
--- @return returnType Description of return value.
function ModuleName.methodName(argName, argName2)
    -- implementation
end

--- Another method.
--- @param value number The value to process.
--- @return boolean Whether the operation succeeded.
function ModuleName.anotherMethod(value)
    -- implementation
end

return ModuleName
```

**Rules for library scripts:**
- All public functions must be assigned to the module table (`ModuleName`).
- Every public function must have **EmmyDoc-style LuaDoc** comments directly above it: description, `@param` for each arg (name + type + description), and `@return` (type + description).
- Private helpers can be local functions not on `ModuleName`.
- The file must end with `return ModuleName`.
- Built-in modules (`imgui`, `player`, `http`, `json`, etc.) are always available — use them freely in libraries. Avoid depending on other user scripts/libraries to prevent "dependency hell".

---

## Checklist Before Generating Code

1. **Inputs / packets** → inside `registerClientTick`?
2. **ImGui** → rendered inside `registerImGuiRenderEvent`?
3. **Images/Fonts** → loaded and saved to a variable beforehand in `registerImGuiInitEvent`?
4. **Every `imgui.beginXxx`** → paired `imgui.endXxx` after the `if/end`?
5. **2D draws** → inside `register2DRenderer(function(ctx) ... end)`?
6. **World draws** → inside `registerWorldRenderer(function(ctx) ... end)`?
7. **Library file** → module table returned, all public functions have EmmyDoc?
8. **Modules** → imported via `require("name")` at the top?
9. **Lua version** → 5.3 compatible (no `<<`, no `//` integer division issues, etc.)?
10. **`box` objects** → always built with `creator.createBox(minX, minY, minZ, maxX, maxY, maxZ)` — raw tables won't work in `renderFilled`/`renderOutline`/`renderBox`

---

## Quick Reference: Common Patterns

### ImGui Widget Return Values

Most interactive widgets return `changed, newValue` — always unpack both:

```lua
local _, enabled   = imgui.checkbox("Enabled", enabled)
local _, value     = imgui.sliderFloat("Speed", value, 0.0, 10.0)
local _, count     = imgui.sliderInt("Count", count, 1, 100)
```

> Never assign the raw call directly (`enabled = imgui.checkbox(...)`) — that captures the `changed` boolean, not the new state.

---

```lua
-- Toggle a boolean with a keybind (key event is fired independently of other events )
registerKeyEvent(function (key, action)
    if action == "Press" and key == 50 then
        var = not var
    end
end)


-- Simple ImGui window with a checkbox
registerImGuiRenderEvent(function()
    if imgui.begin("My Menu") then
        local _, enabled   = imgui.checkbox("Enabled", enabled)
        -- keep in mind buttons return only clicked state (bool)
        if imgui.button("Click Me") then
            print("Clicked!")
        end
    end
    imgui.endBegin()
end)

-- ESP box over an entity (world renderer)
registerWorldRenderer(function(ctx)
    for _, e in ipairs(entities.getAll()) do
        ctx.renderBox(e.box, ...args) -- the rest of the arguments are to be read from lua_types of this specific function
    end
end)
```

### ImGui font and image loading pattern

As per outdated documentation loading images requires to create an image object first and then load it and store the id, the following pattern is recommended:

```lua
local img = imgui.createImageObject()
img.loadImage("path/to/image.png")
local imgId = img.getId() -- <-- store this id and use it when calling any imgui image render functions
```
Common paths people use for images are stored within scripts folder `/scripts/images/pong.png` but it is wrong, the mod itself takes a different path.
`config/neoscripts/scripts/images/pong.png` is correct


See `Neo-Scripts-LSP` or grep search for `lua_types` for additional module-specific notes for functions and their arg / return types if available.
