# 💾 로블록스 레이드 저장 시스템 아키텍처 (DataStoreService.md)

본 문서는 플레이어의 소지 골드, 현재 장착 무기 종류, 무기 등급 및 전설 무기 강화 등급(+강) 데이터를 서버 이탈 시점에도 영구히 보호하는 데이터 퍼시스턴스 보안 연동 문서입니다.

---

## 🗃️ 저장 데이터 구조 스키마 (JSON 세이브 프로필)

각 유저 고유의 `Player.UserId`를 Key값으로 매핑하여 저장하는 가벼운 딕셔너리 테이블 구조입니다.

```json
{
  "Gold": 1250,
  "EquippedWeapon": "AssaultRifle",
  "WeaponTier": "Heroic",
  "UpgradeLevel": 0,
  "UnlockedWeapons": {
    "AssaultRifle": "Heroic",
    "Shotgun": "Rare",
    "SMG": "Common"
  }
}
```

---

## 🛠️ 서버 핵심 DataStore 스크립트 소스코드

`ServerScriptService` 내부에 배치할 스크립트 표준안입니다. 스튜디오 내 **[Game Settings -> Security -> Enable Studio Access to API Services]** 옵션 활성화가 필수적입니다.

```lua
local DataStoreService = game:GetService("DataStoreService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- 메인 세이브 데이터 스토어 이름 설정
local GameDataStore = DataStoreService:GetDataStore("PVERaid_SaveSystem_v1")

-- 데이터 로드 함수 (PlayerAdded)
local function onPlayerAdded(player)
	local leaderstats = Instance.new("Folder")
	leaderstats.Name = "leaderstats"
	leaderstats.Parent = player

	local gold = Instance.new("IntValue")
	gold.Name = "Gold"
	gold.Value = 0
	gold.Parent = leaderstats

	-- 내부 보관용 프로필 값 세팅
	local weaponData = Instance.new("Folder")
	weaponData.Name = "WeaponData"
	weaponData.Parent = player

	local currentWeapon = Instance.new("StringValue")
	currentWeapon.Name = "EquippedWeapon"
	currentWeapon.Value = "AssaultRifle" -- 기본 지급 무기 명시
	currentWeapon.Parent = weaponData

	local weaponTier = Instance.new("StringValue")
	weaponTier.Name = "WeaponTier"
	weaponTier.Value = "Common"
	weaponTier.Parent = weaponData

	local upgradeLevel = Instance.new("IntValue")
	upgradeLevel.Name = "UpgradeLevel"
	upgradeLevel.Value = 0
	upgradeLevel.Parent = weaponData

	-- 데이터스토어 백엔드 호출 (pcall 안전망 적용)
	local playerKey = "User_" .. player.UserId
	local success, savedData = pcall(function()
		return GameDataStore:GetAsync(playerKey)
	end)

	if success and savedData then
		-- 기존 유저인 경우 데이터 입력 복구
		gold.Value = savedData.Gold or 0
		currentWeapon.Value = savedData.EquippedWeapon or "AssaultRifle"
		weaponTier.Value = savedData.WeaponTier or "Common"
		upgradeLevel.Value = savedData.UpgradeLevel or 0
		print(player.Name .. " 데이터 로드 성공.")
	else
		print(player.Name .. " 신규 유저 또는 데이터 없음 기본값 적용.")
	end
end

-- 데이터 저장 함수 (PlayerRemoving)
local function onPlayerRemoving(player)
	if not player:FindFirstChild("leaderstats") or not player:FindFirstChild("WeaponData") then return end

	local playerKey = "User_" .. player.UserId
	
	-- 저장용 테이블 패킹
	local dataToSave = {
		Gold = player.leaderstats.Gold.Value,
		EquippedWeapon = player.WeaponData.EquippedWeapon.Value,
		WeaponTier = player.WeaponData.WeaponTier.Value,
		UpgradeLevel = player.WeaponData.UpgradeLevel.Value
	}

	-- 데이터 스토어 저장 시도 (pcall 자동 안전 제어)
	local success, err = pcall(function()
		GameDataStore:SetAsync(playerKey, dataToSave)
	end)

	if success then
		print(player.Name .. " 데이터 안전하게 클라우드 저장 완료.")
	else
		warn(player.Name .. " 데이터 저장 실패 에러 로그: " .. tostring(err))
	end
end

-- 서버 불시 셧다운 대응용 바인딩 (자동 전체 세이브 보호)
local function onBindToClose()
	for _, player in pairs(Players:GetPlayers()) do
		onPlayerRemoving(player)
	end
end

Players.PlayerAdded:Connect(onPlayerAdded)
Players.PlayerRemoving:Connect(onPlayerRemoving)
game:BindToClose(onBindToClose)
```

---

## 🔒 데이터 보안 및 트래픽 최적화 가이드라인

1. **상점 해킹 방지 (Server Authorization)**
   * 유저가 무기를 사거나 강화할 때, 로컬스크립트(클라이언트 GUI)에서 임의로 골드 값을 깎아 통신하면 안 됩니다.
   * 반드시 `RemoteFunction`을 이용해 서버에 "나 강화 원함" 신호를 보낸 뒤, **서버 스크립트 내부에서 직접 유저 골드를 검증하고 무기 데이터를 변경**해야 복사 및 치트 행위를 완벽히 차단할 수 있습니다.
2. **저장 제한 우수 우회법 (Data Throttling)**
   * 보스를 처치할 때마다 실시간으로 `SetAsync` 클라우드 함수를 강제 연속 구동하면 안 됩니다.
   * 게임 도중에는 플레이어 객체 내부의 `Value` 인스턴스 정보만 동적으로 실시간 누적 연산하고, **데이터스토어 호출은 유저가 방을 나갈 때(`PlayerRemoving`) 단 한 번만 수행**하도록 설계하여 트래픽 오버헤드를 제어해야 합니다.
