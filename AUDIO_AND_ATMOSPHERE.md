





### **TECHNICAL DESIGN DOCUMENT // AUDIO & VISUALS** 

# **사운드 시스템 & 연출 명세서** 

3D Dynamic Audio & Horror Atmospheric Rendering Specification 

**DOCUMENT ID SPEC-AUDIO-006** 

**VERSION / DATE v1.0.0 (2026.08)** 

**TARGET ENGINE Roblox Luau** 

**DEPENDENCIES SoundService, Lighting** 



## **1. SCP 위협 기반 동적 심장소리 연출 (Proximity Heartbeat)** 

```
-- HeartbeatController.lua (Client Script)
```

`local SoundService = game:GetService("SoundService"` ) `local Workspace = game:GetService("Workspace"` ) `local Player = game:GetService("Players").LocalPlayer` 

`local HeartbeatSound = SoundService:WaitForChild("Heartbeat"` ) `local MAX_DANGER_DIST =` 60 

`task.spawn(function` () 

`while task.wait(` 0.2) `do` 

```
local char = Player.Character
```

`if char and char:FindFirstChild` ( `"HumanoidRootPart"` ) `then local hrp = char.HumanoidRootPart` 

```
local closestDist = MAX_DANGER_DIST
```

`for _, scp in ipairs(Workspace.SCPs:GetChildren` ()) `do` 

`if scp:FindFirstChild` ( `"PrimaryPart"` ) `then` 

```
local dist = (hrp.Position - scp.PrimaryPart.Position).Magnitude
if dist < closestDist then closestDist = dist end
```

```
end
```

```
end
```

```
if closestDist < MAX_DANGER_DIST then
```

`if not HeartbeatSound.IsPlaying then HeartbeatSound:Play` () `end` 

`local intensity =` 1 `- (closestDist / MAX_DANGER_DIST) HeartbeatSound.Volume = intensity *` 0.8 

`HeartbeatSound.PlaybackSpeed =` 0.8 `+ (intensity *` 0.6) `else` 

`HeartbeatSound:Stop` () `end` 

```
end
```

`end end` ) 

Page 1 of 1 

CONFIDENTIAL // SCP FOUNDATION ROBLOX DEV TEAM 

