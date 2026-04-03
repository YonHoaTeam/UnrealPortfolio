# 전투 시스템 단계별 구현 가이드 (의존성 순서 정리)

> Implementation_Guide.md 기반, 초보자용으로 순서 재정리  
> 아직 안 만든 것을 연결하라는 문제 해결됨

---

## 진행 상황

| Phase | 작업 | 상태 |
|-------|------|------|
| A-1 | WBP_EnemyHP (위젯) | **완료** |
| A-2 | E_BossState (Enum) | **완료** |
| B | BP_EnemyBase 내부 구현 | **완료** |
| C-1~2 | BP_MonsterBasic 생성 + 변수 | **완료** |
| C-3 | BP_MonsterBasic BeginPlay | **완료** (DetectionRange→AttackRange 중복 버그 수정 필요) |
| C-4 | BP_MonsterBasic PerformAttack | **완료** |
| C-5 | ANS_AttackCheck 몽타주 연결 | **완료** |
| D-1 | BB_Monster (Blackboard) | **완료** |
| D-2 | BT_Monster (Behavior Tree) | **완료** |
| D-3 | BT Task 4종 + BTTask_UpdateTargetLocation | **완료** |
| D-4 | BT_Monster 트리 조립 | **완료** |
| D-5 | AIC_Monster (AI Controller) | **완료** |
| D-6 | BP_MonsterBasic에 AI Controller 설정 | **완료** |
| D-7 | ABP_Monster (적 애니메이션 블루프린트) | **완료** (Default Slot 추가) |
| D-8 | NavMesh + 테스트 | **완료** |
| D-9 | BP_MonsterSpawner (몬스터 스폰 시스템) | **완료** |
| E-1 | 플레이어 스태미너 (ConsumeStamina) | **완료** |
| E-2 | 플레이어 HP (Event AnyDamage) | **완료** |
| E-3 | 콤보 스태미너 연동 수정 | **완료** |
| E-4 | 플레이어 히트박스 시스템 | **완료** |
| E-5 | 적 히트박스 시스템 (BP_EnemyBase) | **완료** |
| E-6 | ANS_AttackCheck 플레이어 지원 추가 | **완료** |
| E-7 | 플레이어 몽타주 ANS_AttackCheck 부착 | **완료** |
| E-8 | BP_EnemyBase Event AnyDamage | **완료** |
| E-9 | BP_EnemyBase Die() 함수 수정 | **완료** |
| F | UI (WBP_MainHUD) | 미완 |
| G-1 | IA_Sprint 생성 | **완료** |
| G-2 | IMC_Game 매핑 변경 (LockOn→Ctrl, Dodge→Alt, Sprint→Shift) | **완료** |
| G-3 | BP_ToonCharacter 달리기 (Sprint) | **완료** (스태미너 소모 없이, WalkSpeed=300, SprintSpeed=800) |
| G-4 | BP_ToonCharacter 락온 변수 추가 | **완료** |
| G-5 | WBP_LockOnMarker 위젯 생성 | **완료** |
| G-6 | EnableLockOn / DisableLockOn 함수 | **완료** |
| G-7 | IA_LockOn 이벤트 연결 | **완료** |
| G-8 | Event Tick 락온 카메라 + UI 추적 | **완료** (Pitch 자유, 마우스 큰 입력 시 해제, 거리/사망 자동 해제) |
| G-9 | 플레이어 사망 시스템 | **완료** (bIsDead, ABP Dead 상태, DisableInput, WBP_GameOver) |
| G-10 | 피격 시 적 즉시 인지 (BB State→Chase + TargetLocation 세팅) | **완료** |
| G-11 | ABP BlendSpace 속도 조정 | **완료** |
| H-next | 강공격 넉백 + 이펙트 | 미완 (가이드 작성 완료: Phase_H_HeavyAttack_Knockback_Guide.md) |
| I | 보스 (BP_BossEnemy + AI) | 미완 |
| J | 게임 플로우 | 미완 |
| K | 미니맵 | 미완 |

### C-3 수정 필요 사항
- VariableSet_3과 VariableSet_4가 둘 다 AttackRange를 Set하고 있음
- VariableSet_4는 DetectionRange 값을 AttackRange에 덮어쓰는 버그
- **수정**: VariableSet_4 삭제 → VariableSet_3(AttackRange)의 then을 VariableSet_5(AttackCooldown)에 직접 연결

### D-3 가이드 변경사항
- BB_Monster에 **TargetLocation** (Vector) 키 추가
- **BTTask_UpdateTargetLocation** 추가 생성 (TargetActor 위치 → TargetLocation 벡터로 변환)
- 분기 2(Chase) MoveTo는 TargetActor(Object) 대신 **TargetLocation**(Vector) 사용
- 원인: UE5 MoveTo 노드가 Object 타입 Blackboard Key를 지원하지 않음

---

## C-5. ANS_AttackCheck 몽타주 연결

> 이미 ANS_AttackCheck가 존재하므로, 몽타주에 부착하는 방법만 설명

공격 몽타주 (예: AM_LightAttack1)를 더블클릭으로 열고:

1. 하단 **Notifies** 트랙 영역에서 빈 곳 우클릭
2. **Add Notify State** → `ANS_AttackCheck` 선택
3. 타격이 시작되는 프레임부터 끝나는 프레임까지 드래그해서 길이 조절
   - 보통 팔을 휘두르는 구간 (전체 애니메이션의 30%~70% 정도)

이렇게 하면 해당 구간에서 자동으로:
- 시작 → `OnAttackCheckStart()` 호출 (히트박스 ON)
- 끝 → `OnAttackCheckEnd()` 호출 (히트박스 OFF)

**Compile → Save**

---

## Phase D: 몬스터 AI 시스템

### D-1. Blackboard: BB_Monster

1. `Content/BluePrint/` 안에 `AI` 폴더 생성
2. `AI` 폴더에서 우클릭 → **Artificial Intelligence** → **Blackboard** → 이름: `BB_Monster`
3. 더블클릭으로 열기
4. **New Key** 버튼으로 키 추가:

| 순서 | + 클릭 → 타입 선택 | Key Name | 추가 설정 |
|------|-------------------|----------|----------|
| 1 | **Enum** | `State` | Key Type → Base Class: `E_MonsterState` |
| 2 | **Object** | `TargetActor` | Key Type → Base Class: `Actor` |
| 3 | **Vector** | `HomeLocation` | 없음 |
| 4 | **Vector** | `PatrolLocation` | 없음 |
| 5 | **Float** | `PatrolWaitTime` | 없음 |

**Enum 키 만드는 법 (State):**
1. New Key → **Enum** 선택
2. Key Name: `State`
3. 우측 Details → **Enum Type**: `E_MonsterState` 선택

5. **Save**

### D-2. Behavior Tree: BT_Monster

1. `Content/BluePrint/AI/` 에서 우클릭 → **Artificial Intelligence** → **Behavior Tree** → 이름: `BT_Monster`
2. 더블클릭으로 열기
3. 상단에 **Blackboard Asset** 드롭다운 → `BB_Monster` 선택

> Task를 먼저 만든 후(D-3) 트리를 조립(D-4)합니다.  
> BT_Monster는 열어만 두고, D-3으로 이동.

### D-3. BT Task 4종 생성

모두 `Content/BluePrint/AI/` 폴더에서 생성.  
생성법: 우클릭 → **Blueprint Class** → **All Classes** → `BTTask_BlueprintBase` 검색 → 선택

---

#### Task 1: BTTask_SetMonsterState

**변수 추가:**

| 변수명 | 타입 | 기본값 | Instance Editable |
|--------|------|--------|------------------|
| `NewState` | `E_MonsterState` | `Idle` | ☑ 체크 |

> **Instance Editable**을 켜면 BT 에디터에서 Task 노드마다 다른 값을 지정할 수 있음.  
> 변수 선택 → Details → 눈 모양 아이콘(👁) 클릭

**이벤트 그래프:**

```
Event Receive Execute AI (Owner Controller 핀)
 │
 ├─ Owner Controller → 드래그 → "Get Blackboard" 검색
 │
 ├─ Get Blackboard → 반환값 드래그 → "Set Value as Enum"
 │   - Key Name: "State"  (직접 문자열 입력)
 │   - Enum Value: NewState (변수 Get 드래그)
 │
 └─ Finish Execute (Success)
     (우클릭 → "Finish Execute" 검색 → Success 체크 ✓)
```

**Compile → Save**

---

#### Task 2: BTTask_RunEnemyAttack

**이벤트 그래프:**

```
Event Receive Execute AI (Owner Controller, Controlled Pawn)
 │
 ├─ Controlled Pawn → 드래그 → "Cast to BP_MonsterBasic"
 │
 ├─ 성공 → As BP_MonsterBasic → 드래그 → "PerformAttack" 호출
 │
 └─ Finish Execute (Success ✓)
```

**Compile → Save**

---

#### Task 3: BTTask_TurnToTarget

**이벤트 그래프:**

```
Event Receive Execute AI (Owner Controller, Controlled Pawn)
 │
 ├─① Owner Controller → Get Blackboard
 │    → "Get Value as Object" → Key Name: "TargetActor"
 │    → 반환값 → "Is Valid" 매크로 (우클릭 → "IsValid" 검색)
 │
 ├─ Is Valid?
 │   ├─ False → Finish Execute (Success: ✗ 체크 해제)
 │   └─ True ↓
 │
 ├─② Find Look at Rotation
 │    (우클릭 → "Find Look at Rotation" 검색)
 │    - Start: Controlled Pawn → Get Actor Location
 │    - Target: TargetActor → Get Actor Location
 │    → 결과 Rotator
 │
 ├─③ Break Rotator (결과를 분해)
 │    Yaw 값만 필요
 │    → Make Rotator:
 │      - Roll: 0
 │      - Pitch: 0  
 │      - Yaw: ②에서 나온 Yaw
 │
 ├─④ Controlled Pawn → Set Actor Rotation (③의 Make Rotator 결과)
 │
 └─ Finish Execute (Success ✓)
```

> **주의**: TargetActor는 Object 타입으로 반환되므로,  
> Get Actor Location을 쓰려면 "Cast to Actor" 또는 직접 Actor로 형변환 필요

**Compile → Save**

---

#### Task 4: BTTask_SelectPatrolLocation

**이벤트 그래프:**

```
Event Receive Execute AI (Owner Controller, Controlled Pawn)
 │
 ├─① Owner Controller → Get Blackboard
 │    → "Get Value as Vector" → Key Name: "HomeLocation"
 │    → HomeLocation 값
 │
 ├─② Get Random Reachable Point in Radius
 │    (우클릭 → "Get Random Reachable Point in Radius" 검색)
 │    ※ 안 보이면: "GetRandomPointInNavigableRadius" 시도
 │    - Origin: ①의 HomeLocation
 │    - Radius: 800
 │    → Random Location (Vector)
 │
 ├─③ Owner Controller → Get Blackboard
 │    → "Set Value as Vector"
 │    - Key Name: "PatrolLocation"
 │    - Vector Value: ②의 Random Location
 │
 └─ Finish Execute (Success ✓)
```

> **주의**: 이 노드가 작동하려면 레벨에 **NavMeshBoundsVolume**이 있어야 함!

**Compile → Save**

---

### D-4. BT_Monster 트리 조립

다시 `BT_Monster`를 열기. Task 4개가 모두 준비되었으므로 조립 가능.

**트리 구조 만들기:**

1. **Root** 노드에서 아래로 드래그 → **Selector** 추가
2. Selector에서 아래로 드래그 → 4개의 **Sequence** 추가 (왼쪽→오른쪽 순서 중요!)

#### 분기 1 (맨 왼쪽): Dead

1. 첫 번째 Sequence 우클릭 → **Add Decorator** → `Blackboard` 선택
2. Decorator 선택 → Details:
   - Blackboard Key: `State`
   - Key Query: `Is Equal To`
   - Enum Value: `Dead`
   - Observer Aborts: `Both`
3. Sequence 아래에 → **Wait** 추가 → Wait Time: `100`

#### 분기 2: Chase

1. Decorator → Blackboard: State == Chase, Observer Aborts: Both
2. Sequence 아래에:
   - **Move To** → Blackboard Key: `TargetActor`, Acceptable Radius: `150`
   - **BTTask_SetMonsterState** → NewState: `Attack`

#### 분기 3: Attack

1. Decorator → Blackboard: State == Attack, Observer Aborts: Both
2. Sequence 아래에 (왼→오른):
   - **BTTask_TurnToTarget**
   - **BTTask_RunEnemyAttack**
   - **Wait** → Wait Time: `2.0`
   - **BTTask_SetMonsterState** → NewState: `Chase`

#### 분기 4 (맨 오른쪽): Idle

1. Decorator → Blackboard: State == Idle, Observer Aborts: Both
2. Sequence 아래에:
   - **Wait** → Wait Time: `3.0`
   - **BTTask_SelectPatrolLocation**
   - **Move To** → Blackboard Key: `PatrolLocation`

**완성 모습:**
```
[Root]
 └─ [Selector]
     ├─ [Seq] Dec:State==Dead    → Wait 100
     ├─ [Seq] Dec:State==Chase   → MoveTo(TargetActor) → SetState(Attack)
     ├─ [Seq] Dec:State==Attack  → TurnToTarget → RunAttack → Wait 2 → SetState(Chase)
     └─ [Seq] Dec:State==Idle    → Wait 3 → SelectPatrol → MoveTo(PatrolLocation)
```

**Save**

---

### D-5. AI Controller: AIC_Monster

1. `Content/BluePrint/AI/` 에서 우클릭 → **Blueprint Class** → **All Classes** → `AIController` 검색 → 선택
2. 이름: `AIC_Monster`

#### 컴포넌트 추가

1. Components → **Add** → `AI Perception` 검색 → **AIPerception** 추가
2. AIPerception 선택 → Details:
   - **Senses Config** → **+** 클릭 → `AI Sight Config` 선택
   - AI Sight Config 펼치기:
     - Sight Radius: `1500`
     - Lose Sight Radius: `2000`
     - Peripheral Vision Half Angle: `60`
     - **Detection by Affiliation:**
       - Detect Enemies: ✓
       - Detect Neutrals: ✓ ← BP에서 Enemy만 설정 시 인지 안 되는 UE 버그 있음
       - Detect Friendlies: ✗
     - Auto Success Range from Last Seen Location: `500`

#### 변수 추가

| 변수명 | 타입 | 기본값 |
|--------|------|--------|
| `StateKey` | Name | `State` |
| `TargetActorKey` | Name | `TargetActor` |
| `HomeLocationKey` | Name | `HomeLocation` |

#### Event: On Possess

```
Event On Possess (Possessed Pawn)
 │  ※ 우클릭 → "Event On Possess" 검색
 │
 ├─① Use Blackboard
 │    (우클릭 → "Use Blackboard" 검색)
 │    - Blackboard Asset: BB_Monster
 │
 ├─② Run Behavior Tree
 │    (우클릭 → "Run Behavior Tree" 검색)
 │    - BT Asset: BT_Monster
 │
 ├─③ Self → "Get Blackboard" → 반환값에서:
 │
 │    ├─ "Set Value as Vector"
 │    │   Key Name: "HomeLocation"
 │    │   Vector Value: Possessed Pawn → Get Actor Location
 │    │
 │    ├─ "Set Value as Float"
 │    │   Key Name: "PatrolWaitTime"
 │    │   Float Value: 3.0
 │    │
 │    └─ "Set Value as Enum"
 │        Key Name: "State"
 │        Enum Value: E_MonsterState의 Idle (0)
```

**①→②→③ 순서대로 실행핀 연결**

#### Event: On Target Perception Updated

1. Components에서 **AIPerception** 선택
2. Details → Events → **On Target Perception Updated** 옆 **+** 클릭

```
On Target Perception Updated (Actor, Stimulus)
 │
 ├─ Actor → "Cast to BP_ToonCharacter"
 │   ├─ 실패 → Return
 │   └─ 성공 ↓
 │
 ├─ Stimulus → "Break AIStimulus" → Was Successfully Sensed (Bool)
 │
 ├─ Branch (Was Successfully Sensed)
 │   │
 │   ├─ True (플레이어 발견!)
 │   │   ├─ Get Blackboard → Set Value as Object
 │   │   │   Key: "TargetActor", Value: Actor
 │   │   └─ Get Blackboard → Set Value as Enum
 │   │       Key: "State", Value: Chase (1)
 │   │
 │   └─ False (시야 상실)
 │       ├─ Get Blackboard → Set Value as Object
 │       │   Key: "TargetActor", Value: None (빈 핀)
 │       └─ Get Blackboard → Set Value as Enum
 │           Key: "State", Value: Idle (0)
```

**Compile → Save**

---

### D-6. BP_MonsterBasic에 AI Controller 설정

1. `BP_MonsterBasic` 열기
2. 상단 **Class Defaults** 클릭
3. Details → **Pawn** 섹션:
   - **AI Controller Class**: `AIC_Monster`
   - **Auto Possess AI**: `Placed in World or Spawned`

**Compile → Save**

---

### D-7. ABP_Monster (적 애니메이션 블루프린트)

1. `Content/BluePrint/AI/` 에서 우클릭 → **Animation** → **Animation Blueprint**
2. 스켈레톤 선택 (BP_MonsterBasic에서 사용하는 Skeleton)
3. 이름: `ABP_Monster`

#### AnimGraph 구성

1. **State Machine** 추가 → 이름: `Locomotion`
2. State Machine 안에서:
   - **Idle** State: Idle 애니메이션
   - **Walk** State: Walk 애니메이션
   - Idle → Walk 전환 조건: Speed > 0
   - Walk → Idle 전환 조건: Speed <= 0

#### Event Graph에서 Speed 변수 업데이트

```
Event Blueprint Update Animation (DeltaTime)
 │
 ├─ Try Get Pawn Owner → Get Velocity → Vector Length → Set Speed
```

변수: `Speed` (Float)

#### BP_MonsterBasic에 ABP 적용

1. BP_MonsterBasic 열기
2. Mesh(SkeletalMesh) 컴포넌트 선택
3. Details → **Animation** → Anim Class: `ABP_Monster`

**Compile → Save**

---

### D-8. 테스트 준비

1. **NavMeshBoundsVolume** 배치 (필수!)
   - Place Actors → Volumes → Nav Mesh Bounds Volume
   - 전투 영역 전체를 덮도록 스케일 조절
   - `P` 키를 눌러 NavMesh 표시 확인 (초록색 = OK)

2. **BP_MonsterBasic을 레벨에 배치** (드래그)

3. **플레이 테스트**
   - 몬스터 Idle → 주변 정찰
   - 플레이어 접근 → Chase → Attack
   - 플레이어 멀어짐 → Idle 복귀

---

## Phase E: 플레이어 스태미너/HP

### BP_ToonCharacter 수정

#### 변수 추가

| 이름 | 타입 | 기본값 |
|------|------|--------|
| `CurrentStamina` | Float | `200` |
| `MaxStamina` | Float | `200` |
| `StaminaRegenRate` | Float | `30` |
| `StaminaRegenDelay` | Float | `1.5` |
| `bCanRegenStamina` | Boolean | `true` |
| `StaminaRegenTimerHandle` | Timer Handle | - |
| `CurrentHP` | Float | `500` |
| `MaxHP` | Float | `500` |
| `DodgeStaminaCost` | Float | `25` |
| `LightAttackStaminaCost` | Float | `20` |
| `HeavyAttackStaminaCost` | Float | `35` |
| `SprintStaminaCost` | Float | `15` |

#### Function: ConsumeStamina (Float Amount) → Boolean

1. Functions → **+** → 이름: `ConsumeStamina`
2. Inputs: `Amount` (Float)
3. Outputs: `ReturnValue` (Boolean)

```
ConsumeStamina(Amount) → Bool
 │
 ├─ Branch (CurrentStamina >= Amount)
 │   │
 │   ├─ True
 │   │   ├─ Set CurrentStamina = CurrentStamina - Amount
 │   │   ├─ Set bCanRegenStamina = false
 │   │   ├─ Clear and Invalidate Timer by Handle (StaminaRegenTimerHandle)
 │   │   ├─ Set Timer by Event (StaminaRegenDelay, Looping: false)
 │   │   │   └─ Custom Event "StartStaminaRegen"
 │   │   │       └─ Set bCanRegenStamina = true
 │   │   │   → Timer Handle 출력 → Set StaminaRegenTimerHandle
 │   │   └─ Return TRUE
 │   │
 │   └─ False
 │       └─ Return FALSE
```

#### Event Tick에서 스태미너 회복

```
Event Tick (DeltaTime)
 │
 ├─ Branch (bCanRegenStamina AND CurrentStamina < MaxStamina)
 │   ├─ True → Set CurrentStamina = Clamp(
 │   │           CurrentStamina + StaminaRegenRate * DeltaTime,
 │   │           0,
 │   │           MaxStamina)
 │   └─ False → (아무것도 안 함)
```

#### 기존 입력에 스태미너 체크 추가

**IA_Dodge:**
```
IA_Dodge Triggered
 ├─ ConsumeStamina(DodgeStaminaCost)
 │   ├─ True → (기존 구르기 로직)
 │   └─ False → (아무것도 안 함)
```

**IA_LightAttack:**
```
IA_LightAttack Triggered
 ├─ ConsumeStamina(LightAttackStaminaCost)
 │   ├─ True → (기존 약공격 로직)
 │   └─ False → Return
```

**IA_HeavyAttack:**
```
IA_HeavyAttack Triggered
 ├─ ConsumeStamina(HeavyAttackStaminaCost)
 │   ├─ True → (기존 강공격 로직)
 │   └─ False → Return
```

#### BeginPlay에 DataTable 로드 추가

```
Event BeginPlay (기존 로직 뒤에 추가)
 │
 ├─ Get Data Table Row (DT_PlayerStat, "1")
 │   ※ RowName은 DT_PlayerStat에서 사용하는 키 확인
 ├─ Break S_PlayerStat
 │   ├─ Set MaxHP, Set CurrentHP = MaxHP
 │   ├─ Set MaxStamina, Set CurrentStamina = MaxStamina
 │   ├─ Set StaminaRegenRate, Set StaminaRegenDelay
 │   └─ Set DodgeStaminaCost, LightAttackStaminaCost, HeavyAttackStaminaCost
```

#### Event AnyDamage 추가 (플레이어 피격)

```
Event AnyDamage (Damage)
 │
 ├─ Set CurrentHP = Clamp(CurrentHP - Damage, 0, MaxHP)
 ├─ Branch (CurrentHP <= 0)
 │   ├─ True → (사망 처리: 게임 오버 UI 등)
 │   └─ False → (히트 리액션 몽타주 재생, 선택)
```

---

### 콤보 스태미너 수정사항 (E-3)

기존 문제: IA_LightAttack Started에서 ConsumeStamina가 **무조건** 호출되어, 콤보 예약(bComboQueued)만 할 때도 스태미너가 빠짐

수정 후 흐름:
```
IA_LightAttack Started
 → Branch (IsAnyMontagePlaying?)
   ├─ True (몽타주 재생 중) → Branch (bCanCombo?)
   │   ├─ True → Set bComboQueued = true  ← 스태미너 안 빠짐
   │   └─ False → 무시
   └─ False (첫 공격)
       → ConsumeStamina(LightAttackStaminaCost)  ← 여기서만 소모
       → Branch → True → Set ComboIndex = 0 → Play Montage
```
※ 콤보가 실제 실행되는 곳에서도 ConsumeStamina 호출 필요

---

### 플레이어 히트박스 시스템 (E-4)

#### BP_ToonCharacter 추가 변수
| 이름 | 타입 | 기본값 |
|------|------|--------|
| `AttackDamage` | Float | `30` |
| `HitActorThisSwing` | Array (Actor) | - |

#### AttackHitbox 컴포넌트 설정
- SphereCollision, Radius: 50~80
- 기본 Collision: **No Collision** (공격 시에만 활성화)
- Generate Overlap Events: ✅

#### Function: OnAttackCheckStart
```
Entry → Array Clear(HitActorThisSwing) → AttackHitbox → Set Collision Enabled(Query Only)
```

#### Function: OnAttackCheckEnd
```
Entry → AttackHitbox → Set Collision Enabled(No Collision) → Array Clear(HitActorThisSwing)
```

#### Event: AttackHitbox → On Component Begin Overlap
```
On Component Begin Overlap (AttackHitbox)
 ├─ Branch (Other Actor == Self) → True: 끝
 ├─ HitActorThisSwing → Contains(Other Actor) → True: 끝
 ├─ HitActorThisSwing → Add(Other Actor)
 └─ Apply Damage (Damaged Actor: Other Actor, Base Damage: AttackDamage,
     Event Instigator: Get Controller, Damage Causer: Self)
```

---

### ANS_AttackCheck 수정 (E-6)

BP_EnemyBase + BP_ToonCharacter 양쪽 모두 지원하도록 수정:

```
Received_NotifyBegin
 ├─ MeshComp → Get Owner
 ├─ Cast to BP_EnemyBase → 성공 → OnAttackCheckStart() → Return true
 └─ Cast to BP_ToonCharacter → 성공 → OnAttackCheckStart() → Return true
     실패 → Return false

Received_NotifyEnd (동일 구조, OnAttackCheckEnd 호출)
```

---

### BP_EnemyBase Event AnyDamage (E-8)

```
Event AnyDamage (Damage)
 ├─ Branch (bIsDead) → True: 끝
 ├─ Clamp(CurrentHP - Damage, 0, MaxHP) → Set CurrentHP
 ├─ Branch (CurrentHP <= 0)
 │   ├─ True → Die()
 │   └─ False → Play Montage(AM_EnemyHitReact)
```

---

### BP_EnemyBase Die() 수정 (E-9)

```
Die()
 ├─ Set bIsDead = true
 ├─ GetAIController(Self) → GetBlackboard → Set Value as Enum (State = Dead)
 │   ※ GetAIController의 ControlledActor에 Self 연결 필수
 │   ※ State 로컬변수 기본값: "State" (None 아님)
 ├─ CapsuleComponent → Set Collision Enabled (No Collision)
 ├─ Play Anim Montage (AM_EnemyDie)
 ├─ Get GameMode → Cast to BP_GameMode → OnEnemyKilled(Self)
 ├─ Set Timer (3초, Delegate: DelayedDestroy)
```

---

### 기타 수정사항

- **ABP_Monster**: BlendSpacePlayer → **Default Slot** → Output Pose (몽타주 재생 필수)
- **BP_MonsterBasic PerformAttack**: bCanAttack 기본값 **true**, PlayAnimMontage Target에 **Self** 연결
- **적 카메라 충돌 방지**: BP_EnemyBase CapsuleComponent → Camera 채널 **Ignore**
- **Outline SkeletalMesh 동기화**: BeginPlay에서 `Set Leader Pose Component` (메인 메시 지정)

---

### 미해결 이슈

- **BT Patrol 중 Chase 전환 지연**: Decorator는 Both로 설정되어 있으나, Patrol MoveTo 중 즉시 중단되지 않는 현상. AIPerception 감지 타이밍 또는 BT 관련 추가 조사 필요.
- **강공격 개별 데미지**: HeavyAttack 1~3 각각 다른 데미지 적용 (미구현)
- **AttackDamage DataTable 연동**: DT_PlayerStat에서 AttackDamage 로드 (미구현)

---

## Phase F: UI 시스템

### F-1. WBP_MainHUD (User Widget)

**생성:** `Content/BluePrint/UI/` → 우클릭 → User Interface → Widget Blueprint → `WBP_MainHUD`

#### Designer 탭 레이아웃

```
[Canvas Panel]
 │
 ├─ [Vertical Box] ── Anchor: Top Left, Position: (20, 20)
 │   ├─ [Horizontal Box] "HP Section"
 │   │   ├─ Text "HP" (Font Size 14, White)
 │   │   └─ ProgressBar "HPBar" (300x25, Fill: Red)
 │   │       ☑ Is Variable 체크
 │   ├─ [Spacer] (Size Y: 5)
 │   └─ [Horizontal Box] "Stamina Section"
 │       ├─ Text "ST" (Font Size 14, White)
 │       └─ ProgressBar "StaminaBar" (250x18, Fill: Green)
 │           ☑ Is Variable 체크
 │
 ├─ Text "MissionText" ── Anchor: Top Center, Position: (0, 30)
 │   Font Size 18, White, ☑ Is Variable
 │
 ├─ [Vertical Box] "BossHPSection" ── Anchor: Top Center, Position: (0, 60)
 │   ☑ Is Variable, Visibility: Hidden (기본)
 │   ├─ Text "BossNameText" (Font Size 16, Center)
 │   │   ☑ Is Variable
 │   └─ ProgressBar "BossHPBar" (500x30, Fill: Red)
 │       ☑ Is Variable
 │
 └─ [Border] ── Anchor: Top Right, Position: (-220, 20), Size: 200x200
     └─ Image "MinimapImage"
         ☑ Is Variable (나중에 RT_Minimap 연결)
```

> **Is Variable 체크**: 위젯을 그래프에서 참조하려면 반드시 체크해야 함.
> 위젯 선택 → 이름 왼쪽의 체크박스 ✓

#### Graph 탭

**변수 추가:**

| 변수명 | 타입 |
|--------|------|
| `PlayerRef` | BP_ToonCharacter (Object Reference) |
| `BossRef` | BP_EnemyBase (Object Reference) |

**Event Construct:**
```
Event Construct
 ├─ Get Player Character → Cast to BP_ToonCharacter → Set PlayerRef
 ├─ Set Timer by Event (0.03초, Looping: true) → "UpdateHUD" 커스텀 이벤트
```

**Custom Event: UpdateHUD**
```
UpdateHUD
 ├─ HPBar → Set Percent (PlayerRef.CurrentHP / PlayerRef.MaxHP)
 └─ StaminaBar → Set Percent (PlayerRef.CurrentStamina / PlayerRef.MaxStamina)
```

**Function: UpdateMissionText (String NewText)**
```
MissionText → Set Text (NewText)
```

**Function: ShowBossHP (BP_EnemyBase BossRef)**
```
├─ Set BossRef
├─ BossHPSection → Set Visibility (Visible)
├─ BossNameText → Set Text ("BOSS")
├─ Set Timer (0.03초, Looping) → BossHPBar.SetPercent(BossRef.CurrentHP / BossRef.MaxHP)
```

**Function: HideBossHP**
```
BossHPSection → Set Visibility (Hidden)
```

### F-2. WBP_LockOnMarker (User Widget)

**생성:** `Content/BluePrint/UI/` → Widget Blueprint → `WBP_LockOnMarker`

```
[Canvas Panel]
 └─ Text "◆" (White, Font Size 30, Anchor: Center)
```

### F-3. WBP_GameClear (User Widget)

**생성:** `Content/BluePrint/UI/` → Widget Blueprint → `WBP_GameClear`

```
[Canvas Panel]
 └─ [Overlay - Center]
     └─ Text "GAME CLEAR" (Font Size 60, Gold/Yellow, Anchor: Center)
```

선택: Animation 추가 (Designer에서 Animations → + → FadeIn 2초)

---

## Phase G: 락온 시스템

### G-1. Input Action 생성

`Content/BluePrint/Input/Actions/` → 우클릭 → Input → Input Action → `IA_LockOn`
- Value Type: `Digital (Bool)`

### G-2. IMC_Game에 매핑 추가

`IMC_Game` 열기 → IA_LockOn 추가 → Mouse Middle Button

### G-3. BP_ToonCharacter에 락온 변수 추가

| 이름 | 타입 | 기본값 |
|------|------|--------|
| `bIsLockedOn` | Boolean | `false` |
| `LockOnTarget` | Actor Reference | None |
| `LockOnRange` | Float | `2000` |
| `LockOnWidget` | WBP_LockOnMarker Reference | None |

### G-4. IA_LockOn 이벤트

```
IA_LockOn Started
 ├─ Branch (bIsLockedOn)
 │   ├─ True → Call DisableLockOn()
 │   └─ False → Call EnableLockOn()
```

### G-5. Function: EnableLockOn

```
EnableLockOn()
 │
 ├─ SphereOverlapActors
 │   - Location: GetActorLocation
 │   - Radius: LockOnRange
 │   - Object Types: Pawn
 │   - Filter: ActorHasTag("Enemy")
 │   → OutActors
 │
 ├─ Branch (OutActors is empty)
 │   ├─ True → Return
 │   └─ False ↓
 │
 ├─ BestTarget = None, BestDot = -1  (로컬 변수)
 ├─ For Each Actor in OutActors
 │   ├─ Cast to BP_EnemyBase → Skip if bIsDead
 │   ├─ DirectionToEnemy = (Enemy.Location - Camera.Location).Normalize
 │   ├─ CameraForward = GetPlayerCameraManager → Get Forward Vector
 │   ├─ DotProduct = Dot(CameraForward, DirectionToEnemy)
 │   ├─ Branch (DotProduct > BestDot)
 │   │   └─ BestDot = DotProduct, BestTarget = Actor
 │
 ├─ Branch (BestTarget valid)
 │   ├─ True
 │   │   ├─ Set LockOnTarget = BestTarget
 │   │   ├─ Set bIsLockedOn = true
 │   │   ├─ CharacterMovement → bOrientRotationToMovement = false
 │   │   ├─ CharacterMovement → bUseControllerDesiredRotation = true
 │   │   └─ Create Widget (WBP_LockOnMarker) → Attach to LockOnTarget → Set LockOnWidget
 │   └─ False → Return
```

### G-6. Function: DisableLockOn

```
DisableLockOn()
 ├─ Set bIsLockedOn = false
 ├─ Set LockOnTarget = None
 ├─ CharacterMovement → bOrientRotationToMovement = true
 ├─ CharacterMovement → bUseControllerDesiredRotation = false
 ├─ LockOnWidget → Remove from Parent
 ├─ Set LockOnWidget = None
```

### G-7. Event Tick에 락온 카메라 추가

```
Event Tick (기존 스태미너 회복 뒤에 추가)
 │
 ├─ Branch (bIsLockedOn AND LockOnTarget Valid AND NOT LockOnTarget.bIsDead)
 │   ├─ True
 │   │   ├─ TargetLocation = LockOnTarget.GetActorLocation + (0, 0, 100)
 │   │   ├─ LookAtRotation = Find Look at Rotation(GetActorLocation, TargetLocation)
 │   │   ├─ NewRotation = RInterpTo(GetControlRotation, LookAtRotation, DeltaTime, 10.0)
 │   │   ├─ SetControlRotation(NewRotation)
 │   │   │
 │   │   ├─ Distance = GetDistanceTo(LockOnTarget)
 │   │   ├─ Branch (Distance > LockOnRange * 1.5)
 │   │   │   └─ True → DisableLockOn()
 │   │   │
 │   └─ False
 │       ├─ Branch (bIsLockedOn) → DisableLockOn() (타겟 사망/무효)
```

---

## Phase H: 보스 (BP_BossEnemy + 보스 AI)

### H-1. BP_BossEnemy 생성

`Content/BluePrint/Enemy/` → BP_EnemyBase 상속 → `BP_BossEnemy`

#### 추가 컴포넌트 (부위별 히트박스)

SkeletalMesh 아래에 CapsuleComponent 추가:

| 이름 | Attach To Bone | Tag |
|------|---------------|-----|
| `Hitbox_Head` | head | `Head` |
| `Hitbox_Body` | spine_02 | `Body` |
| `Hitbox_LeftArm` | upperarm_l | `LeftArm` |
| `Hitbox_RightArm` | upperarm_r | `RightArm` |
| `Hitbox_Legs` | thigh_l | `Legs` |

각 히트박스: Collision Preset = Custom, Object Type = Pawn

#### 추가 변수

| 이름 | 타입 | 기본값 |
|------|------|--------|
| `BossTid` | Integer | `2001` |
| `SkillsDataTable` | DataTable Reference | DT_BossSkill |
| `CurrentPhase` | Integer | `1` |
| `bPhaseTransitioning` | Boolean | `false` |
| `AvailableSkills` | Array of S_BossSkill | - |
| `CurrentSkillIndex` | Integer | `-1` |
| `Phase2HPThreshold` | Float | `0.5` |
| `DamageMultiplier_Head` | Float | `1.5` |
| `DamageMultiplier_Body` | Float | `1.0` |
| `DamageMultiplier_Arms` | Float | `0.8` |
| `DamageMultiplier_Legs` | Float | `0.7` |

#### Event BeginPlay

```
Event BeginPlay
 ├─ Call Parent
 ├─ Get Data Table Row (DT_BossStat, BossTid → String → Name)
 ├─ Break S_BossStat → Set MaxHP, CurrentHP, MoveSpeed, Phase2HPThreshold, 배율들
 ├─ HPBarWidget → Set Visibility (Hidden)  ← 보스는 머리 위 HP바 안 씀
 ├─ Load All Skills from DT_BossSkill where BossTid matches
 │   → For Each Row → Filter by BossTid → Add to AvailableSkills
 ├─ Tags → Add "Enemy", Add "Boss"
```

#### Function: SelectSkillByDistance (Float Distance) → Integer

```
SelectSkillByDistance(Distance) → SkillIndex
 ├─ FilteredIndices = 빈 배열
 ├─ For Each (AvailableSkills, Index)
 │   ├─ Branch: MinRange <= Distance <= MaxRange
 │   │   AND Phase <= CurrentPhase
 │   │   AND Skill not on cooldown
 │   │   └─ True → Add Index to FilteredIndices
 ├─ Branch (FilteredIndices empty)
 │   ├─ True → Return -1
 │   └─ False → Return Random from FilteredIndices
```

#### Function: ExecuteSkill (Integer SkillIndex)

```
ExecuteSkill(SkillIndex)
 ├─ SkillData = AvailableSkills[SkillIndex]
 ├─ Play Anim Montage (SkillData.AnimMontage)
 ├─ Set Timer (SkillData.Cooldown) → 쿨다운 해제
```

#### Function: CheckPhaseTransition

```
CheckPhaseTransition()
 ├─ Branch (CurrentPhase == 1 AND CurrentHP/MaxHP <= Phase2HPThreshold)
 │   ├─ True
 │   │   ├─ Set CurrentPhase = 2
 │   │   ├─ Set bPhaseTransitioning = true
 │   │   ├─ Play Phase Transition Montage (선택)
 │   │   ├─ CharacterMovement → MaxWalkSpeed *= 1.3
 │   │   └─ Set bPhaseTransitioning = false
 │   └─ False → Return
```

### H-2. BB_Boss (Blackboard)

`Content/BluePrint/AI/` → Blackboard → `BB_Boss`

| Key | Type | 설명 |
|-----|------|------|
| `State` | Enum (E_BossState) | 현재 상태 |
| `TargetActor` | Object (Actor) | 타겟 |
| `DistanceToTarget` | Float | 거리 |
| `SelectedSkillIndex` | Int | 선택된 스킬 |
| `CurrentPhase` | Int | 페이즈 |

### H-3. BT Task 추가 (보스 전용 3종)

#### BTTask_UpdateDistance

```
Execute AI
 ├─ Get BB TargetActor
 ├─ Distance = GetDistanceTo(TargetActor)
 ├─ Set BB DistanceToTarget = Distance
 └─ Finish Execute (Success)
```

#### BTTask_SelectBossSkill

```
Execute AI
 ├─ Cast to BP_BossEnemy
 ├─ Get BB DistanceToTarget
 ├─ Call SelectSkillByDistance(Distance) → Index
 ├─ Set BB SelectedSkillIndex = Index
 └─ Finish Execute (Success)
```

#### BTTask_ExecuteBossSkill

```
Execute AI
 ├─ Cast to BP_BossEnemy
 ├─ Get BB SelectedSkillIndex
 ├─ Call ExecuteSkill(Index)
 ├─ Wait for Montage End (또는 Delay)
 ├─ Set BB SelectedSkillIndex = -1
 └─ Finish Execute (Success)
```

### H-4. BT_Boss 트리

```
[Root]
 └─ [Selector]
     ├─ [Seq] State==Dead → Wait 100
     ├─ [Seq] State==PhaseTransition → Wait 3
     ├─ [Seq] State==Skill, Abort:Both
     │   → TurnToTarget → ExecuteBossSkill → Wait 1 → SetBossState(Chase)
     ├─ [Seq] State==Chase, Abort:Both
     │   → UpdateDistance → SelectBossSkill
     │   → [Selector]
     │       ├─ [Seq] Dec:SelectedSkillIndex >= 0
     │       │   → MoveTo(TargetActor) → SetBossState(Skill)
     │       └─ MoveTo(TargetActor, 200)
     └─ [Seq] State==Idle, Abort:Both → Wait 2
```

### H-5. AIC_Boss

AIC_Monster와 동일 패턴:
- BB: BB_Boss
- BT: BT_Boss
- Sight Radius: 3000 (보스는 넓은 감지)
- 초기 State: Idle, CurrentPhase: 1, SelectedSkillIndex: -1

---

## Phase I: 게임 플로우

### I-1. BP_MonsterSpawnPoint

`Content/BluePrint/` → Actor 상속 → `BP_MonsterSpawnPoint`
- BillboardComponent 추가 (에디터에서 보이게)
- 레벨에 3개 배치

### I-2. BP_BossSpawnPoint

동일하게 생성, 레벨에 1개 배치

### I-3. BP_CombatGameMode 수정

> 기존 BP_GameMode를 수정하거나, 새로 BP_CombatGameMode 생성

#### 변수

| 이름 | 타입 | 기본값 |
|------|------|--------|
| `AliveMonsterCount` | Integer | `0` |
| `TotalMonsterCount` | Integer | `3` |
| `bBossSpawned` | Boolean | `false` |
| `bGameCleared` | Boolean | `false` |
| `MonsterClass` | Class (BP_MonsterBasic) | BP_MonsterBasic |
| `BossClass` | Class (BP_BossEnemy) | BP_BossEnemy |
| `MonsterSpawnPoints` | Array of Actor | - |
| `BossSpawnPoint` | Actor Reference | - |
| `MainHUDWidget` | WBP_MainHUD Reference | - |

#### Event BeginPlay

```
├─ Get All Actors of Class (BP_MonsterSpawnPoint) → MonsterSpawnPoints
├─ Get All Actors of Class (BP_BossSpawnPoint) → BossSpawnPoint (첫 번째)
├─ Create Widget (WBP_MainHUD) → Add to Viewport → Set MainHUDWidget
├─ SpawnMonsters()
├─ SpawnActor (BP_MinimapCapture, Location: 0,0,5000)
```

#### Function: SpawnMonsters

```
For Each SpawnPoint in MonsterSpawnPoints
 ├─ SpawnActor (MonsterClass, SpawnPoint.GetTransform)
 ├─ AliveMonsterCount++
Update Mission UI ("몬스터 처치: 0/" + TotalMonsterCount)
```

#### Function: OnEnemyKilled (Actor KilledEnemy)

```
├─ Branch (KilledEnemy has Tag "Boss")
│   ├─ True → OnBossKilled()
│   └─ False
│       ├─ AliveMonsterCount--
│       ├─ Update Mission Text
│       ├─ Branch (AliveMonsterCount <= 0 AND NOT bBossSpawned)
│       │   └─ True → SpawnBoss()
```

#### Function: SpawnBoss

```
├─ Set bBossSpawned = true
├─ Update Mission Text ("보스 출현!")
├─ Delay 2초
├─ SpawnActor (BossClass, BossSpawnPoint.GetTransform)
├─ Delay 1초
├─ Update Mission Text ("보스를 처치하라!")
├─ MainHUDWidget → ShowBossHP(SpawnedBoss)
```

#### Function: OnBossKilled

```
├─ Set bGameCleared = true
├─ MainHUDWidget → HideBossHP()
├─ Update Mission Text ("게임 클리어!")
├─ Delay 2초
├─ Create Widget (WBP_GameClear) → Add to Viewport
```

### I-4. Die() 함수 연결 (BP_EnemyBase)

> 이제 GameMode가 있으므로, 이전에 보류했던 Die() 함수에 추가:

```
Die() 에서 ⑤번:
 ├─ Get Game Mode → Cast to BP_CombatGameMode
 │   → OnEnemyKilled(Self)
```

### I-5. World Settings

레벨 에디터 → World Settings → Game Mode Override = `BP_CombatGameMode`

---

## Phase J: 미니맵

### J-1. RT_Minimap (Render Target)

`Content/BluePrint/UI/` → 우클릭 → Materials & Textures → Render Target → `RT_Minimap`
- Size X: 512, Size Y: 512

### J-2. BP_MinimapCapture (Actor)

`Content/BluePrint/` → Actor 상속 → `BP_MinimapCapture`

#### 컴포넌트

- **SceneComponent** (Root)
- **SceneCaptureComponent2D** "MinimapCamera"
  - Projection Type: Orthographic
  - Ortho Width: 3000
  - Texture Target: RT_Minimap
  - Rotation: (-90, 0, 0) → 아래를 봄
  - Capture Every Frame: true

#### Event Tick

```
├─ Get Player Character → GetActorLocation
├─ Set Actor Location (PlayerX, PlayerY, PlayerZ + 5000)
```

### J-3. 미니맵 아이콘

BP_EnemyBase에 컴포넌트 추가:
- StaticMeshComponent "MinimapIcon"
  - Static Mesh: Plane
  - Material: 빨간색 Unlit
  - Location: (0, 0, 4500)
  - Scale: (0.3, 0.3, 0.3)
  - Collision: No Collision
  - 보스는 (0.5, 0.5, 0.5) + 보라색

### J-4. WBP_MainHUD 미니맵 연결

MinimapImage → Brush → Image = RT_Minimap

---

## 빠른 체크리스트

### 새로 만들 것 (남은 것만)

| # | 이름 | 타입 | Phase |
|---|------|------|-------|
| 1 | BB_Monster | Blackboard | D-1 |
| 2 | BT_Monster | Behavior Tree | D-2 |
| 3 | BTTask_SetMonsterState | BTTask | D-3 |
| 4 | BTTask_RunEnemyAttack | BTTask | D-3 |
| 5 | BTTask_TurnToTarget | BTTask | D-3 |
| 6 | BTTask_SelectPatrolLocation | BTTask | D-3 |
| 7 | AIC_Monster | AIController | D-5 |
| 8 | WBP_MainHUD | Widget | F-1 |
| 9 | WBP_LockOnMarker | Widget | F-2 |
| 10 | WBP_GameClear | Widget | F-3 |
| 11 | IA_LockOn | InputAction | G-1 |
| 12 | BP_BossEnemy | BP_EnemyBase 상속 | H-1 |
| 13 | BB_Boss | Blackboard | H-2 |
| 14 | BTTask_UpdateDistance | BTTask | H-3 |
| 15 | BTTask_SelectBossSkill | BTTask | H-3 |
| 16 | BTTask_ExecuteBossSkill | BTTask | H-3 |
| 17 | BT_Boss | Behavior Tree | H-4 |
| 18 | AIC_Boss | AIController | H-5 |
| 19 | BP_MonsterSpawnPoint | Actor | I-1 |
| 20 | BP_BossSpawnPoint | Actor | I-2 |
| 21 | RT_Minimap | RenderTarget | J-1 |
| 22 | BP_MinimapCapture | Actor | J-2 |

### 수정할 기존 것

| BP | 수정 | Phase |
|----|------|-------|
| BP_MonsterBasic | PerformAttack 함수, ANS 연결, AI Controller 설정 | C-4~D-6 |
| BP_ToonCharacter | 스태미너, HP, 락온, DataTable 로드 | E, G |
| BP_EnemyBase | Die()에 GameMode 연결, 미니맵 아이콘 | I-4, J-3 |
| IMC_Game | IA_LockOn 매핑 추가 | G-2 |
| BP_GameMode | GameMode 로직 전체 | I-3 |
| World Settings | GameMode Override | I-5 |
