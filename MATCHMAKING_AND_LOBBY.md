# 🌐 로비 & 매치메이킹 시스템 명세서 (Matchmaking & Lobby System Specification)

> **Document ID:** `SPEC-LOBBY-003`  
> **Target Engine:** Roblox Luau (Server-Side)  
> **Dependencies:** MemoryStoreService, TeleportService  
> **Version:** `v1.0.0`

---

## 1. 개요 (Overview)

본 명세서는 메인 로비(Lobby Place) 서버와 게임플레이 전용 세션(Raid Place) 간의 플레이어 이동 및 인스턴스 생성을 제어하는 **매치메이킹 및 라운드 수송 관리 시스템**을 정의합니다. High-throughput 인메모리 큐(`MemoryStoreQueue`)와 전용 비공개 서버 API(`TeleportService:ReserveServer`)를 결합하여 안정적인 대기열 처리와 인스턴스 격리를 보장합니다.

---

## 2. 매치메이킹 아키텍처 및 세션 이동 흐름

전체 매치메이킹 프로세스는 다음과 같은 4단계 시퀀스로 진행됩니다.

| 단계 (Step) | 주요 서비스 API | 설명 및 처리 로직 |
| :--- | :--- | :--- |
| **1. 대기열 등록** | `MemoryStoreQueue:AddAsync` | 클라이언트가 대기열 진입 버튼 클릭 시, 서버는 해당 파티/유저의 정보 및 난이도 설정을 메모리 큐에 대기 등록 |
| **2. Matchmaker 스케줄링** | `MemoryStoreQueue:ReadAsync` | 로비 서버의 매치메이커 프로세서가 큐에서 지정된 정원(예: 4명)의 플레이어를 묶어 매칭 그룹 구성 |
| **3. 예약 서버 생성** | `TeleportService:ReserveServer` | 매칭이 성사된 그룹을 위해 대상 Place ID(레이드 맵)의 전용 비공개 인스턴스 AccessCode 동적 발행 |
| **4. 파티 Teleport** | `TeleportService:TeleportPartyAsync` | 세션 설정 메타데이터를 `TeleportOptions`에 동봉하여 그룹 전체를 지정된 예약 인스턴스로 수송 |

---

## 3. TeleportData 보안 및 Server-To-Server 검증

1. **데이터 변조 방지 (Anti-Tampering):**  
   * 클라이언트 측에서 손대거나 조작할 수 없도록, 중요 장비 데이터/재화 정보는 `TeleportData`에 담지 않고 수신 서버가 **`ProfileService` DataStore**에서 직접 읽어옵니다.
2. **세션 메타데이터 전달:**  
   * `TeleportData`에는 오직 **`SessionId`**, **`Difficulty` (선택된 난이도)**, **`MatchmakingTimestamp`** 등 인스턴스 초기화용 설정값만 포함시킵니다.

---

## 4. 핵심 매치메이킹 로직 구현 (`MatchmakingManager.lua`)

```lua
local MemoryStoreService = game:GetService("MemoryStoreService")
local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")

local RAID_PLACE_ID = 123456789 -- 레이드 인게임 Place ID
local MatchQueue = MemoryStoreService:GetQueue("RaidMatchQueue_v1")

local PARTY_SIZE = 4

-- 1. 매칭 대기열 추가 함수
local function EnqueuePlayer(player: Player, difficulty: string)
    local playerData = {
        UserId = player.UserId,
        Difficulty = difficulty or "Normal",
        Timestamp = os.time()
    }
    MatchQueue:AddAsync(playerData, 300) -- 5분 TTL 설정
end

-- 2. 매치메이커 프로세서 (주기적 실행)
task.spawn(function()
    while task.wait(2) do
        local success, items, id = pcall(function()
            return MatchQueue:ReadAsync(PARTY_SIZE, false, 10)
        end)

        if success and items and #items >= PARTY_SIZE then
            local matchedPlayers = {}
            for _, item in ipairs(items) do
                local plr = Players:GetPlayerByUserId(item.UserId)
                if plr then
                    table.insert(matchedPlayers, plr)
                end
            end

            -- 정원이 완전히 가득 찬 경우 예약 서버 생성 후 이동
            if #matchedPlayers == PARTY_SIZE then
                local reservedCode, serverId = TeleportService:ReserveServer(RAID_PLACE_ID)
                local options = Instance.new("TeleportOptions")
                options.ReservedServerAccessCode = reservedCode
                options:SetTeleportData({
                    SessionId = serverId,
                    Difficulty = items[1].Difficulty
                })

                TeleportService:TeleportPartyAsync(RAID_PLACE_ID, matchedPlayers, options)
                MatchQueue:RemoveAsync(id) -- 매칭 완료 항목 제거
            end
        end
    end
end)
```

---

## 5. 실패 및 예외 처리 (Error Handling)
TeleportInitData 실패 시 복귀 (Teleport Failure Fallback):

이동 실패 시(Enum.TeleportResult.Failed) TeleportService.TeleportInitFailed 이벤트를 수신하여 클라이언트 UI에 알림 메시지를 표시하고, 대기열 재진입 여부를 묻는 팝업을 띄웁니다.

이탈자 처리 (Match Leave):

매칭 대기 도중 플레이어가 로비를 이탈하면 Players.PlayerRemoving 이벤트를 탐지하여 MemoryStoreQueue에서 해당 유저의 큐 항목을 즉시 제거/취소 처리합니다.
