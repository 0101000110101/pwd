# pwd.MAIN

> A client-side Roblox exploit framework targeting **Da Hood**-style games, built with executor-level hooks for aiming, ESP, and anti-detection.

**⚠️ Disclaimer:** This repository is for **educational and research purposes only**. Using this in any live Roblox game violates the Roblox Terms of Service and will result in account termination. The techniques demonstrated here (metamethod hooking, remote spoofing, module patching) are documented so that **game developers can understand and defend against them**. Do not use this to gain an unfair advantage.

---

## Table of Contents

- [Environment Requirements](#environment-requirements)
- [Feature Overview](#feature-overview)
- [1. Silent Aim — GunHandler Module Hook](#1-silent-aim--gunhandler-module-hook)
- [2. Silent Aim — Mouse Metamethod Hook (HC)](#2-silent-aim--mouse-metamethod-hook-hc)
- [3. Force Hit — Remote Event Spoofing](#3-force-hit--remote-event-spoofing)
- [4. Flamelock — OS Cursor Aim](#4-flamelock--os-cursor-aim)
- [5. Camlock — Smooth Camera Aim](#5-camlock--smooth-camera-aim)
- [6. ESP — Drawing Library Rendering](#6-esp--drawing-library-rendering)
- [7. Hitbox Expander](#7-hitbox-expander)
- [8. Bullet Spread Reduction — `math.random` Hook](#8-bullet-spread-reduction--mathrandom-hook)
- [9. HC Godmode — Animation Freeze](#9-hc-godmode--animation-freeze)
- [10. Anti Fall — Humanoid State Override](#10-anti-fall--humanoid-state-override)
- [11. Delay Changer — Weapon Cooldown Override](#11-delay-changer--weapon-cooldown-override)
- [12. Anti Aim View — Accuracy Flag Scrubbing](#12-anti-aim-view--accuracy-flag-scrubbing)
- [13. Anti Mod — Staff Detection & Self-Kick](#13-anti-mod--staff-detection--self-kick)
- [14. Whitelist System](#14-whitelist-system)
- [15. Speed Master](#15-speed-master)
- [16. Atmosphere / Lighting Overrides](#16-atmosphere--lighting-overrides)
- [17. Utility Functions](#17-utility-functions)
- [Detection Notes for Game Developers](#detection-notes-for-game-developers)

---

## Environment Requirements

This script relies on **executor-only** functions. It will not run in a vanilla LocalScript.

| Function | Purpose |
|---|---|
| `hookmetamethod(game, name, func)` | Intercept Lua metamethods on userdata |
| `hookfunction(target, func)` | Replace any function (including C functions like `math.random`) |
| `checkcaller()` | Determine if current call originates from exploit or game |
| `Drawing.new(type)` | Create 2D overlay primitives (Circle, Line, Square, Text) |
| `mousemoverel(x, y)` | Move the OS cursor programmatically |
| `setfpscap(n)` | Unlock/cap framerate |
| `writefile`, `readfile`, `isfolder`, `makefolder`, `listfiles`, `delfile` | Filesystem access |
| `setclipboard(str)` | Clipboard write |

---

## Feature Overview

| # | Feature | Technique | Server-Authoritative? |
|---|---|---|---|
| 1 | Silent Aim (module) | Patch `GunHandler.getAim` | No — direction spoofed client-side |
| 2 | Silent Aim (HC) | `hookmetamethod` `__index` on `mouse` | No — `mouse.Hit`/`mouse.Target` spoofed |
| 3 | Force Hit | Fire `MainEvent:FireServer("Shoot", ...)` | Yes — fully spoofed packet |
| 4 | Flamelock | `mousemoverel` per frame | N/A — real mouse movement |
| 5 | Camlock | `Camera.CFrame:Lerp` | N/A — camera only |
| 6 | ESP | `Drawing.new` overlay | N/A — client render |
| 7 | Hitbox | Resize `HumanoidRootPart` | Depends on server trust |
| 8 | Spread Reduction | Hook `math.random` | Partial |
| 9 | HC Godmode | Freeze emote animation | Depends on hit registration |
| 10 | Anti Fall | `Humanoid:ChangeState` | Client |
| 11 | Delay Changer | Override `ShootingCooldown` | Weak — server may override |
| 12 | Anti Aim View | Zero `ShotLand` / `Warning` / `LockFlagged` | Server-owned — questionable |
| 13 | Anti Mod | `LocalPlayer:Kick` | Self-destruct |
| 14 | Whitelist | Per-UserID skip logic | N/A |
| 15 | Speed Master | Reapply `WalkSpeed` | Weak |
| 16 | Atmosphere | Client `Lighting` | Client only |

---

## 1. Silent Aim — GunHandler Module Hook

The cleanest form of silent aim on games that require a shared gun module. The game's own code calls `GunHandler.getAim(origin, maxDist)` to compute the bullet direction — we replace that function so it returns a direction toward the target instead of toward the camera.

### Source

```lua
local handler, oldFunc = nil, nil
pcall(function()
    local modules = ReplicatedStorage:FindFirstChild("Modules")
    if modules then
        local gunHandler = modules:FindFirstChild("GunHandler")
        if gunHandler then
            handler = require(gunHandler)
            if handler and handler.getAim then
                oldFunc = handler.getAim
            end
        end
    end
end)

if handler and oldFunc then
    handler.getAim = function(origin, maxDist)
        if not _G.SilentAimEnabled then
            return oldFunc(origin, maxDist)
        end

        -- Optional per-weapon bypass (e.g. Revolver uses a different code path)
        if _G.RevolverBypass then
            local tool = LocalPlayer.Character
                and LocalPlayer.Character:FindFirstChildOfClass("Tool")
            if tool and (tool.Name == "[Revolver]" or tool.Name == "Revolver") then
                return oldFunc(origin, maxDist)
            end
        end

        local targetPos = getClosest()
        if targetPos then
            return (targetPos - origin).Unit,
                   math.min((targetPos - origin).Magnitude, maxDist or 200)
        end

        return oldFunc(origin, maxDist)
    end
end
```

### Target selection

```lua
local function getClosest()
    local mousePos = Vector2.new(mouse.X, mouse.Y)
    local best, bestDist = nil, _G.FOV_RADIUS

    for _, v in pairs(Players:GetPlayers()) do
        if v == LocalPlayer then continue end
        if _G.Whitelist and _G.Whitelist[v.UserId] then continue end

        local char = v.Character
        if not char then continue end

        local targetPos = getTargetPosition(v, char)
        if not targetPos then continue end

        local screenPos, onScreen = cam:WorldToScreenPoint(targetPos)
        if onScreen then
            local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
            if dist < bestDist then
                if _G.WallCheck then
                    local ray = Ray.new(
                        cam.CFrame.Position,
                        (targetPos - cam.CFrame.Position).Unit * 500
                    )
                    local hit = workspace:FindPartOnRayWithIgnoreList(
                        ray, {LocalPlayer.Character, cam}
                    )
                    if hit and hit:IsDescendantOf(char) then
                        bestDist = dist
                        best = targetPos
                    end
                else
                    bestDist = dist
                    best = targetPos
                end
            end
        end
    end
    return best
end
```

### How it works

1. On load, `require(GunHandler)` returns the shared module table. Every weapon script calls `GunHandler.getAim(...)` to get a direction.
2. We cache `oldFunc` (the real implementation) so we can call it as a fallback.
3. Our replacement returns `(targetPos - origin).Unit` — a unit vector pointing from the gun muzzle to the target's hit part — plus the correct distance.
4. The game then constructs a ray from `origin` along that direction. **The camera never moves**, so the user sees no visual change, but the bullet flies to the target.
5. `_G.WallCheck` raycasts against the world to skip targets through walls.
6. `_G.KnockCheck` skips targets whose `BodyEffects.K.O.Value == true`.

### Why it's strong

- No mouse movement → no input-based detection
- Uses the game's own code path → the server receives a structurally valid shoot request
- Only works if the game uses a shared module (many don't after anti-cheat updates)

---

## 2. Silent Aim — Mouse Metamethod Hook (HC)

A more invasive version that works on games that read `mouse.Hit` and `mouse.Target` directly. Uses `hookmetamethod` to intercept `__index` on the `mouse` userdata.

### Source

```lua
local oldMouseIndex_HC = nil

local function enableHCSilentAim(enable)
    if enable then
        if oldMouseIndex_HC then return end

        oldMouseIndex_HC = hookmetamethod(game, "__index", function(self, idx)
            if not checkcaller()
               and _G.HCSilentAimEnabled
               and self == mouse
               and (idx == "Hit" or idx == "Target") then

                local mousePos = Vector2.new(mouse.X, mouse.Y)
                local targetPart, targetChar = nil, nil
                local bestDist = _G.HCFOVRadius

                local HC_HIT_PARTS = {
                    "Head", "HumanoidRootPart", "UpperTorso", "LowerTorso",
                    "LeftUpperArm", "LeftLowerArm", "LeftHand",
                    "RightUpperArm", "RightLowerArm", "RightHand",
                    "LeftUpperLeg", "LeftLowerLeg", "LeftFoot",
                    "RightUpperLeg", "RightLowerLeg", "RightFoot",
                }

                for _, v in pairs(Players:GetPlayers()) do
                    if v == LocalPlayer then continue end
                    local char = v.Character
                    if not char then continue end

                    local hum = char:FindFirstChild("Humanoid")
                    if hum and hum.Health <= 0 then continue end

                    if _G.HCKnockCheck then
                        local be = char:FindFirstChild("BodyEffects")
                        if be and be:FindFirstChild("K.O") and be["K.O"].Value then
                            continue
                        end
                    end

                    if _G.Whitelist and _G.Whitelist[v.UserId] then continue end

                    for _, partName in ipairs(HC_HIT_PARTS) do
                        local part = char:FindFirstChild(partName)
                        if part then
                            local sp, onScreen = cam:WorldToScreenPoint(part.Position)
                            if onScreen then
                                local d = (Vector2.new(sp.X, sp.Y) - mousePos).Magnitude
                                if d < bestDist then
                                    bestDist = d
                                    targetPart = part
                                    targetChar = char
                                end
                            end
                        end
                    end
                end

                if targetPart and targetChar then
                    return (idx == "Hit" and CFrame.new(targetPart.Position)
                            or targetChar:FindFirstChild("HumanoidRootPart"))
                end
            end
            return oldMouseIndex_HC(self, idx)
        end)
    else
        if oldMouseIndex_HC then
            hookmetamethod(game, "__index", oldMouseIndex_HC)
            oldMouseIndex_HC = nil
        end
    end
end
```

### How it works

1. `hookmetamethod(game, "__index", fn)` replaces the `__index` metamethod that fires on `game` (which is what the `mouse` userdata routes through when the game reads `mouse.Hit`).
2. `checkcaller()` returns `false` when the call originates from the game's own code (not from our exploit). We only intercept those calls.
3. When the game reads `mouse.Hit`, we return a `CFrame` pointing at the target. When it reads `mouse.Target`, we return the target's `HumanoidRootPart`.
4. Because the game trusts `mouse.Hit` for its raycast origin, the bullet goes to our fake position.

### Important considerations

- **Extremely invasive**: any other script (or the game) reading `mouse.Hit` for legitimate UI purposes gets the wrong value.
- **`checkcaller()` is essential** — without it, the exploit's own reads of `mouse.Hit` recurse infinitely.
- Some games read `mouse.Hit` via `UserInputService:GetMouseLocation()` instead, which this hook does not affect.

---

## 3. Force Hit — Remote Event Spoofing

Bypasses the gun-handling pipeline entirely. Constructs a fake `"Shoot"` payload and fires it at the server's remote event.

### Source

```lua
local ForceHitAllowedTools = {
    "[DoubleBarrel]", "[Revolver]", "[Shotgun]",
    "[SMG]", "[Silencer]", "[TacticalShotgun]"
}

local function ForceHit_IsValidTarget(pl)
    if not pl or pl == LocalPlayer then return false end
    if not pl.Character then return false end
    local hum = pl.Character:FindFirstChild("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    if _G.KnockCheck and isKnocked(pl) then return false end
    return true
end

local function ForceHit_GetBarrelPosition()
    local char = LocalPlayer.Character
    if not char then return nil end
    local tool = char:FindFirstChildOfClass("Tool")
    if tool then
        local h = tool:FindFirstChild("Handle")
                  or tool:FindFirstChild("Barrel")
                  or tool:FindFirstChild("Muzzle")
        if h and h:IsA("BasePart") then return h.Position end
    end
    local arm = char:FindFirstChild("Right Arm")
                or char:FindFirstChild("RightUpperArm")
    if arm and arm:IsA("BasePart") then return arm.Position end
    return char:GetPivot().Position
end

local function ForceHit_Fire(targetPart)
    if not targetPart then return end
    local impactPos = targetPart.Position
    local hrpPos = LocalPlayer.Character
        and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        and LocalPlayer.Character.HumanoidRootPart.Position
        or Vector3.zero

    ReplicatedStorage.MainEvent:FireServer(unpack({
        "Shoot",
        {
            {
                { Normal = impactPos, Instance = targetPart, Position = impactPos },
                { Normal = impactPos, Instance = targetPart, Position = impactPos },
                { Normal = impactPos, Instance = targetPart, Position = impactPos },
                { Normal = impactPos, Instance = targetPart, Position = impactPos },
                { Normal = impactPos, Instance = targetPart, Position = impactPos }
            },
            {
                { thePart = targetPart, theOffset = Vector3.new(0, 0, 0) },
                { thePart = targetPart, theOffset = Vector3.new(0, 0, 0) },
                { thePart = targetPart, theOffset = Vector3.new(0, 0, 0) },
                { thePart = targetPart, theOffset = Vector3.new(0, 0, 0) },
                { thePart = targetPart, theOffset = Vector3.new(0, 0, 0) }
            },
            hrpPos, hrpPos, workspace:GetServerTimeNow()
        }
    }))

    if _G.ForceHitTracerEnabled then
        local barrelPos = ForceHit_GetBarrelPosition()
        if barrelPos then ForceHit_SpawnTracer(barrelPos, impactPos) end
    end
end
```

### Tracer

```lua
local function ForceHit_SpawnTracer(startPos, endPos)
    if (endPos - startPos).Magnitude < 0.1 then return end
    local beam = Instance.new("Beam")
    local attach0 = Instance.new("Attachment")
    local attach1 = Instance.new("Attachment")
    beam.Segments = 1
    beam.Width0 = 0.1
    beam.Width1 = 0.1
    beam.Color = ColorSequence.new(Color3.fromRGB(255, 200, 0))
    beam.Transparency = NumberSequence.new(0.4)
    beam.FaceCamera = true
    attach0.Position = startPos
    attach1.Position = endPos
    attach0.Parent = workspace.Terrain
    attach1.Parent = workspace.Terrain
    beam.Attachment0 = attach0
    beam.Attachment1 = attach1
    beam.Parent = workspace.Terrain
    task.delay(0.08, function()
        beam:Destroy() attach0:Destroy() attach1:Destroy()
    end)
end
```

### Binding

```lua
CAS:BindAction("NHForceHit", ForceHit_MouseClick, false, Enum.UserInputType.MouseButton1)

-- Full auto loop
RunService.Heartbeat:Connect(function()
    if _G.ForceHitFullAutoEnabled and ForceHitIsHoldingMouse and _G.ForceHitEnabled then
        local now = tick()
        if now - ForceHitLastFireTime < _G.ForceHitFireRate then return end
        ForceHitLastFireTime = now
        -- ... pick target, ForceHit_Fire(part)
    end
end)
```

### How it works

1. Directly fires the game's `ReplicatedStorage.MainEvent` with a `"Shoot"` action.
2. The payload contains **five repeated hit entries** — this is a hardcoded assumption about how many pellets/raycasts the server expects. If the server validates `#hits == #pellets`, this breaks for single-pellet weapons.
3. `hrpPos, hrpPos, workspace:GetServerTimeNow()` provide the shooter position, a comparison position, and a timestamp.
4. The tracer beam is purely cosmetic — it appears between the barrel and the impact point so it *looks* like you shot the target.

### Why this is dangerous (for the cheater)

- Server-side validation of `MainEvent` can trivially reject this (e.g. checking pellet count matches the equipped tool, cooldown, ammo).
- Anti-cheats that log `FireServer` payload structure see the mismatched hit table immediately.

---

## 4. Flamelock — OS Cursor Aim

Moves the actual operating system cursor toward the target's screen position. This is not camera-based and not silent — the mouse genuinely moves.

### Source

```lua
local flameTargetPart = nil

UIS.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if _G.FlamelockEnabled then
        local isTriggered =
            (_G.FlameRightClick and input.UserInputType == Enum.UserInputType.MouseButton2)
            or (not _G.FlameRightClick and input.KeyCode == _G.FlameKey)

        if isTriggered then
            if _G.FlameMode == "Hold" then
                _G.FlameActive = true
            else
                _G.FlameActive = not _G.FlameActive
            end

            if _G.FlameActive then
                local target = getFlameTarget()
                if target then flameTargetPart = target
                else _G.FlameActive = false; flameTargetPart = nil end
            else
                flameTargetPart = nil
            end
        end
    end
end)
```

Per-frame update:

```lua
RunService.RenderStepped:Connect(function()
    if _G.FlamelockEnabled and _G.FlameActive then
        if not flameTargetPart or not flameTargetPart.Parent then
            local target = getFlameTarget()
            if target then flameTargetPart = target
            else _G.FlameActive = false end
        end

        if flameTargetPart and flameTargetPart.Parent then
            local targetPlayer = Players:GetPlayerFromCharacter(flameTargetPart.Parent)
            if targetPlayer and not (_G.Whitelist and _G.Whitelist[targetPlayer.UserId]) then
                -- Prediction: lead target
                local predPos = flameTargetPart.Position
                    + (flameTargetPart.Velocity * _G.FlamePrediction)

                -- Offset: shift aim point in screen-space
                local offsetPos = predPos
                    + (cam.CFrame.RightVector * _G.FlameLeftOffset)
                    + Vector3.new(0, _G.FlameUpOffset, 0)

                local sp, on = cam:WorldToViewportPoint(offsetPos)
                if on then
                    local deltaX = (sp.X - mouse.X) * _G.FlameSmoothness
                    local deltaY = (sp.Y - mouse.Y) * _G.FlameSmoothness
                    mousemoverel(deltaX, deltaY)
                end
            else
                flameTargetPart = nil
                _G.FlameActive = false
            end
        end
    end
end)
```

### How it works

1. `mousemoverel(dx, dy)` moves the OS-level cursor relative to its current position — this is a real input event and indistinguishable from a human moving the mouse to the game's input system.
2. `(sp.X - mouse.X) * _G.FlameSmoothness` produces a lerp factor. `FlameSmoothness = 0` → no movement; `= 1` → instant snap.
3. `FlamePrediction` multiplies `Velocity`, effectively leading the target — useful for strafing players.
4. `FlameLeftOffset` / `FlameUpOffset` shift the aim point in the camera's right vector and world Y — this lets the user aim at a body part offset without changing `FlameHitPart`.

### Why this is stealthier than silent aim

- Real cursor movement → passes "human input" heuristics.
- Movement is smooth and continuous → no frame-perfect snaps.
- Doesn't touch any game-owned state.

### Drawbacks

- Visible to the user (the cursor visibly moves).
- Slower than silent aim — target must be on screen.
- Can be detected by checking `UserInputService:GetMouseLocation()` vs `GetMouseDelta()` anomaly (moving without physical input).

---

## 5. Camlock — Smooth Camera Aim

Smoothly rotates the local camera toward a target without moving the mouse.

### Source

```lua
local Camlock = { Target = nil, Active = false, Connection = nil }

local function GetCamlockHitPosition(Target)
    if not Target or not Target.Character then return nil end
    local Character = Target.Character
    local Humanoid = Character:FindFirstChild("Humanoid")
    if not Humanoid then return nil end

    local NearestPart = getClosestPartToMouse(Character)
    if not NearestPart then return nil end

    local HitPosition
    if _G.CamlockHitPart == "Closest Point" then
        if _G.CamlockClosestPointMode == "Default" then
            HitPosition = GetClosestPointOnPart(NearestPart, _G.CamlockClosestPointScale)
        else
            HitPosition = GetClosestPointOnPartBasic(NearestPart)
        end
    elseif _G.CamlockHitPart == "Closest Part" then
        HitPosition = NearestPart.Position
    else
        local part = Character:FindFirstChild(_G.CamlockHitPart)
        HitPosition = part and part.Position
    end

    if not HitPosition then return nil end

    if _G.CamlockPredictionEnabled then
        local RootPart = Character:FindFirstChild("HumanoidRootPart")
        if RootPart then
            local Velocity = RootPart.Velocity
            local PredictionVector = Vector3.new(
                _G.CamlockPredictionX, _G.CamlockPredictionY, _G.CamlockPredictionZ
            )
            HitPosition = HitPosition + Velocity * PredictionVector
        end
    end
    return HitPosition
end

local function UpdateCamlock()
    if not _G.CamlockEnabled then Camlock.Active = false; Camlock.Target = nil; return end
    if not Camlock.Active then return end
    if not Camlock.Target or not Camlock.Target.Character then
        Camlock.Active = false; return
    end

    local Character = Camlock.Target.Character
    if not Character:FindFirstChild("HumanoidRootPart") then Camlock.Active = false; return end

    -- Conditions
    if _G.CamlockConditionsForceField and Character:FindFirstChild("Forcefield") then return end
    if _G.CamlockConditionsKnocked and IsKnocked(Character) then return end
    if _G.CamlockConditionsSelfKnocked and IsKnocked(LocalPlayer.Character) then return end
    if _G.CamlockConditionsCarried and IsGrabbed(Camlock.Target) then return end

    local HitPosition = GetCamlockHitPosition(Camlock.Target)
    if not HitPosition then return end

    -- Dynamic pull strength based on target speed
    local Smoothing = _G.CamlockSmoothness
    if _G.CamlockPullStrengthEnabled then
        local RootPart = Character:FindFirstChild("HumanoidRootPart")
        if RootPart then
            local VelocityMagnitude = RootPart.Velocity.Magnitude
            if VelocityMagnitude > 15 then
                Smoothing = _G.CamlockPullStrengthMoveValue
            else
                Smoothing = _G.CamlockPullStrengthBaseValue
            end
        end
    end

    local EasedSmoothing = TweenService:GetValue(
        Smoothing,
        Enum.EasingStyle[_G.CamlockEasingStyle],
        Enum.EasingDirection[_G.CamlockEasingDirection]
    )

    cam.CFrame = cam.CFrame:Lerp(CFrame.new(cam.CFrame.Position, HitPosition), EasedSmoothing)
end
```

### `GetClosestPointOnPart` — precise hit point

```lua
local function GetClosestPointOnPart(Part, Scale)
    local PartCFrame = Part.CFrame
    local PartSize = Part.Size
    local PartSizeTransformed = PartSize * (Scale / 2)
    local MousePosition = UIS:GetMouseLocation()
    local CurrentCamera = Workspace.CurrentCamera
    local MouseRay = CurrentCamera:ViewportPointToRay(MousePosition.X, MousePosition.Y)
    local Transformed = PartCFrame:PointToObjectSpace(
        MouseRay.Origin + (MouseRay.Direction * MouseRay.Direction:Dot(PartCFrame.Position - MouseRay.Origin))
    )
    if mouse.Target == Part then
        return Vector3.new(mouse.Hit.X, mouse.Hit.Y, mouse.Hit.Z)
    end
    return PartCFrame * Vector3.new(
        math.clamp(Transformed.X, -PartSizeTransformed.X, PartSizeTransformed.X),
        math.clamp(Transformed.Y, -PartSizeTransformed.Y, PartSizeTransformed.Y),
        math.clamp(Transformed.Z, -PartSizeTransformed.Z, PartSizeTransformed.Z)
    )
end
```

### How it works

1. `GetBestCamlockTarget()` iterates players, projects `HumanoidRootPart` to screen, filters by FOV radius and conditions, returns the closest.
2. Each frame, `cam.CFrame = cam.CFrame:Lerp(CFrame.new(cam.CFrame.Position, HitPosition), easedSmoothing)` rotates the camera to look at the target while keeping position fixed.
3. `TweenService:GetValue(alpha, style, direction)` converts a linear `0–1` smoothness into an eased value — enabling `Back`, `Elastic`, `Bounce` style snaps.
4. `ClosestPoint` mode raycasts from the mouse and clamps the hit point onto the part's surface, so aim feels natural (you aim at the part you're looking at).
5. `PullStrength` swaps between two smoothing values depending on whether the target is moving fast — tighter aim on strafers, looser on stationary targets.

---

## 6. ESP — Drawing Library Rendering

Renders 2D overlay primitives for each player. This is pure client rendering — no game state is modified.

### Source

```lua
local espObjects = {}

local function createESP(plr)
    if espObjects[plr] then return end

    local box = Drawing.new("Square")
    box.Thickness = 1; box.Filled = false; box.Color = _G.ESP_Color; box.Visible = false

    local name = Drawing.new("Text")
    name.Size = 13; name.Center = true; name.Outline = true; name.Color = _G.ESP_Color; name.Visible = false

    local health = Drawing.new("Text")
    health.Size = 13; health.Center = false; health.Outline = true
    health.Color = Color3.fromRGB(50, 255, 50); health.Visible = false

    local distance = Drawing.new("Text")
    distance.Size = 12; distance.Center = true; distance.Outline = true
    distance.Color = Color3.fromRGB(200, 200, 200); distance.Visible = false

    local tracer = Drawing.new("Line")
    tracer.Thickness = 1; tracer.Color = _G.ESP_Color; tracer.Visible = false

    local skeleton = {}

    espObjects[plr] = {
        Box = box, Name = name, Health = health,
        Distance = distance, Tracer = tracer, Skeleton = skeleton
    }
end
```

### Per-frame update (abbreviated)

```lua
RunService.RenderStepped:Connect(function()
    for plr, objs in pairs(espObjects) do
        local isWhitelisted = _G.Whitelist and _G.Whitelist[plr.UserId] or false

        if _G.ESP_Enabled and plr.Character
           and plr.Character:FindFirstChild("HumanoidRootPart")
           and not isWhitelisted then

            local char = plr.Character
            local hrp  = char.HumanoidRootPart
            local hum  = char:FindFirstChild("Humanoid")
            local rootPos, onScreen = cam:WorldToViewportPoint(hrp.Position)

            if onScreen and hum and hum.Health > 0 then
                local head = char:FindFirstChild("Head") or hrp
                local headPos = cam:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
                local legPos  = cam:WorldToViewportPoint(hrp.Position - Vector3.new(0, 3, 0))

                local boxHeight = math.abs(headPos.Y - legPos.Y)
                local topLeft = Vector2.new(
                    rootPos.X - (boxHeight / 2) / 2,
                    rootPos.Y - boxHeight / 2
                )

                if _G.ESP_Boxes then
                    objs.Box.Size = Vector2.new(boxHeight / 2, boxHeight)
                    objs.Box.Position = topLeft
                    objs.Box.Color = _G.ESP_Color
                    objs.Box.Visible = true
                else objs.Box.Visible = false end

                if _G.ESP_Names then
                    objs.Name.Position = Vector2.new(rootPos.X, topLeft.Y - 16)
                    objs.Name.Text = plr.Name
                    objs.Name.Color = _G.ESP_Color
                    objs.Name.Visible = true
                else objs.Name.Visible = false end

                if _G.ESP_Health then
                    local hp = hum.Health / hum.MaxHealth
                    objs.Health.Position = Vector2.new(topLeft.X - 24, topLeft.Y)
                    objs.Health.Text = tostring(math.floor(hp * 100)) .. "%"
                    objs.Health.Color = hp > 0.5 and Color3.fromRGB(50, 255, 50)
                                     or (hp > 0.25 and Color3.fromRGB(255, 255, 0)
                                     or Color3.fromRGB(255, 50, 50))
                    objs.Health.Visible = true
                else objs.Health.Visible = false end

                if _G.ESP_Distance then
                    local dist = math.floor((LocalPlayer.Character.HumanoidRootPart.Position - hrp.Position).Magnitude)
                    objs.Distance.Position = Vector2.new(rootPos.X, topLeft.Y + boxHeight + 4)
                    objs.Distance.Text = tostring(dist) .. "m"
                    objs.Distance.Color = _G.ESP_Color
                    objs.Distance.Visible = true
                else objs.Distance.Visible = false end

                if _G.ESP_Tracer then
                    objs.Tracer.From = Vector2.new(cam.ViewportSize.X / 2, cam.ViewportSize.Y)
                    objs.Tracer.To   = Vector2.new(rootPos.X, rootPos.Y)
                    objs.Tracer.Color = _G.ESP_Color
                    objs.Tracer.Visible = true
                else objs.Tracer.Visible = false end

                if _G.ESP_Skeleton then
                    for _, conn in ipairs(boneConnections) do
                        local p1 = char:FindFirstChild(conn[1])
                        local p2 = char:FindFirstChild(conn[2])
                        if p1 and p2 then
                            local a, onA = cam:WorldToViewportPoint(p1.Position)
                            local b, onB = cam:WorldToViewportPoint(p2.Position)
                            if onA and onB then
                                local key = conn[1] .. conn[2]
                                if not objs.Skeleton[key] then
                                    objs.Skeleton[key] = Drawing.new("Line")
                                    objs.Skeleton[key].Thickness = 1.5
                                    objs.Skeleton[key].Transparency = 0.6
                                end
                                local line = objs.Skeleton[key]
                                line.From = Vector2.new(a.X, a.Y)
                                line.To   = Vector2.new(b.X, b.Y)
                                line.Color = _G.ESP_Color
                                line.Visible = true
                            end
                        end
                    end
                end
            else
                -- hide everything
            end
        end
    end
end)
```

### Skeleton bone pairs

```lua
local boneConnections = {
    {"Head", "UpperTorso"}, {"UpperTorso", "LowerTorso"},
    {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"},
    {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"},
    {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"},
    {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}
}
```

### FOV circle

```lua
local fovCircle = Drawing.new("Circle")
fovCircle.Thickness = 1
fovCircle.NumSides  = 60
fovCircle.Radius    = _G.FOV_RADIUS
fovCircle.Filled    = false
fovCircle.Color     = ColorPresets[_G.CurrentTheme].AccentColor
fovCircle.Visible   = false
```

### How it works

1. `Drawing.new` primitives are drawn directly into the game's viewport by the executor's rendering layer. They are not `Instance`s and do not replicate — completely undetectable by `:GetChildren()` walks.
2. The box is sized by projecting head and feet separately and using the pixel distance as height.
3. `objs.Skeleton[key]` creates a `Line` per bone pair on first use, then reuses it.
4. FOV circle is used purely as a visual indicator for where the aimbot will pick a target.

### Cleanup

```lua
Players.PlayerRemoving:Connect(function(p)
    if espObjects[p] then
        espObjects[p].Box:Remove()
        espObjects[p].Name:Remove()
        espObjects[p].Health:Remove()
        espObjects[p].Distance:Remove()
        espObjects[p].Tracer:Remove()
        for _, v in pairs(espObjects[p].Skeleton) do v:Remove() end
        espObjects[p] = nil
    end
end)
```

`Drawing` primitives **must** be explicitly `.Remove()`'d — they leak memory and stay rendered forever otherwise.

---

## 7. Hitbox Expander

Enlarges the target's `HumanoidRootPart` client-side. On games that trust the client's `HumanoidRootPart` position/size for hit validation, this makes shots connect from further away.

### Source

```lua
local function UpdateHitboxes()
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and not _G.Whitelist[plr.UserId] and plr.Character then
            local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                if _G.HitboxEnabled then
                    hrp.Size = Vector3.new(_G.HitboxSize, _G.HitboxSize, _G.HitboxSize)
                    hrp.Transparency = 1 - _G.HitboxTransparency
                    hrp.Color = Color3.fromRGB(145, 210, 240)
                    hrp.Material = Enum.Material.Neon
                    hrp.CanCollide = false
                else
                    hrp.Size = Vector3.new(2, 2, 1)
                    hrp.Transparency = 1
                end
            end
        end
    end
end

-- Refresh loop (server may reset on physics ticks)
task.spawn(function()
    while task.wait(0.5) do
        if _G.HitboxEnabled then UpdateHitboxes() end
    end
end)

Players.PlayerAdded:Connect(function(plr)
    if plr ~= LocalPlayer then
        plr.CharacterAdded:Connect(function()
            task.wait(0.5)
            UpdateHitboxes()
        end)
    end
end)
```

### How it works

1. Directly writes to `hrp.Size` on other clients' characters. In Roblox, **client-side part changes to other players' characters do not replicate to the server** — but they *do* affect local raycasts and any hit detection the game performs client-side.
2. Games that do their raycast on the client (silent aim targets) and send the hit part to the server are vulnerable.
3. Games that validate on the server using server-side part sizes are not.
4. The 0.5s refresh loop re-applies the size, because physics or the game's own scripts may reset it.
5. Restoring on disable is done both here and in the UI toggle to avoid leaving a visual artifact.

---

## 8. Bullet Spread Reduction — `math.random` Hook

Many Roblox shooters compute bullet spread via `math.random(-0.05, 0.05)` and add it to the direction vector. Hooking `math.random` lets us shrink that spread.

### Source

```lua
local BulletSpreadSettings = { Enabled = true }

local _0x9ba38e
_0x9ba38e = hookfunction(math.random, function(...)
    local args = {...}
    if checkcaller() then return _0x9ba38e(...) end

    if (#args == 0)
       or (args[1] == -0.05 and args[2] == 0.05)
       or (args[1] == -0.1)
       or (args[1] == -0.05) then
        if BulletSpreadSettings.Enabled then
            return _0x9ba38e(...) * (_G.BulletSpreadAmount / 100)
        end
    end
    return _0x9ba38e(...)
end)
```

### How it works

1. `hookfunction(math.random, fn)` replaces the C-level `math.random` used by the game.
2. `checkcaller()` ensures our own calls to `math.random` are unaffected.
3. The `args` heuristic matches the common patterns game code uses when computing spread — `math.random(-0.05, 0.05)` is the classic "small offset" call.
4. Multiplying by `_G.BulletSpreadAmount / 100` scales the spread. `100` = normal, `0` = no spread (perfect accuracy).
5. Because the game uses `math.random` for many things (particles, idle animations), only calls matching the spread heuristic are affected.

### Caveat

The `args[1] == -0.1` and `args[1] == -0.05` checks are **single-argument** calls, meaning `math.random(-0.1)` — which is actually invalid Lua (should be a positive integer). These branches likely never fire in practice, but exist as defensive measures.

---

## 9. HC Godmode — Animation Freeze

Freezes a specific emote animation at a keyframe that tucks the character's limbs inwards. Games that hit-test against body parts may fail to register damage.

### Source

```lua
local HCGodmode_Active = false
local HCGodmode_Track = nil
local HCGodmode_Heartbeat = nil
local HCGodmode_AnimConn = nil
local HCGodmode_EmoteID = "rbxassetid://70883871260184"
local HCGodmode_FreezeTime = 0.1265

local function HCGodmode_Cleanup()
    if HCGodmode_Track then HCGodmode_Track:Stop() HCGodmode_Track:Destroy() HCGodmode_Track = nil end
    if HCGodmode_Heartbeat then HCGodmode_Heartbeat:Disconnect() HCGodmode_Heartbeat = nil end
    if HCGodmode_AnimConn then HCGodmode_AnimConn:Disconnect() HCGodmode_AnimConn = nil end
end

local function HCGodmode_GetHumanoid()
    local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    return char:WaitForChild("Humanoid")
end

local function HCGodmode_Animate()
    if not HCGodmode_Active then return end
    local hum = HCGodmode_GetHumanoid()
    if not hum then return end

    HCGodmode_Cleanup()
    local anim = Instance.new("Animation")
    anim.AnimationId = HCGodmode_EmoteID

    HCGodmode_Track = hum:LoadAnimation(anim)
    HCGodmode_Track:Play(0, 1, 1)

    HCGodmode_Heartbeat = RunService.Heartbeat:Connect(function()
        if HCGodmode_Track and HCGodmode_Active then
            HCGodmode_Track.TimePosition = HCGodmode_FreezeTime
            HCGodmode_Track:AdjustSpeed(0)
        end
    end)

    HCGodmode_AnimConn = hum.AnimationPlayed:Connect(function(newtrack)
        if HCGodmode_Active and HCGodmode_Track and newtrack ~= HCGodmode_Track then
            task.delay(0.02 + math.random() * 0.03, HCGodmode_Animate)
        end
    end)
end

local function HCGodmode_Stop()
    HCGodmode_Cleanup()
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.25)
    if HCGodmode_Active then HCGodmode_Animate() end
end)
```

### How it works

1. Loads an emote animation that has a specific pose at ~0.1265s.
2. On every `Heartbeat`, sets `TimePosition` back to that frame and `AdjustSpeed(0)` — the character is stuck in that pose.
3. Emotes in Roblox typically disable the default walk/idle animation while playing, so the character's limbs remain in the frozen pose even while moving.
4. If the game plays another animation (e.g. a tool equip), we re-apply our emote after a short randomized delay.
5. `_G.HCGodmodeEnabled` toggles `HCGodmode_Active` and calls `HCGodmode_Animate`/`HCGodmode_Stop`.

### Why it might work

- The pose folds the character so the actual limb parts overlap with the torso — bullets aimed at the head/torso miss because those parts are physically somewhere else.
- Some hit registration systems use the `Head` part's position, which the emote moves into the torso.

### Why it might not

- Server-side hit verification uses server-side part positions, which lag behind the animation state.
- Modern Da Hood clones may force-stop emotes during combat.

---

## 10. Anti Fall — Humanoid State Override

Prevents the `FallingDown` state (knockdown/ragdoll animation) by immediately transitioning to `GettingUp`.

### Source

```lua
RunService.Heartbeat:Connect(function()
    if _G.AntiFallEnabled and LocalPlayer.Character then
        local hum = LocalPlayer.Character:FindFirstChild("Humanoid")
        if hum and hum.Health > 1 and hum:GetState() == Enum.HumanoidStateType.FallingDown then
            hum:ChangeState("GettingUp")
        end
    end
end)
```

### How it works

1. Every heartbeat, checks if the humanoid is in `FallingDown`.
2. `ChangeState("GettingUp")` forces the humanoid to start standing up.
3. `hum.Health > 1` avoids fighting the death state.
4. This does **not** prevent the server from applying the knockdown (server can force state changes back), but it reduces downtime significantly.

---

## 11. Delay Changer — Weapon Cooldown Override

Writes to weapon cooldown values (typically `ShootingCooldown` and `ToleranceCooldown` under each `Tool`).

### Source

```lua
local function applyCustomDelay(v)
    if not _G.DelayChangerEnabled then return end
    if (v.Name == "ShootingCooldown" or v.Name == "ToleranceCooldown")
       and v:IsA("ValueBase") then
        local tool = v:FindFirstAncestorOfClass("Tool")
        local delayValue = _G.DelayChangerOthers
        if tool then
            if tool.Name == "[Revolver]" then
                delayValue = _G.DelayChangerRevolver
            elseif tool.Name == "[Double-Barrel SG]" then
                delayValue = _G.DelayChangerDoubleBarrel
            elseif tool.Name == "[TacticalShotgun]" then
                delayValue = _G.DelayChangerTacticalShotgun
            end
        end

        v.Value = delayValue
        v:GetPropertyChangedSignal("Value"):Connect(function()
            if v.Value ~= delayValue then v.Value = delayValue end
        end)
    end
end

for _, v in ipairs(game:GetDescendants()) do applyCustomDelay(v) end
game.DescendantAdded:Connect(applyCustomDelay)
```

### How it works

1. Scans the entire game tree for `ShootingCooldown` / `ToleranceCooldown` values.
2. Sets them to the user's configured per-weapon value.
3. If the game resets the value, `GetPropertyChangedSignal` sets it back.

### Caveat

If the server owns these values (`ValueBase` under a `Tool` is usually client-owned for cooldowns, but some games replicate), the change is visual only. The hint "Respawn to apply" suggests the writer observed it being overridden.

---

## 12. Anti Aim View — Accuracy Flag Scrubbing

The game tracks `ShotLand` (shots that hit), `ShotTotal` (shots fired), `Warning`, and `LockFlagged` inside `LocalPlayer.DataFolder`. High accuracy (`ShotLand/ShotTotal ≈ 1`) is a signal of aimbotting.

### Source

```lua
local AccuracyTarget = 0

local function toggleAntiAimView(enable)
    for _, conn in ipairs(antiAimConnections) do
        pcall(function() conn:Disconnect() end)
    end
    antiAimConnections = {}

    if not enable then return end

    local dataFolder = LocalPlayer:FindFirstChild("DataFolder")
        or LocalPlayer:WaitForChild("DataFolder", 5)
    if not dataFolder then return end

    local shotland   = dataFolder:FindFirstChild("ShotLand")
    local shottotal  = dataFolder:FindFirstChild("ShotTotal")
    local warning    = dataFolder:FindFirstChild("Warning")
    local lockflagged = dataFolder:FindFirstChild("LockFlagged")

    local function safeConnect(obj, callback)
        if obj then
            local conn = obj:GetPropertyChangedSignal("Value"):Connect(callback)
            table.insert(antiAimConnections, conn)
        end
    end

    safeConnect(shottotal, function()
        if shottotal and shotland then
            local total = shottotal.Value
            if total > 0 then
                local targetLand = math.floor(total * (AccuracyTarget / 100))
                shotland.Value = targetLand
            end
        end
    end)

    safeConnect(warning,     function() if warning     then warning.Value     = 0 end end)
    safeConnect(lockflagged, function() if lockflagged then lockflagged.Value = 0 end end)

    local function onCharacterAdded(char)
        local be = char:FindFirstChild("BodyEffects")
        if be then
            local gf  = be:FindFirstChild("GunFiring")
            local gsc = be:FindFirstChild("GunShotChanges")

            if gf then
                local conn = gf:GetPropertyChangedSignal("Value"):Connect(function()
                    if gf then gf.Value = false end
                end)
                table.insert(antiAimConnections, conn)
            end
            if gsc then
                local conn = gsc:GetPropertyChangedSignal("Value"):Connect(function()
                    if gsc then gsc.Value = 0 end
                end)
                table.insert(antiAimConnections, conn)
            end
        end
    end

    if LocalPlayer.Character then onCharacterAdded(LocalPlayer.Character) end
    local conn = LocalPlayer.CharacterAdded:Connect(onCharacterAdded)
    table.insert(antiAimConnections, conn)
end
```

### How it works

1. Every time `ShotTotal` changes (i.e. every shot fired), it recomputes `ShotLand = ShotTotal * (AccuracyTarget / 100)`. With `AccuracyTarget = 0`, this sets accuracy to `0%` regardless of how many shots landed.
2. `Warning` and `LockFlagged` are zeroed whenever the game increments them.
3. `GunFiring` (probably used to gate certain server-side checks) is forced to `false`.
4. `GunShotChanges` (probably tracks suspicious changes in shot patterns) is forced to `0`.

### Why this might backfire

- **Server owns these**: `DataFolder` may be replicated from the server, and if the game detects "client keeps rewriting accuracy to 0", that's *more* suspicious than a high accuracy.
- Some games only check `Warning`/`LockFlagged` server-side, making client writes no-ops.

---

## 13. Anti Mod — Staff Detection & Self-Kick

Detects members of the game's staff group and immediately kicks the local player before they can be investigated.

### Source

```lua
local antiStaffGroupId = 10604500

local function antiStaffNotify(message)
    if _G.AntiModNotification then
        pcall(function()
            game:GetService("StarterGui"):SetCore("SendNotification", {
                Title = "Anti-Mod", Text = message, Duration = 5
            })
        end)
    end
end

local function isStaff(player)
    if not player or not player:IsInGroup(antiStaffGroupId) then return false end
    local success, role = pcall(function()
        return player:GetRoleInGroup(antiStaffGroupId)
    end)
    return success and role ~= "" and role ~= "Guest"
end

local function handleStaffDetected(player)
    local staffName = player.Name
    antiStaffNotify(string.format("STAFF DETECTED: %s has joined!", staffName))
    if _G.AntiModKick then
        task.wait(_G.AntiModKickDelay)
        if isStaff(player) and player.Parent then
            LocalPlayer:Kick(string.format(
                "Anti-Mod: Staff member %s detected. Protection activated.", staffName
            ))
        end
    end
end

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer and isStaff(player) then
        antiStaffNotify(string.format("STAFF ALREADY IN GAME: %s", player.Name))
        if _G.AntiModKick then
            task.wait(_G.AntiModKickDelay)
            LocalPlayer:Kick(string.format(
                "Anti-Mod: Staff member %s is already in game.", player.Name
            ))
        end
        break
    end
end

Players.PlayerAdded:Connect(function(player)
    task.wait(0.5)
    if isStaff(player) then handleStaffDetected(player) end
    player:GetPropertyChangedSignal("GroupRank"):Connect(function()
        task.wait(0.5)
        if isStaff(player) then
            antiStaffNotify(string.format("STAFF DETECTED: %s was promoted!", player.Name))
            handleStaffDetected(player)
        end
    end)
end)
```

### How it works

1. `GroupId = 10604500` is hardcoded (this is the Da Hood staff group).
2. `isStaff(player)` checks group membership + non-empty role.
3. On join, or on `GroupRank` change (promotion), if the player is staff, we notify and then `LocalPlayer:Kick(...)` after a configurable delay.
4. The rationale: if you get kicked before the moderator can spectate you, they can't ban you (they can still manually review logs, but you're out of the session).

### Concerns

- Self-kick is a **desperate measure** — it doesn't prevent server-side bans based on logs.
- `GetRoleInGroup` is a web API call with rate limits; if a large group joins, this spams the endpoint.

---

## 14. Whitelist System

Per-UserID skip list for all target-selection code.

```lua
_G.Whitelist = _G.Whitelist or {}

-- Every target loop checks:
if _G.Whitelist[v.UserId] then continue end
```

Applied in:
- `getClosest` (Silent Aim)
- `enableHCSilentAim` (HC Silent Aim)
- `getFlameTarget` (Flamelock)
- `GetBestCamlockTarget` (Camlock)
- ESP render loop
- `UpdateHitboxes`

The whitelist is stored as `{[UserId] = true}` and is saved/loaded alongside configs.

---

## 15. Speed Master

```lua
RunService.Heartbeat:Connect(function()
    if _G.SpeedMaster and _G.SpeedActive and LocalPlayer.Character then
        local hum = LocalPlayer.Character:FindFirstChild("Humanoid")
        if hum and hum.WalkSpeed ~= _G.SpeedValue then
            hum.WalkSpeed = _G.SpeedValue
        end
    end
end)
```

`WalkSpeed` is a client-owned property on `Humanoid`. The server may or may not replicate. The toggle hint ("Respawn to apply") suggests the server re-applies speed on spawn. The reapply loop only runs while the keybind toggle is active.

---

## 16. Atmosphere / Lighting Overrides

Purely client-side visuals.

```lua
local FogPresets = {
    ["Red"]        = {Color = Color3.fromRGB(255, 60, 60),   Density = 0.45},
    ["Light Red"]  = {Color = Color3.fromRGB(255, 120, 120), Density = 0.44},
    -- ...
}

for name, config in pairs(FogPresets) do
    CreateButton(LeftContent, name, function()
        for _, v in ipairs(game:GetService("Lighting"):GetChildren()) do
            if v:IsA("Atmosphere") then v:Destroy() end
        end
        local atm = Instance.new("Atmosphere", game:GetService("Lighting"))
        atm.Color = config.Color
        atm.Density = config.Density
        atm.Haze = 4
        game:GetService("Lighting").FogStart = 30
        game:GetService("Lighting").FogEnd   = 200
    end)
end
```

`ColorCorrection` uses a `ColorCorrectionEffect` named `ValColorEffect` with adjustable saturation.

```lua
CreateToggle(RightContent, "Color Correction", _G.ColorCorrectionEnabled, function()
    _G.ColorCorrectionEnabled = not _G.ColorCorrectionEnabled
    local Lighting = game:GetService("Lighting")
    local cc = Lighting:FindFirstChild("ValColorEffect")
    if _G.ColorCorrectionEnabled then
        if not cc then cc = Instance.new("ColorCorrectionEffect") end
        cc.Name = "ValColorEffect"
        cc.Enabled = true
        cc.Saturation = 0.5
        cc.Contrast = 0
        cc.Brightness = 0
        cc.TintColor = Color3.fromRGB(255, 255, 255)
        cc.Parent = Lighting
    else
        if cc then cc.Enabled = false end
    end
    return _G.ColorCorrectionEnabled
end)
```

---

## 17. Utility Functions

### Knockdown check

```lua
local function IsKnocked(character)
    if not character then return false end
    local bodyEffects = character:FindFirstChild('BodyEffects')
    if bodyEffects then
        local ko = bodyEffects:FindFirstChild('K.O')
        return ko and ko.Value == true
    end
    return false
end
```

### Grabbed check

```lua
local function IsGrabbed(player)
    return player and player.Character
        and player.Character:FindFirstChild('GRABBING_CONSTRAINT') ~= nil
end
```

### Visibility raycast

```lua
local raycastParams = RaycastParams.new()
raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
raycastParams.IgnoreWater = true

local function isPartVisible(origin, targetPart, ignoreList)
    if not origin or not targetPart then return false end
    local direction = (targetPart.Position - origin).Unit
    local distance  = (targetPart.Position - origin).Magnitude

    local filter = {LocalPlayer.Character}
    if ignoreList then
        for _, v in ipairs(ignoreList) do table.insert(filter, v) end
    end
    raycastParams.FilterDescendantsInstances = filter

    local result = workspace:Raycast(origin, direction * distance, raycastParams)
    if not result then return true end
    return result.Instance == targetPart
        or result.Instance:IsDescendantOf(targetPart.Parent)
end
```

### Closest part to mouse

```lua
local function getClosestPartToMouse(char)
    local m = UIS:GetMouseLocation()
    local nearestPart, nearestDist = nil, math.huge

    local parts = {
        "Head", "UpperTorso", "LowerTorso",
        "LeftUpperArm", "LeftLowerArm", "LeftHand",
        "RightUpperArm", "RightLowerArm", "RightHand",
        "LeftUpperLeg", "LeftLowerLeg", "LeftFoot",
        "RightUpperLeg", "RightLowerLeg", "RightFoot",
        "HumanoidRootPart"
    }

    for _, name in ipairs(parts) do
        local part = char:FindFirstChild(name)
        if part then
            local screenPos, onScreen = cam:WorldToViewportPoint(part.Position)
            if onScreen then
                local dist = (Vector2.new(screenPos.X, screenPos.Y)
                              - Vector2.new(m.X, m.Y)).Magnitude
                if dist < nearestDist then
                    nearestDist = dist
                    nearestPart = part
                end
            end
        end
    end
    return nearestPart
end
```

### Knock tracking (death position cache)

```lua
local function setupKnockTracking(player)
    local function onKnockChanged()
        local character = player.Character
        if not character then return end
        local bodyEffects = character:FindFirstChild("BodyEffects")
        if not bodyEffects then return end
        local KO = bodyEffects:FindFirstChild("K.O")
        if not KO then return end

        if KO.Value then
            local part = character:FindFirstChild("HumanoidRootPart")
                      or character:FindFirstChild("Head")
            if part then _G.DeathPositions[player] = part.Position end
        else
            _G.DeathPositions[player] = nil
        end
    end

    player.CharacterAdded:Connect(function(char)
        local be = char:WaitForChild("BodyEffects", 5)
        if be then
            local KO = be:WaitForChild("K.O", 5)
            if KO then
                KO:GetPropertyChangedSignal("Value"):Connect(onKnockChanged)
                if KO.Value then
                    local part = char:FindFirstChild("HumanoidRootPart")
                              or char:FindFirstChild("Head")
                    if part then _G.DeathPositions[player] = part.Position end
                end
            end
        end
    end)
end
```

---

## Detection Notes for Game Developers

If you're building a game and want to detect/defend against these techniques:

### Metamethod hooking

- `hookmetamethod` on `game` affects only the client. **You cannot detect it directly from the server.**
- However, you can detect *inconsistent* `mouse.Hit` reads: if the client's `mouse.Hit.Instance` is significantly different from the direction they're facing, it's suspicious.
- Server should **never** trust client-provided `mouse.Hit` for hit validation. Always raycast server-side from the shooter's `HumanoidRootPart` in the direction of the target.

### Remote event validation

- Validate `MainEvent:FireServer("Shoot", payload)`:
  - Pellet count must match the equipped tool's stats
  - `payload.Hits` entries must be within reasonable cone from the shooter
  - Cooldown between shots must exceed the weapon's fire rate
  - Ammo count must decrement correctly
  - Server should re-raycast each claimed hit and verify it actually connects

### Accuracy tracking

- If the client is rewriting `ShotLand`/`Warning`/`LockFlagged`, they're not server-owned. **Move these to server-side storage** that the client cannot write to.
- Log every write attempt from the client for a heuristic "cheater score".

### Input anomalies

- **`mousemoverel` signatures**: mouse movement without corresponding physical input device events. On PC, you can compare `UserInputService:GetMouseDelta()` with hardware-reported deltas — mousemoverel injections may not match.
- Aim smoothing via `Camera.CFrame:Lerp` produces mathematically perfect lerps — a detection heuristic can compare camera angular velocity to mouse angular velocity.

### Spread hook detection

- After firing, verify the spread pattern server-side. If the client's claimed shot direction is closer to the target than the max possible spread, reject it.
- Use **server-computed spread** rather than trusting client `math.random`.

### Anti-fall detection

- Track `Humanoid:GetState()` transitions server-side. A player who never enters `FallingDown` despite taking falling damage or being hit by knockdown attacks is suspicious.

### Hitbox expander

- Never trust client `Part.Size` for hit validation.
- If your game replicates `HRP.Size` from client to server for any reason, that's a vulnerability — instead, send only the *intent* to resize and let the server clamp it.

### General anti-cheat

- **Never trust the client with anything gameplay-critical.**
- Use encrypted remote payloads (rolling XOR or AES with a server-generated key rotated each session).
- Log every `FireServer` with payload size/hash — rate and content anomalies are strong signals.
- Consider a server-side heuristic: `(shots_fired / shots_landed)` and `(headshot_ratio)` tracked per-player over time.

---

## Legal

This repository is published **purely for educational study of anti-cheat design**. The author does not condone using this code to disrupt Roblox games or gain unfair advantages. Using executor tools to modify Roblox clients violates the [Roblox Terms of Use](https://en.roblox.com/info/terms) and can result in permanent account termination.

The techniques shown here (Lua metamethod hooking, remote event analysis, client-side prediction) are widely documented in academic reverse-engineering literature and are presented here so that **game developers can build effective countermeasures**.

**Do not use this code.**
