





### **TECHNICAL DESIGN DOCUMENT // GAME MECHANICS** 

# **라운드 루프 & 게임 모드 명세서** 

Round Loop & Game Mode Architecture Specification 

**DOCUMENT ID SPEC-ROUND-004** 

**VERSION / DATE v1.0.0 (2026.08)** 

**TARGET ENGINE Roblox Luau** 

**DEPENDENCIES ProfileService, TeleportService** 



## **1. 라운드 상태 머신 (Round FSM)** 

### **상태 (State)** 

### **설명 및 제어 로직** 

|**WAITINGFORPLAYERS**|수송 완료된 플레이어 로딩 검증 및 캐릭터 스<br>폰 준비|
|---|---|
|**INGAME**|SCP AI활성화, 미션 목표(탈출/격리) 수행 타<br>이머 가동|
|**ENDING**|결과 창 출력, 보상(크레딧/EXP) 서버 데이터<br>연동 저장|
|**CLEANUP**|모든 플레이어 로비Place로 수송 및 서버 인<br>스턴스 종료|



### **다음 상태 전이 조건** 

전원 로딩 완료 또는 대기 시간(30초) 초과 ➔ **InGame** 

목표 달성/전원 탈출 시 ➔ **Ending** / 제한시간 초과/전멸 시 ➔ **Ending** 

보상 정산 처리 완료(10초) ➔ **Cleanup** 

플레이어 수송 완료 후 인스턴스 자동 파기 

## **2. 라운드 매니저 서버 구현 (Lua Example)** 

- _`-- RoundManager.lua (Server Script)`_ 

`local Players = game:GetService("Players"` ) `local TeleportService = game:GetService("TeleportService"` ) 

`local LOBBY_PLACE_ID =` 987654321 `local RoundState = "WaitingForPlayers"` 

`local function StartRound` () `RoundState = "InGame"` 

_`-- SCP AI` 및 타이머 활성화_ `task.delay(` 600, `function` () 

`if RoundState == "InGame" then EndRound` ( `false` , `"Time Out"` ) 

```
end
```

`end` ) 

```
end
```

```
functionEndRound(isSuccess, reason)
    RoundState = "Ending"
```

_-- 결과 저장 및 보상 지급 로직_ `task.wait(` 10) `RoundState = "Cleanup"` 

Page 1 of 2 

CONFIDENTIAL // SCP FOUNDATION ROBLOX DEV TEAM 

`TeleportService:TeleportPartyAsync(LOBBY_PLACE_ID, Players:GetPlayers` ()) `end` 

Page 2 of 2 

CONFIDENTIAL // SCP FOUNDATION ROBLOX DEV TEAM 

