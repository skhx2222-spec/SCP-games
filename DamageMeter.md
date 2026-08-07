# 📊 로블록스 실시간 대미지 미터기 및 레이드 결과 창 기획서 (DamageMeter.md) 

본 문서는 보스 레이드 진행 중 파티원들의 누적 대미지를 실시간으로 추적하는 HUD UI와, 보스 처치(또는 타임오버) 후 최종 기여도 및 골드 보상을 정산하여 보여주는 결과 창 GUI 시스템을 정의합니다.

---

## 🖥️ 1. 실시간 대미지 미터기 HUD 레이아웃 (In-Game HUD)

보스방 레이드 구역에 진입하면 화면 우측 상단에 자동으로 활성화되는 소형 프레임(`Frame`) 구조입니다. 5분 동안 파티원 간의 대미지 경쟁을 실시간으로 유도합니다.

+-------------------------------------------------------------+
|  📊 대미지 미터기 (DAMAGE METER)                           
|+-------------------------------------------------------------+

|  1. Player_A  (AR 전설)   |||||||||||||||||||||  12,450 (42%) |
|  2. Player_B  (SR 영웅)   |||||||||||||||        9,120  (31%) |
|  3. Player_C  (SMG 희귀)  ||||||||               4,550  (15%) |
|  4. Player_D  (샷건 일반) |||||                  2,100  (7%)  |
|  5. Player_E  (메딕 일반) |||                    1,200  (4%)  |
|  6. Player_F  (석궁 일반) |                      450    (1%)  
|+-------------------------------------------------------------+

## 🧱 2. HUD 컴포넌트 상세 명세 (HUD Component Specification)

1. **MeterContainer (메인 프레임 Container)**
   * 보스방 진입 및 레이드 시작 원격 신호를 받으면 `Visible = true` 상태로 활성화되며, 레이드 종료 시 자동으로 화면에서 사라집니다.
   * 내부에 `UIListLayout` 인스턴스를 포함하여, 파티원들의 누적 대미지 변수 변화에 따라 1등부터 6등까지의 UI 행 포지션이 실시간으로 상하 재정렬됩니다.

2. **PlayerRow (플레이어별 정렬 행 Frame)**
   * **기본 정보 텍스트 (NameLabel)**: `[현재 순위]. [유저 닉네임] ([장착 중인 무기 유형 및 현재 등급])` 정보를 한눈에 볼 수 있도록 표시합니다.
   * **대미지 그래프 바 (ProgressBar)**: 파티 전체가 보스에게 가한 총 누적 대미지 양을 기준으로, 해당 유저가 기여한 대미지 지분을 시각화하는 가로 막대 그래프입니다. 
   * **우측 스탯 데이터 (ValueLabel)**: 콤마(,)가 포함된 정확한 누적 대미지 정수 값과 괄호 안의 소수점 첫째 자리 백분율 지분(`%`)을 0.5초 주기로 갱신하여 출력합니다. (예: `12,450 (42.5%)`)

## 🏆 3. 레이드 최종 결과 정산 창 레이아웃 (Raid Result UI Window)

보스를 처치하거나 5분 제한 시간이 만료되었을 때, 화면 중앙에 거대하게 팝업되는 최종 정산 프레임(`Frame`) 구조입니다. 파티의 성패와 최고 기여자를 웅장하게 연출합니다.

+-----------------------------------------------------------------------------------------+
| 🎉 레이드 클리어! (VICTORY)|
| [ SCP-173 파괴 완료 ] |
+-----------------------------------------------------------------------------------------+
|👑 PARTY MVP 👑|
| Player_A |
| (총 대미지: 15,400)  |
+-----------------------------------------------------------------------------------------+
|[ 파티원 기여도 리포트 ] |
|   ■ 1등: Player_A  (AR 전설)   --------------------- 15,400 Damage (기여도 45%) |
|   ■ 2등: Player_B  (SR 영웅)   --------------------- 10,200 Damage (기여도 30%) |
|   ■ 3등: Player_C  (SMG 희귀)  ---------------------  5,100 Damage (기여도 15%) |
|   ■ 4등: Player_D  (샷건 일반) ---------------------  2,300 Damage (기여도 7%) |
|   ■ 5등: Player_E  (메딕 일반) ---------------------  1,000 Damage (기여도 3%) |
|   ■ 6등: Player_F  (석궁 일반) ---------------------      0 Damage (기여도 0%)  |
+-----------------------------------------------------------------------------------------+
|  💰 기본 처치 보상: +65 Gold                                                           |
|  👑 MVP 추가 보너스: +20 Gold (Player_A 전용)                                    |
+-----------------------------------------------------------------------------------------+
| [ 메인 로비로 돌아가기 ] |
+-----------------------------------------------------------------------------------------+

## 🧱 4. 결과 창 컴포넌트 상세 명세 (Result UI Component Specification)

1. **TitleLabel (결과 타이틀 레이블)**
   * 레이드 성공 시: 초록색 폰트와 함께 "🎉 레이드 클리어! (VICTORY)" 및 처치한 SCP 이름이 출력됩니다.
   * 레이드 실패(5분 초과/전멸) 시: 붉은색 폰트와 함께 "💀 레이드 실패 (DEFEAT)" 및 "제한 시간 초과" 또는 "파티 전멸" 메시지가 출력됩니다.

2. **MvpFrame (최고 기여 유저 프레임)**
   * 파티원 중 보스에게 가장 높은 누적 대미지를 입힌 1등 유저(MVP)의 닉네임과 총 대미지 양을 중앙에 크게 강조하여 독점 노출합니다.
   * 로블록스 `UIStroke`를 사용해 골드 테두리 광원 연출을 추가합니다.

3. **RewardFrame (보상 정산 레이블)**
   * `BossCharacter.md`에 설정된 해당 보스 고유의 기본 골드 보상 수치를 연동하여 표시합니다. (예: "+65 Gold")
   * MVP 달성 유저에게는 기본 처치 보상의 30%에 해당하는 [MVP 추가 보너스 골드]가 UI에 추가로 명시되며 해당 유저의 세이브 데이터에만 가산됩니다.

4. **LobbyReturnButton (로비 귀환 버튼)**
   * 클릭 시 로비 공간으로 안전하게 퇴장시키는 버튼입니다.
   * 유저가 클릭하지 않더라도 결과 창 팝업 15초 후 자동으로 로비 텔레포트 함수가 서버에서 강제 작동하도록 설계합니다.

## 📡 5. 백엔드 네트워크 통신 규격 (Damage Network Protocol)

대미지 데이터는 유저의 치트(변수 조작)에 취약하므로, 반드시 서버가 주도하여 계산하고 클라이언트로 내려주는 **하향식 단방향 동기화 구조**를 채택합니다.

* **사용한 원격 이벤트**: 
  1. `ReplicatedStorage.Network.UpdateDamageMeter` (RemoteEvent - 서버에서 클라이언트로 브로드캐스트)
  2. `ReplicatedStorage.Network.RaidMatchResult` (RemoteEvent - 레이드 종료 시 최종 데이터 전송)

* **통신 아키텍처 흐름**:
  1. **사격 적중 (서버)**: 유저가 레이캐스트 사격을 하여 보스를 맞추면, 무기 스펙(`Weapon.md`) 대미지 연산을 서버 스크립트 내부에서 처리하고 보스 체력을 깎습니다.
  2. **실시간 누적 (서버)**: 서버는 내부 딕셔너리 데이터 테이블에 `[Player] = CumulativeDamage` 형태로 실시간 누적 대미지를 가산합니다.
  3. **주기적 동기화 (서버 ➔ 클라이언트)**: 서버는 0.5초마다 파티원 전체의 누적 대미지 테이블을 패킹하여 `UpdateDamageMeter:FireAllClients(DamageTable)`로 보스방 안의 유저들에게 브로드캐스트합니다.
  4. **최종 정산 (서버 ➔ 클라이언트)**: 보스 사망 혹은 5분 종료 시, 서버는 최종 정산 데이터를 묶어 `RaidMatchResult:FireAllClients(ResultData)`를 호출하여 결과 창 GUI를 강제로 팝업시킵니다.

## 🛠️ 6. 서버사이드 대미지 트래커 핵심 스크립트 (전반부)

`ServerScriptService` 내부에 `DamageTrackManager`라는 이름의 Script를 생성하고 아래 코드를 삽입합니다. 이 스크립트는 레이드 시작 시 유저들의 대미지 저장 테이블을 초기화하고 사격 적중 시 데이터를 누적합니다.

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

-- RemoteEvent 참조 (ReplicatedStorage.Network 폴더 내에 선행 생성 필수)
local UpdateDamageMeter = ReplicatedStorage.Network:WaitForChild("UpdateDamageMeter")
local RaidMatchResult = ReplicatedStorage.Network:WaitForChild("RaidMatchResult")

-- 현재 레이드 세션의 파티원 대미지 누적 딕셔너리 테이블
local raidDamageTable = {}
local totalPartyDamage = 0
local isRaidActive = false

-- 레이드 세션 시작 함수 (보스방 텔레포트 완료 시 서버에서 호출)
local function startRaidSession(partyPlayers)
	raidDamageTable = {}
	totalPartyDamage = 0
	isRaidActive = true
	
	-- 파티 진입 인원들의 대미지 테이블 초기값 0으로 세팅
	for _, player in pairs(partyPlayers) do
		if player:IsA("Player") then
			raidDamageTable[player.UserId] = {
				Name = player.Name,
				Damage = 0,
				WeaponInfo = player.WeaponData.EquippedWeapon.Value .. " (" .. player.WeaponData.WeaponTier.Value .. ")"
			}
		end
	end
	print("레이드 대미지 트래커 세션 활성화 완료.")
end

-- 유저가 보스/잡몹 타격 시 서버에서 호출되는 대미지 가산 함수
local function registerPlayerDamage(player, damageValue)
	if not isRaidActive or not player then return end
	
	local userData = raidDamageTable[player.UserId]
	if userData then
		userData.Damage = userData.Damage + damageValue
		totalPartyDamage = totalPartyDamage + damageValue
	end
end
```

-- 3. 0.5초 주기 실시간 데이터 브로드캐스트 동기화 루프
task.spawn(function()
	while true do
		task.wait(0.5)
		if isRaidActive and totalPartyDamage > 0 then
			-- 클라이언트로 실시간 패킹된 대미지 테이블 전송
			-- 클라이언트 LocalScript는 이 테이블을 받아 화면 우측 HUD를 실시간 정렬함
			UpdateDamageMeter:FireAllClients(raidDamageTable, totalPartyDamage)
		end
	end
end)

-- 4. 레이드 종료 및 골드 보상 최종 정산 처리 함수
local function endRaidSession(isVictory, baseGoldReward)
	if not isRaidActive then return end
	isRaidActive = false
	
	local mvpPlayerId = nil
	local maxDamage = -1
	
	-- MVP (최대 대미지 투사 유저) 선출 루프
	for userId, data in pairs(raidDamageTable) do
		if data.Damage > maxDamage and data.Damage > 0 then
			maxDamage = data.Damage
			mvpPlayerId = userId
		end
	end
	
	-- 파티원 전원에게 결과 전송 및 골드 보상 데이터스토어 인가
	for userId, data in pairs(raidDamageTable) do
		local player = Players:GetPlayerByUserId(userId)
		if player then
			local finalGold = baseGoldReward
			local isMvp = (userId == mvpPlayerId)
			
			-- MVP 보너스 상금 가산 (기본 보상의 30% 반올림 연산)
			if isMvp then
				finalGold = finalGold + math.floor(baseGoldReward * 0.3)
			end
			
			-- leaderstats 골드 실시간 가산 (DataStoreService.md 연동 구조)
			if player:FindFirstChild("leaderstats") and player.leaderstats:FindFirstChild("Gold") then
				player.leaderstats.Gold.Value = player.leaderstats.Gold.Value + finalGold
			end
		end
	end
	
	-- 최종 랭킹 및 보상 데이터 패키지를 클라이언트로 쏘아 결과창 GUI 강제 토글
	local resultPayload = {
		Victory = isVictory,
		MvpId = mvpPlayerId,
		BaseReward = baseGoldReward,
		FinalTable = raidDamageTable
	}
	RaidMatchResult:FireAllClients(resultPayload)
	print("레이드 세션 종료 및 경제 데이터 정산 동기화 완료.")
end

-- 외부에 함수 노출 바인딩 (보스 제어 메인 스크립트에서 호출용)
_G.StartRaidDamageTrack = startRaidSession
_G.RegisterRaidDamage = registerPlayerDamage
_G.EndRaidDamageTrack = endRaidSession
---

## 🛠️ 7. 클라이언트 HUD 실시간 UI 업데이트 스크립트

`StarterPlayerScripts` 내부 혹은 대미지 미터기 HUD GUI 프레임 내부의 `LocalScript`로 배치할 소스코드 구조입니다. 서버가 0.5초마다 쏘아주는 데이터 패키지를 받아 화면 우측 UI의 등수와 그래프 길이를 실시간으로 재정렬합니다.

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local UpdateDamageMeter = ReplicatedStorage.Network:WaitForChild("UpdateDamageMeter")
local localPlayer = Players.LocalPlayer

-- GUI 인스턴스 참조 (HUD 프레임 내부 스크립트 기준)
local MeterContainer = script.Parent:WaitForChild("MeterContainer")
local TemplateRow = MeterContainer:WaitForChild("TemplateRow") -- 원본 복사용 UI 행

-- 서버로부터 실시간 대미지 동기화 신호를 받았을 때 실행되는 루틴
UpdateDamageMeter.OnClientEvent:Connect(function(damageTable, totalPartyDamage)
	if totalPartyDamage <= 0 then return end
	
	-- 1. 데이터 테이블을 정렬하기 위해 배열(Array) 형태로 변환
	local sortedArray = {}
	for userId, data in pairs(damageTable) do
		table.insert(sortedArray, {
			UserId = userId,
			Name = data.Name,
			Damage = data.Damage,
			WeaponInfo = data.WeaponInfo
		})
	end
	
	-- 2. 대미지 양을 기준으로 내림차순(1등부터 순서대로) 정렬
	table.sort(sortedArray, function(a, b)
		return a.Damage > b.Damage
	end)
	
	-- 3. 정렬된 순서대로 UI 요소 리프레시 및 등수 배치
	for rank, data in ipairs(sortedArray) do
		local rowFrame = MeterContainer:FindFirstChild("User_" .. data.UserId)
		
		-- 해당 유저의 UI 행이 없다면 템플릿을 복제하여 새로 생성
		if not rowFrame then
			rowFrame = TemplateRow:Clone()
			rowFrame.Name = "User_" .. data.UserId
			rowFrame.Visible = true
			rowFrame.Parent = MeterContainer
		end
		
		-- 대미지 지분율 백분율 연산 (% 계산)
		local contributionPercent = math.floor((data.Damage / totalPartyDamage) * 100)
		
		-- UI 내부 컴포넌트 텍스트 및 데이터 인가
		rowFrame.NameLabel.Text = rank .. ". " .. data.Name .. " (" .. data.WeaponInfo .. ")"
		rowFrame.ValueLabel.Text = string.format("%d", data.Damage) .. " (" .. contributionPercent .. "%)"
		
		-- 가로 막대 그래프 길이 트윈 연출 (지분율만큼 가로 길이 조정)
		rowFrame.ProgressBar.Bar:TweenSize(
			UDim2.new(contributionPercent / 100, 0, 1, 0),
			Enum.EasingDirection.Out,
			Enum.EasingStyle.Quad,
			0.2,
			true
		)
		
		-- UIListLayout이 자동으로 위아래 정렬하도록 LayoutOrder를 등수로 매핑 (낮을수록 위로 감)
		rowFrame.LayoutOrder = rank
	end
end)
```

### ③ 클라이언트 최종 결과 정산 창 UI 제어 스크립트 (전반부)

`StarterGui` 또는 결과 창 프레임 내부에 배치할 `LocalScript` 소스코드의 전반부입니다. 레이드가 종료되는 순간 서버에서 정산 패키지를 수신하여 화면 중앙에 결과 창을 서서히 띄우고 텍스트를 인가합니다.

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")

local RaidMatchResult = ReplicatedStorage.Network:WaitForChild("RaidMatchResult")
local localPlayer = Players.LocalPlayer

-- GUI 컴포넌트 참조
local ResultFrame = script.Parent:WaitForChild("ResultFrame") -- 전체 결과창 프레임
local TitleLabel = ResultFrame:WaitForChild("TitleLabel")
local MvpLabel = ResultFrame:WaitForChild("MvpLabel")
local ReportContainer = ResultFrame:WaitForChild("ReportContainer")
local RewardLabel = ResultFrame:WaitForChild("RewardLabel")
local MvpRewardLabel = ResultFrame:WaitForChild("MvpRewardLabel")

-- 서버로부터 최종 정산 패키지를 받았을 때 구동되는 이벤트
RaidMatchResult.OnClientEvent:Connect(function(resultPayload)
	-- 1. 성공/실패 여부에 따른 타이틀 및 컬러 변경 연출
	if resultPayload.Victory then
		TitleLabel.Text = "🎉 레이드 클리어! (VICTORY)"
		TitleLabel.TextColor3 = Color3.fromRGB(85, 255, 127) -- 초록색
	else
		TitleLabel.Text = "💀 레이드 실패 (DEFEAT)"
		TitleLabel.TextColor3 = Color3.fromRGB(255, 85, 85) -- 붉은색
	end
	
	-- 2. MVP 유저 닉네임 표출 (MvpId가 존재할 경우)
	if resultPayload.MvpId then
		local mvpData = resultPayload.FinalTable[resultPayload.MvpId]
		if mvpData then
			MvpLabel.Text = "👑 PARTY MVP 👑\n" .. mvpData.Name .. " (총 대미지: " .. string.format("%d", mvpData.Damage) .. ")"
		end
	else
		MvpLabel.Text = "👑 PARTY MVP 👑\n없음"
	end

	-- 3. 기본 골드 보상 텍스트 적용
	RewardLabel.Text = "💰 기본 처치 보상: +" .. resultPayload.BaseReward .. " Gold"
```
	-- 4. 본인이 MVP일 경우에만 전용 보너스 보상 레이블 활성화
	if resultPayload.MvpId == localPlayer.UserId then
		local mvpBonus = math.floor(resultPayload.BaseReward * 0.3)
		MvpRewardLabel.Text = "👑 MVP 추가 보너스: +" .. mvpBonus .. " Gold 달성!"
		MvpRewardLabel.Visible = true
	else
		MvpRewardLabel.Visible = false
	end

	-- 5. 최종 파티원 기여도 리포트 텍스트 정렬 및 출력
	-- 기존에 있던 오래된 텍스트 목록 초기화
	for _, child in pairs(ReportContainer:GetChildren()) do
		if child:IsA("TextLabel") then child:Destroy() end
	end

	-- 대미지 정렬을 위한 배열 생성
	local finalArray = {}
	local grandTotalDamage = 0
	for userId, data in pairs(resultPayload.FinalTable) do
		grandTotalDamage = grandTotalDamage + data.Damage
		table.insert(finalArray, data)
	end
	table.sort(finalArray, function(a, b) return a.Damage > b.Damage end)

	-- 정렬된 순서대로 텍스트 레이블 생성하여 리포트 컨테이너에 배치
	for rank, data in ipairs(finalArray) do
		local share = 0
		if grandTotalDamage > 0 then
			share = math.floor((data.Damage / grandTotalDamage) * 100)
		end
		
		local rankLabel = Instance.new("TextLabel")
		rankLabel.Text = string.format("■ %d등: %s -- %d Damage (기여도 %d%%)", rank, data.Name, data.Damage, share)
		rankLabel.Size = UDim2.new(1, 0, 0, 25)
		rankLabel.BackgroundTransparency = 1
		rankLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
		rankLabel.Font = Enum.Font.SourceSansBold
		rankLabel.TextSize = 16
		rankLabel.TextXAlignment = Enum.TextXAlignment.Left
		rankLabel.Parent = ReportContainer
	end

	-- 6. 결과창 프레임 서서히 나타나는 트윈 연출 실행 및 마우스 해제
	ResultFrame.Size = UDim2.new(0, 0, 0, 0) -- 초기 크기 0
	ResultFrame.Visible = true
	ResultFrame:TweenSize(
		UDim2.new(0, 600, 0, 450), -- 목표 크기
		Enum.EasingDirection.Out,
		Enum.EasingStyle.Back,
		0.5,
		true
	)
end)

-- 7. [메인 로비로 돌아가기] 버튼 기능 바인딩
ResultFrame.LobbyReturnButton.MouseButton1Click:Connect(function()
	-- 버튼 중복 클릭 방지 차단
	ResultFrame.LobbyReturnButton.Active = false
	print("클라이언트: 로비로 귀환 통신 요청 전송 중...")
	-- 별도의 TeleportService RemoteEvent를 호출하여 로비로 전송하는 백엔드 연동 처리
end)

