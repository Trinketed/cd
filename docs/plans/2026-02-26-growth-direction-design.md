# Growth Direction (Anchor Corner) Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a per-team "Growth Direction" setting that controls which corner of a cooldown bar is pinned, so grids expand away from the user's UI when class changes cause size differences.

**Architecture:** New `growthDirection` field in party/enemy settings (TOPLEFT/TOPRIGHT/BOTTOMLEFT/BOTTOMRIGHT). Display.lua's `PositionBar` uses this for anchor-to-frames, fallback, and saved positions. `SaveBarPosition` normalizes drag results to the chosen corner. Options.lua adds a dropdown in each team's POSITIONING section.

**Tech Stack:** WoW Lua 5.1, Frame API (SetPoint/GetPoint/GetLeft/GetRight/GetTop/GetBottom), UIDropDownMenu templates

---

### Task 1: Add growthDirection to DEFAULTS

**Files:**
- Modify: `Core.lua:83-98`

**Step 1: Add the new field to party and enemy defaults**

In `Core.lua`, add `growthDirection = "TOPLEFT"` to both the `party` and `enemy` tables inside `DEFAULTS`:

```lua
party = {
    enabled         = true,
    iconSize        = 36,
    growthDirection = "TOPLEFT",
    anchorToFrames  = true,
    offsetX         = 5,
    offsetY         = 0,
    positions       = {},
},
enemy = {
    enabled         = true,
    iconSize        = 36,
    growthDirection = "TOPLEFT",
    anchorX         = 500,
    anchorY         = -200,
    spacing         = 40,
    positions       = {},
},
```

The existing `MergeDefaults` pattern will backfill this for users with existing saved variables.

**Step 2: Commit**

```
feat: add growthDirection default to party/enemy settings
```

---

### Task 2: Update PositionBar to use growthDirection

**Files:**
- Modify: `Display.lua:92-127`

**Step 1: Add a helper lookup table above PositionBar**

Add this local table near the top of the Bar Positioning section (after line 91), providing the anchor mappings:

```lua
-- Anchor mapping for growth direction
-- For anchorToFrames: maps growthDirection -> {barPoint, framePoint, xSign, ySign}
local GROWTH_ANCHORS = {
    TOPLEFT     = { bar = "TOPLEFT",     frame = "TOPRIGHT",     xSign =  1, ySign =  1 },
    TOPRIGHT    = { bar = "TOPRIGHT",    frame = "TOPLEFT",      xSign = -1, ySign =  1 },
    BOTTOMLEFT  = { bar = "BOTTOMLEFT",  frame = "BOTTOMRIGHT",  xSign =  1, ySign = -1 },
    BOTTOMRIGHT = { bar = "BOTTOMRIGHT", frame = "BOTTOMLEFT",   xSign = -1, ySign = -1 },
}
```

**Step 2: Rewrite the party branch of PositionBar**

Replace the party positioning block (lines 99-115) with:

```lua
    if info.team == "party" then
        local settings = self.db.party
        local saved = settings.positions[tostring(info.slot)]
        if saved and not settings.anchorToFrames then
            bar:SetPoint(saved.point, UIParent, saved.relPoint, saved.x, saved.y)
        else
            local gd = GROWTH_ANCHORS[settings.growthDirection] or GROWTH_ANCHORS.TOPLEFT
            local parentFrame = _G["PartyMemberFrame" .. info.slot]
            if parentFrame and parentFrame:IsShown() then
                bar:SetPoint(gd.bar, parentFrame, gd.frame,
                    gd.xSign * settings.offsetX, gd.ySign * settings.offsetY)
            else
                -- Fallback position (test mode, frames hidden)
                bar:SetPoint(gd.bar, UIParent, gd.bar,
                    gd.xSign * 20,
                    gd.ySign * (200 + (info.slot - 1) * 50))
            end
        end
```

Key changes:
- Anchor-to-frames uses `gd.bar` / `gd.frame` instead of hardcoded `"LEFT"` / `"RIGHT"`
- Fallback uses `gd.bar` anchored to UIParent's same corner with signed offsets
- Fallback y-offset note: for TOPLEFT/TOPRIGHT the formula produces negative y (downward from top) via `gd.ySign * 200 = 200` but we need it negative from the top edge. Let's fix: for TOP variants we go down from the top of UIParent, for BOTTOM variants we go up from the bottom. Since `gd.bar` is the anchor point on UIParent, and y is relative to that corner, we need: `-gd.ySign * (200 + ...)`. Wait — let me reconsider.

Actually, let's simplify the fallback. For `TOPLEFT` the current code does `SetPoint("TOPLEFT", UIParent, "TOPLEFT", 20, -(200 + ...))`. The `y` is negative because we're going DOWN from the TOP. For `BOTTOMLEFT`, we'd want `SetPoint("BOTTOMLEFT", UIParent, "BOTTOMLEFT", 20, (200 + ...))` — positive y going UP from the BOTTOM. So the sign flip on y is correct as `gd.ySign * -(200 + ...)` which simplifies to `-gd.ySign * (200 + ...)`:

```lua
            else
                -- Fallback position (test mode, frames hidden)
                local yOff = 200 + (info.slot - 1) * 50
                bar:SetPoint(gd.bar, UIParent, gd.bar,
                    gd.xSign * 20, -gd.ySign * yOff)
            end
```

**Step 3: Rewrite the enemy branch of PositionBar**

Replace the enemy positioning block (lines 116-126) with:

```lua
    elseif info.team == "enemy" then
        local settings = self.db.enemy
        local saved = settings.positions[tostring(info.slot)]
        if saved then
            bar:SetPoint(saved.point, UIParent, saved.relPoint, saved.x, saved.y)
        else
            local gd = GROWTH_ANCHORS[settings.growthDirection] or GROWTH_ANCHORS.TOPLEFT
            local yOff = settings.anchorY - (info.slot - 1) * settings.spacing
            bar:SetPoint(gd.bar, UIParent, gd.bar,
                gd.xSign * math.abs(settings.anchorX),
                gd.ySign * math.abs(yOff))
        end
    end
```

Note: for enemy defaults (anchorX=500, anchorY=-200), with TOPLEFT the x=500 (right of left edge), y=-200 (down from top). For TOPRIGHT: x=-500 (left of right edge), y=-200 (down from top). The sign flips handle this. We use `math.abs` so the sign always comes from `gd.xSign/ySign`, making the saved anchorX/anchorY values direction-agnostic.

Actually, this changes behavior for existing users who may have custom anchorX/anchorY values. Let me reconsider. The simplest approach: only apply the growth direction to the anchor point name, and let the user's anchorX/anchorY work as-is for TOPLEFT. For other directions, the bar point changes but we keep the same absolute offsets from that corner:

```lua
    elseif info.team == "enemy" then
        local settings = self.db.enemy
        local saved = settings.positions[tostring(info.slot)]
        if saved then
            bar:SetPoint(saved.point, UIParent, saved.relPoint, saved.x, saved.y)
        else
            local gd = GROWTH_ANCHORS[settings.growthDirection] or GROWTH_ANCHORS.TOPLEFT
            bar:SetPoint(gd.bar, UIParent, gd.bar,
                gd.xSign * settings.anchorX,
                settings.anchorY - (info.slot - 1) * settings.spacing)
        end
    end
```

For TOPLEFT (default): `SetPoint("TOPLEFT", UIParent, "TOPLEFT", 500, -200)` — same as before.
For TOPRIGHT: `SetPoint("TOPRIGHT", UIParent, "TOPRIGHT", -500, -200)` — mirrored to right side.
For BOTTOMLEFT: `SetPoint("BOTTOMLEFT", UIParent, "BOTTOMLEFT", 500, -200)` — wait, y=-200 from bottom-left goes DOWN, which is off-screen. We need the y to flip for bottom variants.

OK, let's use the ySign for the y component too:

```lua
            bar:SetPoint(gd.bar, UIParent, gd.bar,
                gd.xSign * settings.anchorX,
                gd.ySign * settings.anchorY + (gd.ySign * -1) * (info.slot - 1) * settings.spacing)
```

Hmm, that's getting complex. Simplest correct version:

```lua
            local slotOffset = (info.slot - 1) * settings.spacing
            bar:SetPoint(gd.bar, UIParent, gd.bar,
                gd.xSign * settings.anchorX,
                -gd.ySign * (math.abs(settings.anchorY) + slotOffset))
```

For TOPLEFT: `SetPoint("TOPLEFT", UIParent, "TOPLEFT", 500, -(200 + slotOffset))` — down from top. Correct.
For BOTTOMLEFT: `SetPoint("BOTTOMLEFT", UIParent, "BOTTOMLEFT", 500, (200 + slotOffset))` — up from bottom. Correct.
For TOPRIGHT: `SetPoint("TOPRIGHT", UIParent, "TOPRIGHT", -500, -(200 + slotOffset))` — left of right, down from top. Correct.

Good, this works. Uses `math.abs(settings.anchorY)` since default is -200 and we want the magnitude.

**Step 4: Commit**

```
feat: PositionBar uses growthDirection for anchor-to-frames and fallback
```

---

### Task 3: Update SaveBarPosition to normalize to growth direction corner

**Files:**
- Modify: `Display.lua:129-144`

**Step 1: Rewrite SaveBarPosition**

Replace the current `SaveBarPosition` function with a version that normalizes the bar's position to the growth direction corner after dragging:

```lua
function addon:SaveBarPosition(guid)
    local bar = self.bars[guid]
    local info = self.state.trackedPlayers[guid]
    if not bar or not info then return end

    local settings = (info.team == "party") and self.db.party or self.db.enemy
    local anchor = settings.growthDirection or "TOPLEFT"

    -- Get bar's screen-space edges (in UIParent-scale coordinates)
    local s = bar:GetEffectiveScale() / UIParent:GetEffectiveScale()
    local left   = bar:GetLeft()   * s
    local right  = bar:GetRight()  * s
    local top    = bar:GetTop()    * s
    local bottom = bar:GetBottom() * s

    local uiW = UIParent:GetWidth()
    local uiH = UIParent:GetHeight()

    -- Compute offset from UIParent's matching corner to bar's matching corner
    local x, y
    if anchor == "TOPLEFT" then
        x, y = left, top - uiH
    elseif anchor == "TOPRIGHT" then
        x, y = right - uiW, top - uiH
    elseif anchor == "BOTTOMLEFT" then
        x, y = left, bottom
    elseif anchor == "BOTTOMRIGHT" then
        x, y = right - uiW, bottom
    end

    -- Re-anchor bar to the normalized point
    bar:ClearAllPoints()
    bar:SetPoint(anchor, UIParent, anchor, x, y)

    settings.positions[tostring(info.slot)] = {
        point = anchor, relPoint = anchor, x = x, y = y,
    }

    -- When party bar is manually moved, disable auto-anchoring
    if info.team == "party" then
        self.db.party.anchorToFrames = false
    end
end
```

Key behavior: after dragging, the bar is re-anchored at the correct growth direction corner. When the grid later resizes (class change), the bar expands away from that pinned corner.

**Step 2: Commit**

```
feat: SaveBarPosition normalizes to growthDirection corner
```

---

### Task 4: Add Growth Direction dropdown to Party settings tab

**Files:**
- Modify: `Options.lua:1964-1976` (between anchorToFrames checkbox and POSITIONING header)

**Step 1: Add the dropdown**

After the anchorToFrames checkbox (line 1963, `y = y - 38`), insert a growth direction dropdown before the POSITIONING section header. Use the existing `UIDropDownMenu` pattern from the test mode tab:

```lua
    -- Growth Direction dropdown
    local gdLabel = sp:CreateFontString(nil, "OVERLAY")
    gdLabel:SetFont(addon.FONT_BODY, 10, "")
    gdLabel:SetPoint("TOPLEFT", 10, y)
    gdLabel:SetText("Growth Direction")
    gdLabel:SetTextColor(C.textNormal[1], C.textNormal[2], C.textNormal[3])

    local GROWTH_OPTIONS = {
        { value = "TOPLEFT",     label = "Top-Left  \226\134\152" },
        { value = "TOPRIGHT",    label = "Top-Right  \226\134\153" },
        { value = "BOTTOMLEFT",  label = "Bottom-Left  \226\134\151" },
        { value = "BOTTOMRIGHT", label = "Bottom-Right  \226\134\150" },
    }

    local gdDDName = "TrinketedCDPartyGrowthDD"
    local gdDD = CreateFrame("Frame", gdDDName, sp, "UIDropDownMenuTemplate")
    gdDD:SetPoint("TOPLEFT", 100, y + 5)
    UIDropDownMenu_SetWidth(gdDD, 130)
    UIDropDownMenu_Initialize(gdDD, function(self2, level)
        for _, opt in ipairs(GROWTH_OPTIONS) do
            local info = UIDropDownMenu_CreateInfo()
            info.text = opt.label
            info.func = function()
                addon.db.party.growthDirection = opt.value
                UIDropDownMenu_SetText(gdDD, opt.label)
                wipe(addon.db.party.positions)
                addon:RefreshAllBars()
                CloseDropDownMenus()
            end
            UIDropDownMenu_AddButton(info, level)
        end
    end)

    -- Set initial text
    local currentGD = self.db.party.growthDirection or "TOPLEFT"
    for _, opt in ipairs(GROWTH_OPTIONS) do
        if opt.value == currentGD then
            UIDropDownMenu_SetText(gdDD, opt.label)
            break
        end
    end
    y = y - 38
```

Note: The `\226\134\152` etc. are UTF-8 byte sequences for arrow characters (↘ ↙ ↗ ↖). These render correctly with FRIZQT font per the project memory notes on full UTF-8 support.

Arrow mapping: TOPLEFT grows ↘ (`\226\134\152`), TOPRIGHT grows ↙ (`\226\134\153`), BOTTOMLEFT grows ↗ (`\226\134\151`), BOTTOMRIGHT grows ↖ (`\226\134\150`).

**Step 2: Commit**

```
feat: add Growth Direction dropdown to party settings tab
```

---

### Task 5: Add Growth Direction dropdown to Enemy settings tab

**Files:**
- Modify: `Options.lua:2026-2036` (in the POSITIONING section, before the drag hint)

**Step 1: Add the dropdown**

After the POSITIONING section header (line 2027, `y = y - 2`), insert the same dropdown pattern but for enemy settings. Move the `GROWTH_OPTIONS` table to module scope (above the party tab function) so both tabs can share it:

```lua
    -- Growth Direction dropdown
    local gdLabel = sp:CreateFontString(nil, "OVERLAY")
    gdLabel:SetFont(addon.FONT_BODY, 10, "")
    gdLabel:SetPoint("TOPLEFT", 10, y)
    gdLabel:SetText("Growth Direction")
    gdLabel:SetTextColor(C.textNormal[1], C.textNormal[2], C.textNormal[3])

    local gdDDName = "TrinketedCDEnemyGrowthDD"
    local gdDD = CreateFrame("Frame", gdDDName, sp, "UIDropDownMenuTemplate")
    gdDD:SetPoint("TOPLEFT", 100, y + 5)
    UIDropDownMenu_SetWidth(gdDD, 130)
    UIDropDownMenu_Initialize(gdDD, function(self2, level)
        for _, opt in ipairs(GROWTH_OPTIONS) do
            local info = UIDropDownMenu_CreateInfo()
            info.text = opt.label
            info.func = function()
                addon.db.enemy.growthDirection = opt.value
                UIDropDownMenu_SetText(gdDD, opt.label)
                wipe(addon.db.enemy.positions)
                addon:RefreshAllBars()
                CloseDropDownMenus()
            end
            UIDropDownMenu_AddButton(info, level)
        end
    end)

    local currentGD = self.db.enemy.growthDirection or "TOPLEFT"
    for _, opt in ipairs(GROWTH_OPTIONS) do
        if opt.value == currentGD then
            UIDropDownMenu_SetText(gdDD, opt.label)
            break
        end
    end
    y = y - 38
```

**Step 2: Commit**

```
feat: add Growth Direction dropdown to enemy settings tab
```

---

### Task 6: Manual testing in-game

**Step 1: Test default behavior (TOPLEFT)**

1. `/reload` — verify existing behavior is unchanged
2. Open options, confirm "Growth Direction" dropdown shows "Top-Left ↘" for both party and enemy
3. Enter test mode with a warrior in slot 1 — note bar position
4. Switch to a class with more cooldowns — verify bar grows right and down, position stable

**Step 2: Test TOPRIGHT**

1. Set party growth direction to "Top-Right ↙"
2. Positions should wipe and bars re-anchor
3. If anchorToFrames is on: bar should appear to the LEFT of party frames
4. Switch class in test mode — verify the RIGHT edge stays fixed, bar extends left

**Step 3: Test manual drag with TOPRIGHT**

1. Unlock bars, drag a party bar to a custom position
2. Lock bars, switch class in test mode — verify right edge stays pinned
3. Switch back — verify position is stable

**Step 4: Test BOTTOMLEFT and BOTTOMRIGHT**

1. Repeat similar checks for bottom variants
2. Verify bars grow upward instead of downward

**Step 5: Test enemy bars**

1. Set enemy growth direction to TOPRIGHT
2. Verify fallback positions mirror correctly
3. Drag and verify normalization works

**Step 6: Test Reset Positions button**

1. After manual positioning with non-default growth direction
2. Click "Reset Party Positions" — verify bars snap back to anchor-to-frames with current growth direction
