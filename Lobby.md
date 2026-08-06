# 🏢 로블록스 레이드 로비 및 대기실 시스템 기획서 (Lobby.md) - [파트 1/2]

본 문서는 플레이어가 최초 접속하는 ‘메인 로비(Lobby Space)’와 보스 레이드 진입을 위한 ‘대기실(Matchmaking/Match Space)’의 UI 레이아웃 및 서버 통신 구조를 정의합니다.

---

## 🗺️ 1. 로비 공간 레이아웃 및 필수 구역 (Lobby Spatial Design)

로비는 모든 유저가 공유하는 안전 구역(Safe Zone)이며, 다음과 같은 3개의 핵심 NPC/구조물이 배치됩니다.

1. **무기 정비 상점 (Weapon Shop NPC)**
   * **기능**: `Weapon.md`에 정의된 15종의 무기를 골드로 구매하거나 전설 무기를 강화하는 구역입니다.
   * **구현**: NPC 앞에 `ProximityPrompt`(E키 상호작용)를 배치하여 유저가 다가오면 상점 GUI가 열리도록 설계합니다.
2. **보스 선택 전광판 (Boss Selection Board)**
   * **기능**: 현재 도전 가능한 SCP 보스(1~20번)의 목록, 권장 인원, 처치 보스 보상 골드를 시각적으로 보여주는 디스플레이 스크린입니다.
3. **보스방 대기 정거장 (Raid Matchmaking Zones)**
   * **기능**: 최대 6명의 플레이어가 발판(Pad) 위에 올라가 인원을 모집하는 물리적 매치메이킹 공간입니다.

---

## 👥 2. 대기실(매치메이킹) 발판 시스템 (Raid Pad System)

유저들이 복잡한 매칭 창을 켜지 않고, 직관적으로 파티를 짤 수 있도록 **"물리 발판(Match Pad)"** 방식을 채택합니다.

### ⚙️ 대기실 발판 규칙
* **발판 구조**: 보스전 입구 앞에 총 6개의 원형 발판(`Part`)이 배치되어 있습니다.
* **인원 감지**: 유저가 발판 위에 올라서면 캐릭터가 고정되며 화면에 **"대기 중인 플레이어 (1/6)"** UI와 **5초 카운트다운**이 시작됩니다.
* **인원별 보스 체력 연동**: 카운트다운이 종료되는 순간, 발판 위에 서 있는 최종 인원수(#Players)를 체크하여 `BossCharacter.md`에 기획된 체력 스케일 변수값을 보스방 서버로 전송합니다.

---

## 🖥️ 3. 로비 HUD 및 GUI 레이아웃 (Lobby UI Screen)

로비에서 항상 화면에 표시되는 기본 HUD 구조입니다.

[화면 상단 중앙] ------------ ⏱️ 보스전 입장 카운트다운 (발판 진입 시 활성화)
[화면 우측 상단] ------------ 💰 현재 보유 골드 (DataStore 연동 leaderstats)
[화면 좌측 하단] ------------ 🔫 현재 장착 중인 무기 아이콘 및 등급 (일반~전설)
[화면 우측 하단] ------------ ⚙️ 환경 설정 / 게임 패드 조작 가이드 보기

## 🛠️ 4. 로블록스 대기실 발판 구현 핵심 스크립트

`Workspace`의 대기 발판 구역에 배치할 파티 매칭 및 보스방 이동(`TeleportService`) 핵심 서버 스크립트 구조입니다.

```lua
-- [서버 스크립트] 로비 대기 발판 매니저 (LobbyMatchManager)
local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")

local MatchPadGroup = workspace.RaidMatchPads -- 6개의 발판이 모인 모델
local BossPlaceId = 12345678 -- 이동할 보스방 전용 Place ID (로블록스 세팅 값)

local readyPlayers = {}
local isCountingDown = false
local countdownTime = 5

-- 발판 위에 올라선 유저 감지
for _, pad in pairs(MatchPadGroup:GetChildren()) do
	pad.Touched:Connect(function(hit)
		local character = hit.Parent
		local player = Players:GetPlayerFromCharacter(character)
		
		if player and not table.find(readyPlayers, player) and #readyPlayers < 6 then
			table.insert(readyPlayers, player)
			print(player.Name .. " 레이드 대기열 합류. 현재 인원: " .. #readyPlayers .. "/6")
			
			-- 캐릭터를 발판 중앙에 살짝 고정 (WalkSpeed 제어)
			character.Humanoid.WalkSpeed = 0
			
			-- 카운트다운 트리거 호출
			if not isCountingDown then
				startRaidCountdown()
			end
		end
	end)
end

-- 레이드 진입 카운트다운 로직
function startRaidCountdown()
	isCountingDown = true
	local currentCount = countdownTime
	
	while currentCount > 0 and #readyPlayers > 0 do
		-- readyPlayers에 속한 유저들의 화면에 카운트다운 GUI 전송 (RemoteEvent 이용)
		print("레이드 진입까지... " .. currentCount .. "초")
		task.wait(1)
		currentCount = currentCount - 1
	end
	
	-- 카운트다운 종료 후 텔레포트 진행
	if #readyPlayers > 0 then
		print("최종 " .. #readyPlayers .. "인 파티 보스방 전송 시작.")
		
		-- TeleportOptions에 현재 파티 인원수를 담아서 보스방 서버로 전송
		-- 보스방 서버는 이 값을 받아 BossCharacter.md의 인원별 체력을 실시간 세팅함
		local teleportOptions = Instance.new("TeleportOptions")
		local teleportData = {
			PlayerCount = #readyPlayers,
			HostPlayer = readyPlayers[1].Name
		}
		teleportOptions:SetTeleportData(teleportData)
		
		-- 파티원 전원 동시 순간이동 (서버 인스턴스 분리)
		local teleportResult = TeleportService:TeleportPartyAsync(BossPlaceId, readyPlayers, teleportOptions)
		
		-- 로비 데이터 초기화
		for _, player in pairs(readyPlayers) do
			if player.Character and player.Character:FindFirstChild("Humanoid") then
				player.Character.Humanoid.WalkSpeed = 16 -- 스피드 복구
			end
		end
		readyPlayers = {}
		isCountingDown = false
	end
end
```

---

## 🔒 5. 대기실 이탈 및 예외 처리 가이드라인 (Exception Handling)

1. **카운트다운 중 유저 탈주 대응**
   * 발판 위에서 유저가 게임을 종료(`PlayerRemoving`)하거나 강제로 움직여 대기열에서 빠질 경우, 즉시 `readyPlayers` 테이블에서 해당 유저를 제거(`table.remove`)합니다.
   * 대기 인원이 0명이 되면 카운트다운 루프를 즉시 취소(`break`)하고 타이머를 초기화합니다.
2. **무기 미장착 유저 제한**
   * 상점에서 무기를 단 하나도 구매하지 않았거나 기본 무기를 장착하지 않은 유저는 발판에 닿아도 대기열 등록을 거부하고, 화면 중앙에 **"무기를 먼저 장착해야 레이드에 참여할 수 있습니다."** 경고 팝업 UI를 띄웁니다.
