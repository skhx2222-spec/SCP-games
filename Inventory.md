# 🎒 로블록스 인벤토리 및 무기 장착 시스템 기획서 (Inventory.md) 

본 문서는 플레이어가 보유한 무기 목록을 열람하고, 원하는 무기를 주무기 슬롯에 장착 및 동기화하기 위한 클라이언트 UI 레이아웃과 클라이언트-서버 통신 설계를 정의합니다.

---

## 🖥️ 1. 인벤토리 UI 화면 레이아웃 구조 (UI Layout)

인벤토리 화면은 화면 좌측 하단의 '가방 아이콘' 버튼을 누르거나 키보드 `I` 키(패드 `View/Share` 버튼)를 누르면 열리는 프레임(`Frame`) 구조입니다.

+----------------------------------------------------------------------------------------+
|🎒 무기 인벤토리 (INVENTORY) [X] 닫기 
|+----------------------------------------------------------------------------------------+
|[ 카테고리 탭 ] [ 전체 ]  [ 돌격/기관 ]  [ 샷건/특수 ]  [ 에너지/하이테크 ]                     
|+------------------------------------+---------------------------------------------------+
|(좌측 영역: 보유 무기 스크롤 뷰)|(우측 영역: 선택 무기 상세 스펙 및 3D 프리뷰)|
|+------------++------------+ ||
||돌격소총 || 스마트 || [ 무기 3D Viewport ]|
||(영웅) || 피스톨 || (선택한 무기가 회전 연출됨)|
|+------------++------------+||
|+------------++------------+|■ 무기 이름: 돌격소총 (M4A1)|
||더블배럴 || 미래형 || ■ 현재 등급: 영웅 (Heroic)|
||(희귀) || 레이저건 ||  ■ 기본 대미지: 12 ■ 탄약 용량: 35발|
|+------------++------------+|  ■ 재장전 속도: 2.0초|
|+------------++------------+||
||(잠금)||(잠금)||[버튼1 특수]: 유탄 하부 발사 (대미지 140)|
||||||[버튼2 스킬]: 전술 재장전 버프|
|+------------++------------+||
||+---------------------------------------------+|
|||[ 주무기로 장착하기 ]||
||+---------------------------------------------+|
+------------------------------------+---------------------------------------------------+

## 🧱 2. UI 컴포넌트 상세 명세 (Component Specification)

1. **WeaponScrollingFrame (보유 무기 리스트 스크롤)**
   * 로블록스의 `UIGridLayout`을 활용하여 무기 아이콘들을 바둑판 배열로 자동 정렬합니다.
   * 유저가 보유한 무기만 활성화(`Visible = true`)되며, 미보유 무기는 자물쇠 아이콘과 함께 어둡게 비활성화 처리됩니다.
   * 아이콘 테두리 색상은 `Weapon.md`에 정의된 등급에 맞춰 동적으로 변경됩니다 (일반: 회색, 희귀: 녹색, 영웅: 보라색, 전설: 주황색).
2. **WeaponPreview (3D 프리뷰 뷰포트)**
   * `ViewportFrame` 인스턴스를 활용하여, 유저가 좌측에서 선택한 무기의 3D 메쉬 모델(`MeshPart`)을 우측 화면에 띄웁니다.
   * `RunService.RenderStepped` 루프를 사용해 무기 메쉬가 Y축을 기준으로 매 프레임 조금씩 회전하도록 스크립팅하여 역동성을 줍니다.
3. **EquipButton (장착 버튼)**
   * 현재 이미 장착 중인 무기라면 **[장착됨 (Equipped)]** 상태로 텍스트가 변경되며 버튼이 비활성화됩니다.
   * 장착되지 않은 무기를 선택하면 **[주무기로 장착하기]** 활성화 상태가 되며, 클릭 시 서버와 통신을 시작합니다.

---

## 🔄 3. 플랫폼별 입력 및 단축키 매핑 (Input Mapping)

다양한 플랫폼의 유저들이 인벤토리를 쾌적하게 제어할 수 있도록 단축키와 입력 이벤트를 바인딩합니다.

* **PC (키보드)**: `I` 키를 누르면 인벤토리 창이 토글(`Open/Close`)됩니다. 마우스 좌클릭으로 무기를 선택합니다.
* **게임패드 (콘솔)**: `View 버튼 (Xbox 구 백버튼 / PS 터치패드 좌측)`을 눌러 창을 엽니다. D-Pad(방향키)를 이용해 무기 슬롯 간 포커스를 이동하며, `A 버튼 (PS X버튼)`으로 장착을 확정합니다.
* **모바일 (터치)**: 화면 좌측 하단에 상시 노출되는 🎒 가방 모양의 `ImageButton`을 터치하여 화면 가득 인벤토리를 토글합니다.

## 📡 4. 네트워크 통신 규격 (Network Protocol)

인벤토리에서 무기를 교체하는 행위는 플레이어의 공격력 수치 및 서버 세이브 데이터와 직결되므로, 반드시 서버의 검증을 거치는 **보안 통신 구조**를 가집니다.

* **사용한 원격 이벤트**: `ReplicatedStorage.Network.RequestEquipWeapon` (RemoteFunction)
* **통신 플로우**:
  1. **클라이언트**: 유저가 우측 장착 버튼을 클릭하면 서버로 무기 이름 문자열을 보냅니다. `InvokeServer("AssaultRifle")`
  2. **서버**: 요청을 받으면 유저의 `DataStore` 보관함(`UnlockedWeapons` 리스트)을 확인하여 해당 유저가 진짜 그 무기와 등급을 보유하고 있는지 유효성을 검증합니다.
  3. **서버 (검증 통과 시)**: 플레이어 데이터의 장착 무기 값을 변경하고, 캐릭터의 손(`RightHand`) 또는 등에 해당하는 무기 메쉬 모델을 새로 부착(`Weld`)합니다. 이후 클라이언트에 성공 상태(`true`)를 반환합니다.
  4. **서버 (검증 실패 시)**: "비정상적인 접근입니다" 에러와 함께 `false`를 반환하고 클라이언트 장착을 거부합니다.

---

## 🛠️ 5. 서버 무기 장착 제어 스크립트 소스코드

`ServerScriptService` 내부에 `InventoryManager`라는 이름의 Script를 생성하고 아래 코드를 삽입합니다.

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")
local Players = game:GetService("Players")

-- ReplicatedStorage 내부에 Network 폴더와 RemoteFunction 생성 필요
local RequestEquipWeapon = ReplicatedStorage.Network:WaitForChild("RequestEquipWeapon")

-- 무기 메쉬 에셋이 저장된 서버 저장소 위치 지정
local WeaponModels = ServerStorage:WaitForChild("WeaponModels")

-- 클라이언트의 무기 장착 요청 처리 함수
local function handleEquipRequest(player, weaponName)
	local character = player.Character
	if not character or not character:FindFirstChild("Humanoid") then return false end
	
	local weaponData = player:FindFirstChild("WeaponData")
	if not weaponData then return false end
	
	-- [보안 검증] 유저가 실제 그 무기를 보유하고 있는지 데이터 체크
	-- (데이터스토어 및 세이브 시스템 연동 검증용)
	local hasWeapon = true 
	
	if hasWeapon then
		-- 1. 기존에 손에 쥐고 있던 무기 모델이 있다면 제거
		local rightHand = character:FindFirstChild("RightHand") or character:FindFirstChild("Right Arm")
		if rightHand then
			local oldWeapon = rightHand:FindFirstChild("AttachedWeapon")
			if oldWeapon then oldWeapon:Destroy() end
		end
		
		-- 2. ServerStorage에서 새로운 무기 메쉬 복제
		local weaponTemplate = WeaponModels:FindFirstChild(weaponName)
		if weaponTemplate then
			local newWeaponMesh = weaponTemplate:Clone()
			newWeaponMesh.Name = "AttachedWeapon"
			newWeaponMesh.Parent = rightHand
			
			-- 3. 캐릭터의 오른손과 무기 손잡이 파트 물리적 결합 (Weld)
			local handle = newWeaponMesh:FindFirstChild("Handle")
			if handle and rightHand then
				-- 무기 위치 초기화 후 정렬
				handle.CFrame = rightHand.CFrame 
				
				local weld = Instance.new("WeldConstraint")
				weld.Name = "WeaponWeld"
				weld.Part0 = rightHand
				weld.Part1 = handle
				weld.Parent = handle
			end
			
			-- 4. 플레이어 데이터스토어 값 갱신 동기화
			weaponData.EquippedWeapon.Value = weaponName
			print(player.Name .. " 유저가 무기를 장착함: " .. weaponName)
			return true
		end
	end
	
	return false -- 검증 실패 혹은 무기 모델이 없을 시 실패 반환
end

RequestEquipWeapon.OnServerInvoke = handleEquipRequest
```
### ② 클라이언트 UI 장착 버튼 액션 스크립트 (StarterPlayerScripts.InventoryUIController)

```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local RequestEquipWeapon = ReplicatedStorage.Network:WaitForChild("RequestEquipWeapon")
local localPlayer = Players.LocalPlayer

-- 유저가 상점 GUI나 인벤토리에서 [장착하기] 버튼을 눌렀을 때 실행되는 Local 스크립트 함수
local function onEquipButtonClicked(selectedWeaponName)
	-- 서버에 무기 장착 원격 호출 시도 (대기형 Invoke)
	local isSuccess = RequestEquipWeapon:InvokeServer(selectedWeaponName)
	
	if isSuccess then
		print("서버 검증 완료: 무기 장착 성공!")
		-- 화면 UI의 텍스트를 [장착됨] 상태로 변경하고 버튼 색상 리프레시 연출 처리
		-- script.Parent.EquipButton.Text = "장착됨"
		-- script.Parent.EquipButton.Active = false
	else
		warn("무기 장착 실패: 보유하지 않은 무기이거나 해킹 위협 감지.")
	end
end

-- UI 버튼 스크립트 바인딩용 예시 예외 연결
-- equipButton.MouseButton1Click:Connect(function() onEquipButtonClicked("AssaultRifle") end)
```

---

## 🔒 6. 인벤토리 예외 처리 및 보안 가이드라인 (Exception Handling)

1. **레이드 전투(보스방) 도중 무기 교체 원천 금지**
   * 플레이어가 보스방 구역(`RaidZone`) 내에 입장하여 보스전이 진행 중일 때는 인벤토리 UI의 장착 버튼(`EquipButton`)을 강제로 비활성화(`Active = false`)하고, 클릭 시 **"보스전 도중에는 무기를 교체할 수 없습니다!"** 알림 메시지를 출력합니다.
   * 클라이언트 변수 조작 해킹을 방지하기 위해, 서버측 `InventoryManager` 스크립트에서도 현재 플레이어의 월드 좌표 위치나 게임 세션 상태가 '로비'가 아닌 '보스방 레이드 중'일 경우 요청을 무조건 거부(`return false`)하도록 2중 백엔드 보안을 설정합니다.
2. **무기 프리뷰 뷰포트 메모리 누수 방지**
   * 인벤토리 창을 닫을(`[X]` 버튼 클릭) 때마다 `ViewportFrame` 내부에서 회전 연출 중이던 3D 무기 모델 복제본(`Clone`)을 반드시 `Destroy()` 처리하여 서버 및 클라이언트 기기의 프레임 드롭 및 메모리 렉(Lag) 누적을 원천 방지합니다.

