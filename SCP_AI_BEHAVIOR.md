# 🤖 SCP AI 행동 패턴 및 시야 감지 명세서 (SCP AI Behavior & Vision Detection Specification)

> **Document ID:** `SPEC-AI-002`  
> **Target Engine:** Roblox Luau (Server-Side)  
> **Dependencies:** PathfindingService, WorldRoot  
> **Version:** `v1.0.0`

---

## 1. 개요 (Overview)

본 명세서는 SCP 재단 게임 내 적대적 SCP 엔티티(SCP-173, SCP-096, SCP-049 등)의 AI 행동 패턴과 시야 감지(Line of Sight / FOV) 및 응시(Gaze) 판단 알고리즘을 정의합니다. 서버 부하를 최소화하면서 개별 SCP의 특수 기믹을 정확히 구현하는 데 초점을 맞춥니다.

---

## 2. 핵심 AI 아키텍처: 유한 상태 머신 (FSM)

모든 SCP AI는 기본적으로 4가지 상태를 기반으로 동작하며, 개별 특수 상태(Frozen, Enraged)가 추가되는 구조입니다.

| 상태 (State) | 설명 및 실행 로직 | 전이 조건 (Transition Trigger) |
| :--- | :--- | :--- |
| **Patrol (정찰)** | 지정된 웨이포인트(Waypoints)를 순찰하며 플레이어 탐색 | 시야/소리로 플레이어 감지 시 ➔ **Chase** |
| **Chase (추적)** | `PathfindingService`를 사용하여 플레이어 최단 경로 추적 | 사거리 진입 ➔ **Attack** / 시야 상실 지속 ➔ **Patrol** |
| **Attack (공격)** | 플레이어 무력화(처치/감염/목 꺾기) 및 데미지 적용 | 목표 사망 시 ➔ **Patrol** / 타겟 탈출 시 ➔ **Chase** |
| **Frozen / Enraged** | SCP 특수 기믹 (173: 시선 고정 시 정지 / 096: 얼굴 목격 시 폭주) | 시선 이탈 시 ➔ **Instant Attack** / 폭주 종료 시 ➔ **Docile** |

---

## 3. 시야 및 응시 감지 알고리즘 (Vision & Gaze Detection)

SCP-173과 같이 "플레이어가 SCP를 바라보고 있는지" 검증하거나, SCP가 플레이어를 발견했는지 판별할 때 **3단계 레이캐스팅 검사**를 수행합니다.

### 📐 시야/응시 감지 3단계 검증 프로세스
1. **거리 검사 (Distance Check):**  
   플레이어와 SCP 간의 거리가 최대 감지 거리 $R$ 이내인지 검사합니다.
2. **각도 검사 (Dot Product / FOV):**  
   플레이어 카메라/헤드의 시선 방향 벡터 $\vec{V}_{\text{look}}$과 SCP를 향하는 방향 벡터 $\vec{V}_{\text{target}}$ 간의 내적(Dot Product)을 계산합니다.
   $$\cos(\theta) = \vec{V}_{\text{look}} \cdot \vec{V}_{\text{target}}$$
   내적 값이 설정된 FOV 임계값(예: $0.5$, 약 $60^\circ$ 시야각) 이상이어야 시야 내에 존재하는 것으로 판단합니다.
3. **시선 차단 검사 (Raycast Line of Sight):**  
   카메라/헤드 위치에서 SCP의 `HumanoidRootPart` 방향으로 레이를 발사하여 중간에 벽이나 장애물이 없는지 확인합니다.

---

## 4. SCP-173 전용 FSM 및 시선 검사 구현 (`SCP173AI.lua`)

```lua
local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")

local SCP173 = script.Parent
local PrimaryPart = SCP173.PrimaryPart

local MAX_LOOK_DISTANCE = 120
local FOV_THRESHOLD = 0.5 -- 전방 약 60도 범위

-- 플레이어 중 한 명이라도 173을 쳐다보고 있는지 검사하는 핵심 함수
local function IsObservedByAnyPlayer(): boolean
    for _, player in ipairs(Players:GetPlayers()) do
        local char = player.Character
        if char and char:FindFirstChild("Humanoid") and char.Humanoid.Health > 0 then
            local head = char:FindFirstChild("Head")
            if head then
                local dist = (PrimaryPart.Position - head.Position).Magnitude
                if dist <= MAX_LOOK_DISTANCE then
                    -- 1. 내적(Dot Product) 각도 검사
                    local lookVector = head.CFrame.LookVector
                    local dirToSCP = (PrimaryPart.Position - head.Position).Unit
                    local dot = lookVector:Dot(dirToSCP)

                    if dot >= FOV_THRESHOLD then
                        -- 2. Raycast 시선 차단 검사
                        local rayParams = RaycastParams.new()
                        rayParams.FilterAncestorEntities = {char}
                        rayParams.FilterType = Enum.RaycastFilterType.Exclude

                        local result = Workspace:Raycast(head.Position, PrimaryPart.Position - head.Position, rayParams)
                        if result and result.Instance:IsDescendantOf(SCP173) then
                            return true -- 누군가 쳐다보고 있음!
                        end
                    end
                end
            end
        end
    end
    return false
end

-- 메인 AI 루프
task.spawn(function()
    while task.wait(0.1) do
        if IsObservedByAnyPlayer() then
            PrimaryPart.Anchored = true -- 시선 고정 시 움직임 즉시 멈춤
        else
            PrimaryPart.Anchored = false
            -- 최단 거리 플레이어 추적 및 목 꺾기(Teleport/Fast Move) 실행
        end
    end
end)

---
## 5. 서버 최적화 및 예외 처리 (Performance Optimization)
연산 연주기 분할 (Interval Throttling):

매 프레임(RenderStepped) 시야 검사를 진행하면 서버 렉이 발생합니다. AI 상태 체크 타이머를 task.wait(0.1) (초당 10회) 수준으로 조절하여 CPU 오버헤드를 제어합니다.

구역 기반 쿼리 (Spatial Partitioning):

시설 전체의 모든 플레이어를 매번 순회하지 않고, SCP가 위치한 구역(Zone) 내의 플레이어 리스트만 우선 추출하여 거리 검사를 수행합니다.
