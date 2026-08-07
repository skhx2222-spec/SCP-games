





### **TECHNICAL DESIGN DOCUMENT // SECURITY** 

# **안티치트 & 보안 서버 검증 명세서** 

Server-Authoritative Anti-Cheat Specification 

**DOCUMENT ID SPEC-SEC-005** 

**VERSION / DATE v1.0.0 (2026.08)** 

**TARGET ENGINE Roblox Luau** 

### **DEPENDENCIES** 

**Server Script Service** 



## **1. 클라이언트 변조 방지 규칙 (Sanity Checks)** 

**감지 대상 (Cheat Type)** 

### **서버 검증 알고리즘** 

### **처벌 제어 (Action)** 

**Speed / Teleport** 

델타 타임 간 이동 거리 계산: $ ext{Speed} = \Delta ext{Position} / \Delta t > ext{MaxAllowed}$ 

이전 위치 서버 롤백 & 경고 스택 부여 

**Wallbang / NoClip** 

**Infinite Ammo / Rapid Fire** 

사격 위치와 타깃 사이에 지형/벽 Raycast 차단 검사 

서버 무기 쿨다운 타임 및 잔여 탄약 수량 매칭 검증 

데미지 판정 무효화 (0 처리) 발사 이벤트 거부 및 세션 차 단 

## **2. 서버 속도 검증 로직 구현 (Lua Example)** 

```
-- SpeedSanityCheck.lua (Server Script)
```

`local MAX_SPEED =` 24 _-- 최대 허용 속도 (_ _`Studs/sec` )_ `local TOLERANCE =` 1.2 

```
local lastPositions = {}
local lastTimes = {}
```

```
game:GetService("Players").PlayerAdded:Connect(function(player)
```

```
    player.CharacterAdded:Connect(function(char)
```

`local hrp = char:WaitForChild("HumanoidRootPart"` ) `lastPositions[player] = hrp.Position lastTimes[player] = os.clock()` 

`task.spawn(function` () `while char.Parent do` 

`task.wait(` 0.5) 

```
local now = os.clock()
```

```
local dt = now - lastTimes[player]
local dist = (hrp.Position - lastPositions[player]).Magnitude
local speed = dist / dt
```

```
if speed > (MAX_SPEED * TOLERANCE) then
```

`hrp.CFrame = CFrame.new(lastPositions[player])` _-- 롤백_ `else` 

```
                    lastPositions[player] = hrp.Position
                    lastTimes[player] = now
```

```
end
```

Page 1 of 2 

CONFIDENTIAL // SCP FOUNDATION ROBLOX DEV TEAM 



