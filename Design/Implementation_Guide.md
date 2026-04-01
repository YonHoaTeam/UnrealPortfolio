# 전투 시스템 Blueprint 구현 가이드

## 목차
1. [CSV → DataTable 임포트](#1-csv--datatable-임포트)
2. [적 베이스 클래스 (BP_EnemyBase)](#2-bp_enemybase)
3. [일반 몬스터 (BP_MonsterBasic)](#3-bp_monsterbasic)
4. [몬스터 AI (AIC_Monster + BT)](#4-몬스터-ai)
5. [보스 (BP_BossEnemy)](#5-bp_bossenemy)
6. [보스 AI (AIC_Boss + BT)](#6-보스-ai)
7. [플레이어 스태미너](#7-플레이어-스태미너)
8. [락온 시스템](#8-락온-시스템)
9. [게임 플로우](#9-게임-플로우)
10. [UI 시스템](#10-ui-시스템)
11. [미니맵](#11-미니맵)

---

## 1. JSON → DataTable 임포트

### Tid 체계
| 범위 | 용도 |
|------|------|
| 1001~1999 | 일반 몬스터 (MonsterTid) |
| 2001~2999 | 보스 (BossTid) |
| 3001~3999 | 보스 스킬 |
| 9001~9999 | 텍스트 - Common (이름) |
| 10001~10999 | 텍스트 - Quest |
| 11001~11999 | 텍스트 - UI |

### 테이블 관계도
```
DT_Monster (MonsterTid, NameTextTid)
    ├─ DT_MonsterStat (MonsterTid → 조인)
    └─ DT_Text_Common (NameTextTid → 조인)

DT_Boss (BossTid, NameTextTid)
    ├─ DT_BossStat (BossTid → 조인)
    ├─ DT_BossSkill (BossTid → 조인, NameTextTid)
    └─ DT_Text_Common (NameTextTid → 조인)

DT_PlayerStat (Tid, Level → 레벨별 스탯)

DT_Text_Common  ─ 몬스터/보스/스킬 이름
DT_Text_Quest   ─ 퀘스트/미션 텍스트 ({0} 포맷 지원)
DT_Text_UI      ─ UI 라벨 텍스트
```

### DataTable용 Structure 생성 (9개)

Content Browser → `Content/Data/Structures/` → 우클릭 → Blueprints → Structure

#### S_Monster (DT_Monster용)
| 변수명 | 타입 | 설명 |
|---------|------|------|
| MonsterTid | Integer | 몬스터 고유 식별자 |
| NameTextTid | Integer | DT_Text_Common 조인키 |

#### S_MonsterStat (DT_MonsterStat용)
| 변수명 | 타입 | 설명 |
|---------|------|------|
| MonsterTid | Integer | DT_Monster 조인키 |
| MaxHP | Float | 최대 체력 |
| AttackDamage | Float | 공격력 |
| AttackRange | Float | 공격 사거리 (cm) |
| DetectionRange | Float | 감지 범위 (cm) |
| AttackCooldown | Float | 공격 쿨다운 (초) |
| MoveSpeed | Float | 이동 속도 |
| ExpReward | Integer | 경험치 보상 |

#### S_Boss (DT_Boss용)
| 변수명 | 타입 | 설명 |
|---------|------|------|
| BossTid | Integer | 보스 고유 식별자 |
| NameTextTid | Integer | DT_Text_Common 조인키 |

#### S_BossStat (DT_BossStat용)
| 변수명 | 타입 | 설명 |
|---------|------|------|
| BossTid | Integer | DT_Boss 조인키 |
| MaxHP | Float | 최대 체력 |
| Phase2HPThreshold | Float | 2페이즈 진입 HP 비율 (0.5 = 50%) |
| MoveSpeed | Float | 이동 속도 |
| DetectionRange | Float | 감지 범위 |
| DamageMultiplier_Head | Float | 머리 데미지 배율 |
| DamageMultiplier_Body | Float | 몸통 데미지 배율 |
| DamageMultiplier_Arms | Float | 팔 데미지 배율 |
| DamageMultiplier_Legs | Float | 다리 데미지 배율 |

#### S_BossSkill (DT_BossSkill용)
| 변수명 | 타입 | 설명 |
|---------|------|------|
| BossTid | Integer | DT_Boss 조인키 |
| NameTextTid | Integer | DT_Text_Common 조인키 (스킬 이름) |
| MinRange | Float | 최소 사용 거리 |
| MaxRange | Float | 최대 사용 거리 |
| Damage | Float | 데미지 |
| Cooldown | Float | 쿨다운 (초) |
| AnimMontage | Name | 애님 몽타주 이름 |
| HitboxType | Name | Sphere / Box |
| HitboxRadius | Float | 히트박스 반경 |
| HitboxLength | Float | 히트박스 길이 (Box용) |
| Phase | Integer | 사용 가능 페이즈 (1 또는 2) |

#### S_PlayerStat (DT_PlayerStat용)
| 변수명 | 타입 | 설명 |
|---------|------|------|
| Tid | Integer | 행 고유 식별자 |
| Level | Integer | 플레이어 레벨 |
| MaxHP | Float | 최대 HP |
| MaxStamina | Float | 최대 스태미너 |
| StaminaRegenRate | Float | 초당 회복량 |
| StaminaRegenDelay | Float | 소모 후 회복 시작 딜레이 (초) |
| AttackDamage | Float | 기본 공격력 |
| Defense | Float | 방어력 |
| DodgeStaminaCost | Float | 구르기 소모량 |
| LightAttackStaminaCost | Float | 약공격 소모량 |
| HeavyAttackStaminaCost | Float | 강공격 소모량 |
| SprintStaminaCost | Float | 초당 달리기 소모량 |

#### S_Text (DT_Text_Common, DT_Text_Quest, DT_Text_UI 공통)
| 변수명 | 타입 | 설명 |
|---------|------|------|
| TextTid | Integer | 텍스트 고유 식별자 |
| Korean | String | 한국어 |
| KoreanSub | String | 한국어 보조 (포맷 텍스트 등) |
| English | String | 영어 |

> 언어 추가 시 S_Text에 컬럼만 추가하면 됨 (예: Japanese, Chinese)

### 임포트 방법
1. Content Browser → `Content/Data/` 폴더로 이동
2. JSON 파일을 Content Browser로 드래그 앤 드롭
3. "DataTable Options" 창 → Row Type에서 해당 Structure 선택
4. Import 클릭
5. 텍스트 테이블 3개(Common, Quest, UI)는 모두 S_Text 구조체 사용

---

## 2. BP_EnemyBase

### 생성
Content Browser → `Content/BluePrint/Enemy/` → 우클릭 → Blueprint Class → Character 선택 → `BP_EnemyBase`

### 컴포넌트 추가
- (이미 있는) **CapsuleComponent** (Root)
- (이미 있는) **SkeletalMeshComponent**
- **WidgetComponent** → "HPBarWidget" (Class: WBP_EnemyHP, Space: Screen, Draw Size: 150x20)
- **SphereCollision** → "AttackHitbox" (기본 No Collision, 공격 시 활성화)

### 변수
| 이름 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| CurrentHP | Float | 0 | 현재 체력 |
| MaxHP | Float | 100 | 최대 체력 |
| bIsDead | Boolean | false | 사망 여부 |
| EnemyTag | Name | "Enemy" | 적 태그 (락온/미니맵용) |
| HitActorsThisSwing | Array (Actor) | - | 현재 공격에서 이미 맞은 대상 (중복 히트 방지) |

### 이벤트/함수

#### Event AnyDamage (이벤트 그래프)
```
Event AnyDamage
├─ Branch (bIsDead)
│  ├─ True → Return
│  └─ False
│     ├─ CurrentHP = Clamp(CurrentHP - Damage, 0, MaxHP)
│     ├─ Update HP Bar Widget (WBP_EnemyHP → SetPercent)
│     ├─ Branch (CurrentHP <= 0)
│     │  ├─ True → Call Die()
│     │  └─ False → Play Hit React Montage
```

#### Function: Die
```
Die()
├─ Set bIsDead = true
├─ Get AIController → Get Blackboard → Set Value as Enum (StateKey, Dead)
│  └─ BT가 Dead 분기로 전환되어 자연스럽게 정지
├─ Disable Collision (CapsuleComponent → Set Collision Enabled: No Collision)
├─ Play Death Montage
├─ Get GameMode → Cast to BP_CombatGameMode → OnEnemyKilled(Self)
├─ Set Timer (3초) → Destroy Actor
```

#### Function: OnAttackCheckStart (AnimNotifyState에서 호출)
```
OnAttackCheckStart()
├─ Clear HitActorsThisSwing 배열
├─ AttackHitbox → Set Collision Enabled (Query Only)
```

#### Function: OnAttackCheckEnd (AnimNotifyState에서 호출)
```
OnAttackCheckEnd()
├─ AttackHitbox → Set Collision Enabled (No Collision)
├─ Clear HitActorsThisSwing
```

#### Event: AttackHitbox → OnComponentBeginOverlap
```
OnBeginOverlap (OtherActor)
├─ Branch (OtherActor already in HitActorsThisSwing?)
│  ├─ True → Return (중복 히트 방지)
│  └─ False
│     ├─ Add OtherActor to HitActorsThisSwing
│     ├─ Cast to BP_ToonCharacter
│     │  └─ Success → Apply Damage (AttackDamage)
```

#### Function: InitFromDataTable (구현은 자식 클래스에서)
- 오버라이드 가능하게 설정

---

## 3. BP_MonsterBasic

### 생성
Content Browser → `Content/BluePrint/Enemy/` → 우클릭 → Blueprint Class → 부모 클래스: BP_EnemyBase → `BP_MonsterBasic`

### 추가 변수
| 이름 | 타입 | 설명 |
|------|------|------|
| MonsterDataRow | DataTable Row Handle | DT_MonsterStats 행 참조 |
| AttackDamage | Float | 공격 데미지 |
| AttackRange | Float | 공격 사거리 |
| AttackCooldown | Float | 쿨다운 |
| bCanAttack | Boolean | 공격 가능 여부 |

### Event BeginPlay
```
Event BeginPlay
├─ Get Data Table Row (DT_MonsterStats, RowName)
├─ Break S_MonsterStats
│  ├─ Set MaxHP, CurrentHP = MaxHP
│  ├─ Set AttackDamage, AttackRange, AttackCooldown
│  ├─ CharacterMovement → Set MaxWalkSpeed = MoveSpeed
├─ Set Tag "Enemy" (Tags → Add)
├─ Set bCanAttack = true
```

### Function: PerformAttack
```
PerformAttack()
├─ Branch (bCanAttack AND NOT bIsDead)
│  ├─ True
│  │  ├─ Set bCanAttack = false
│  │  ├─ Play Attack Montage
│  │  │  └─ 몽타주에 ANS_AttackCheck (AnimNotifyState) 부착
│  │  │     - NotifyBegin → OnAttackCheckStart() (히트박스 ON)
│  │  │     - NotifyEnd   → OnAttackCheckEnd()   (히트박스 OFF)
│  │  │     → 실제 데미지는 Overlap 이벤트에서 처리 (BP_EnemyBase)
│  │  ├─ Set Timer (AttackCooldown) → Set bCanAttack = true
│  └─ False → Return
```

### ANS_AttackCheck (AnimNotifyState 블루프린트)
Content Browser → `Content/BluePrint/Enemy/` → AnimNotifyState 상속 → `ANS_AttackCheck`

> 공격 애니메이션의 타격 구간에 배치. Notify 자체는 얇은 레이어로,
> 실제 로직은 소유 액터(BP_EnemyBase)에 위임하는 구조

```
Received_NotifyBegin (MeshComp, Animation, TotalDuration)
├─ MeshComp → GetOwner → Cast to BP_EnemyBase
│  ├─ 성공 → OnAttackCheckStart()
│  └─ 실패 → Return false
├─ Return true

Received_NotifyEnd (MeshComp, Animation)
├─ MeshComp → GetOwner → Cast to BP_EnemyBase
│  ├─ 성공 → OnAttackCheckEnd()
│  └─ 실패 → Return false
├─ Return true
```

---

## 4. 몬스터 AI

### 4-0. 상태 Enum: E_MonsterState
Content Browser → `Content/BluePrint/Enemy/` → 우클릭 → Blueprints → Enumeration → `E_MonsterState`

| Index | Name | 설명 |
|-------|------|------|
| 0 | Idle | 대기/정찰 |
| 1 | Chase | 추격 |
| 2 | Attack | 공격 |
| 3 | Dead | 사망 |

### 4-1. Blackboard: BB_Monster
Content Browser → `Content/BluePrint/AI/` → 우클릭 → Artificial Intelligence → Blackboard → `BB_Monster`

| Key Name | Type | 설명 |
|----------|------|------|
| State | Enum (E_MonsterState) | 현재 AI 상태 (**모든 BT 분기 조건**) |
| TargetActor | Object (Actor) | 추적/공격 대상 |
| HomeLocation | Vector | 스폰 위치 (정찰 복귀용) |
| PatrolLocation | Vector | 순찰 목표 위치 |
| PatrolWaitTime | Float | 정찰 대기 시간 |

### 4-2. Behavior Tree: BT_Monster
Content Browser → 우클릭 → Artificial Intelligence → Behavior Tree → `BT_Monster`

> **State 기반 Selector**: 각 분기마다 Decorator로 State 값을 체크.
> FlowAbortMode: Both → State 변경 시 현재 분기 즉시 중단 후 새 분기로 전환

```
[Root]
└─ [Selector]
    │
    ├─ [1] Sequence ── Decorator: State == Dead
    │   └─ Wait (100초) ── 사망 후 영구 대기 (Destroy 전까지)
    │
    ├─ [2] Sequence ── Decorator: State == Chase, Abort: Both
    │   ├─ BTTask_MoveTo (TargetActor, AcceptableRadius: 150)
    │   └─ BTTask_SetMonsterState → Attack
    │      └─ 접근 완료 시 공격 상태로 전환
    │
    ├─ [3] Sequence ── Decorator: State == Attack, Abort: Both
    │   ├─ BTTask_TurnToTarget (TargetActor)
    │   │  └─ 타겟 방향으로 회전 후 공격
    │   ├─ BTTask_RunEnemyAttack
    │   │  └─ PerformAttack() 호출 (몽타주 + AnimNotify 히트박스)
    │   ├─ Wait (공격 후 딜레이, AttackCooldown)
    │   └─ BTTask_SetMonsterState → Chase
    │      └─ 공격 후 다시 추격
    │
    └─ [4] Sequence ── Decorator: State == Idle, Abort: Both
        ├─ WaitBlackboardTime (PatrolWaitTime)
        ├─ BTTask_SelectPatrolLocation
        │  └─ HomeLocation 중심 랜덤 포인트 → BB PatrolLocation에 저장
        └─ BTTask_MoveTo (PatrolLocation)
```

### 4-3. AI Controller: AIC_Monster
Content Browser → `Content/BluePrint/AI/` → 우클릭 → Blueprint Class → AIController → `AIC_Monster`

#### 컴포넌트 추가
- **AIPerceptionComponent**
  - Config → Add: **AI Sight Config**
    - Sight Radius: 1500 (DataTable DetectionRange)
    - Lose Sight Radius: 2000
    - Peripheral Vision Half Angle: 60
    - Detection by Affiliation: Enemies ✓, Neutrals ✗, Friendlies ✗
    - Auto Success Range: 500

#### 변수 (BB 키 이름을 변수로 관리 — 오타 방지)
| 이름 | 타입 | 기본값 |
|------|------|--------|
| StateKey | Name | "State" |
| TargetActorKey | Name | "TargetActor" |
| HomeLocationKey | Name | "HomeLocation" |
| PatrolWaitTimeKey | Name | "PatrolWaitTime" |

#### Event ReceivePossess (PossessedPawn)
> BeginPlay 대신 Possess 시점에 초기화 — Pawn이 확정된 시점에서 실행

```
ReceivePossess(PossessedPawn)
├─ Sequence (3갈래 동시 실행)
│  ├─ [0] UseBlackboard (BB_Monster)
│  ├─ [1] RunBehaviorTree (BT_Monster)
│  └─ [2] Blackboard 초기값 세팅
│     ├─ Set HomeLocation = PossessedPawn → GetActorLocation
│     ├─ Set PatrolWaitTime = 3.0
│     └─ Set State = Idle (0)
```

#### AIPerception → On Target Perception Updated
```
On Target Perception Updated (Actor, Stimulus)
├─ Cast Actor to BP_ToonCharacter
│  ├─ Success
│  │  ├─ Branch (Stimulus → Successfully Sensed)
│  │  │  ├─ True
│  │  │  │  ├─ Set BB TargetActor = Actor
│  │  │  │  └─ Set BB State = Chase (1)
│  │  │  │     └─ FlowAbort: Both로 인해 BT가 즉시 Chase 분기로 전환
│  │  │  └─ False (시야 상실)
│  │  │     ├─ Set BB TargetActor = None
│  │  │     └─ Set BB State = Idle (0)
│  └─ Fail → Return
```

### 4-4. 커스텀 BT Tasks

#### BTTask_RunEnemyAttack
Content Browser → `Content/BluePrint/AI/` → BTTask_BlueprintBase 상속

```
Execute AI (OwnerController, ControlledPawn)
├─ Cast ControlledPawn to BP_MonsterBasic
├─ Call PerformAttack()
├─ Finish Execute (Success)
```

#### BTTask_SetMonsterState
> 파라미터로 NewState (E_MonsterState) 노출 — BT에서 인스턴스마다 다른 상태 지정

```
Execute AI (OwnerController, ControlledPawn)
├─ Get Blackboard → Set Value as Enum (StateKey, NewState)
├─ Finish Execute (Success)
```

#### BTTask_TurnToTarget
> BB의 TargetActor 방향으로 Pawn 회전

```
Execute AI (OwnerController, ControlledPawn)
├─ Get BB TargetActor
├─ Branch (Valid?)
│  ├─ True
│  │  ├─ Find Look at Rotation (Pawn → Target)
│  │  ├─ Set Actor Rotation (Yaw만)
│  │  └─ Finish Execute (Success)
│  └─ False → Finish Execute (Failure)
```

#### BTTask_SelectPatrolLocation
> HomeLocation 중심 반경 내 랜덤 포인트

```
Execute AI (OwnerController, ControlledPawn)
├─ Get BB HomeLocation
├─ GetRandomReachablePointInRadius (HomeLocation, 800)
├─ Set BB PatrolLocation = 결과
├─ Finish Execute (Success)
```

### BP_MonsterBasic에 AIC_Monster 설정
BP_MonsterBasic → Class Defaults → Details → AI Controller Class = `AIC_Monster`
Auto Possess AI = Placed in World or Spawned

---

## 5. BP_BossEnemy

### 생성
`Content/BluePrint/Enemy/` → BP_EnemyBase 상속 → `BP_BossEnemy`

### HP 표시 방식
> 보스는 머리 위 HP바(WidgetComponent) 사용 안 함.
> 대신 WBP_MainHUD의 **BossHPSection** (화면 상단 중앙, 엘든링 스타일)에 표시.

```
Event BeginPlay 에서:
├─ HPBarWidget (부모 BP_EnemyBase에서 상속) → Set Visibility (Hidden)
│  └─ 또는 Set Active (false)로 완전 비활성화
```

보스 HP 업데이트는 WBP_MainHUD에서 처리:
- 보스 스폰 시 → GameMode가 MainHUD.ShowBossHP(BossRef) 호출
- 보스 피격 시 → MainHUD가 BossRef.CurrentHP/MaxHP를 폴링하여 ProgressBar 갱신
- 보스 사망 시 → MainHUD.HideBossHP() 호출

### 추가 컴포넌트 (부위별 히트 판정)
SkeletalMesh 아래에 CapsuleComponent 추가:
| 컴포넌트 이름 | Attach To Bone | Tag | 용도 |
|---------------|---------------|-----|------|
| Hitbox_Head | head (또는 해당 본) | "Head" | 머리 판정 |
| Hitbox_Body | spine_02 | "Body" | 몸통 판정 |
| Hitbox_LeftArm | upperarm_l | "LeftArm" | 왼팔 |
| Hitbox_RightArm | upperarm_r | "RightArm" | 오른팔 |
| Hitbox_Legs | thigh_l | "Legs" | 다리 |

각 히트박스: Collision Preset = Custom, Object Type = Pawn, Block: WorldDynamic
주의: 메인 CapsuleComponent(루트)는 네비게이션/이동용으로 유지

### 추가 변수
| 이름 | 타입 | 기본값 |
|------|------|--------|
| BossDataRow | DataTable Row Handle | DT_BossStats 행 참조 |
| SkillsDataTable | DataTable Reference | DT_BossSkills |
| CurrentPhase | Integer | 1 |
| bPhaseTransitioning | Boolean | false |
| AvailableSkills | Array of S_BossSkills | - |
| CurrentSkillCooldowns | Map (Name → Float) | - |

### Event BeginPlay
```
Event BeginPlay
├─ Get Data Table Row (DT_BossStats, "Boss_Default")
├─ Set MaxHP, CurrentHP = MaxHP, MoveSpeed
├─ Load All Skills from DT_BossSkills where BossName matches
│  └─ Store in AvailableSkills array
├─ Set Tag "Enemy"
├─ Set Tag "Boss"
├─ Set AI Controller Class = AIC_Boss
```

### Function: SelectSkillByDistance (Float Distance) → S_BossSkills
```
SelectSkillByDistance(Distance)
├─ FilteredSkills = Filter AvailableSkills where:
│  ├─ MinRange <= Distance <= MaxRange
│  ├─ Phase <= CurrentPhase
│  └─ Skill is NOT on cooldown
├─ Branch (FilteredSkills is empty)
│  ├─ True → Return None (추적 모드로)
│  └─ False → Return Random from FilteredSkills
```

### Function: ExecuteSkill (S_BossSkills SkillData)
```
ExecuteSkill(SkillData)
├─ Play Anim Montage (SkillData.AnimMontage에 해당하는 몽타주)
├─ Set Skill Cooldown Timer (SkillData.SkillName, SkillData.Cooldown)
```

### 부위별 데미지 처리
기본 CapsuleComponent의 Hit Event 대신, 각 Hitbox_* 컴포넌트에서 처리:

**Event Hit / On Component Begin Overlap 사용 시:**
플레이어의 무기 트레이스가 부위별 히트박스와 충돌하면:
```
OnHit (HitComponent)
├─ Get Component Tag → DamageMultiplier 조회
│  ├─ "Head" → DamageMultiplier_Head (1.5x)
│  ├─ "Body" → DamageMultiplier_Body (1.0x)
│  ├─ "LeftArm"/"RightArm" → DamageMultiplier_Arms (0.8x)
│  └─ "Legs" → DamageMultiplier_Legs (0.7x)
├─ FinalDamage = BaseDamage * Multiplier
├─ CurrentHP -= FinalDamage
├─ CheckPhaseTransition()
```

### Function: CheckPhaseTransition
```
CheckPhaseTransition()
├─ Branch (CurrentPhase == 1 AND CurrentHP/MaxHP <= Phase2HPThreshold)
│  ├─ True
│  │  ├─ Set CurrentPhase = 2
│  │  ├─ Set bPhaseTransitioning = true
│  │  ├─ Play Phase Transition Montage (선택)
│  │  ├─ CharacterMovement → Set MaxWalkSpeed *= 1.3 (속도 증가)
│  │  ├─ Set bPhaseTransitioning = false
│  └─ False → Return
```

---

## 6. 보스 AI

### 6-0. 상태 Enum: E_BossState
Content Browser → `Content/BluePrint/Enemy/` → Enumeration → `E_BossState`

| Index | Name | 설명 |
|-------|------|------|
| 0 | Idle | 대기 |
| 1 | Chase | 추격 |
| 2 | Skill | 스킬 실행 중 |
| 3 | Dead | 사망 |
| 4 | PhaseTransition | 페이즈 전환 연출 |

### 6-1. Blackboard: BB_Boss

| Key Name | Type | 설명 |
|----------|------|------|
| State | Enum (E_BossState) | 현재 AI 상태 |
| TargetActor | Object (Actor) | 타겟 플레이어 |
| DistanceToTarget | Float | 타겟과의 거리 |
| SelectedSkillIndex | Int | 선택된 스킬 인덱스 |
| CurrentPhase | Int | 현재 페이즈 (1 or 2) |

### 6-2. Behavior Tree: BT_Boss

> 몬스터 BT와 동일한 State 기반 Selector 패턴이지만,
> 공격 분기가 거리 기반 스킬 선택으로 확장됨

```
[Root]
└─ [Selector]
    │
    ├─ [1] Sequence ── Decorator: State == Dead
    │   └─ Wait (100초)
    │
    ├─ [2] Sequence ── Decorator: State == PhaseTransition
    │   └─ Wait (3초) ── 페이즈 전환 몽타주 대기
    │
    ├─ [3] Sequence ── Decorator: State == Skill, Abort: Both
    │   ├─ BTTask_TurnToTarget (TargetActor)
    │   ├─ BTTask_ExecuteBossSkill
    │   │  └─ 스킬 몽타주 재생 + ANS_AttackCheck로 히트박스 처리
    │   ├─ Wait (1초, 스킬 후 딜레이)
    │   └─ BTTask_SetBossState → Chase
    │
    ├─ [4] Sequence ── Decorator: State == Chase, Abort: Both
    │   ├─ BTTask_UpdateDistance (거리 계산 → BB)
    │   ├─ BTTask_SelectBossSkill (거리 기반)
    │   ├─ [Selector]
    │   │  ├─ [Sequence] ── Decorator: SelectedSkillIndex >= 0 (스킬 선택됨)
    │   │  │  ├─ BTTask_MoveTo (TargetActor, 스킬 MaxRange까지)
    │   │  │  └─ BTTask_SetBossState → Skill
    │   │  └─ BTTask_MoveTo (TargetActor, 200) ── 스킬 없으면 접근
    │
    └─ [5] Sequence ── Decorator: State == Idle, Abort: Both
        └─ Wait (2초) ── 보스는 정찰 안 함, 대기만
```

### 6-3. AI Controller: AIC_Boss
AIC_Monster와 동일한 패턴:

```
ReceivePossess(PossessedPawn)
├─ Sequence
│  ├─ [0] UseBlackboard (BB_Boss)
│  ├─ [1] RunBehaviorTree (BT_Boss)
│  └─ [2] BB 초기값
│     ├─ Set State = Idle (0)
│     ├─ Set CurrentPhase = 1
│     └─ Set SelectedSkillIndex = -1
```

AIPerception:
- Sight Radius: 3000 (보스는 넓은 감지)
- 감지 시 → State = Chase, TargetActor 설정

### 6-4. BTTask_SelectBossSkill
```
Execute AI
├─ Cast to BP_BossEnemy
├─ Get BB DistanceToTarget
├─ Call SelectSkillByDistance(Distance) → SkillIndex
├─ Set BB SelectedSkillIndex = SkillIndex
├─ Finish Execute (Success)
```

### 6-5. BTTask_ExecuteBossSkill
```
Execute AI
├─ Cast to BP_BossEnemy
├─ Get BB SelectedSkillIndex
├─ Call ExecuteSkill(SkillIndex)
│  └─ 몽타주 재생 (ANS_AttackCheck 포함)
│     - Begin → OnAttackCheckStart() (히트박스 크기를 스킬 데이터에서 설정)
│     - End   → OnAttackCheckEnd()
├─ Wait for Montage End
├─ Set BB SelectedSkillIndex = -1
├─ Finish Execute (Success)
```

### 6-6. 보스 공격 히트박스
BP_BossEnemy는 BP_EnemyBase의 ANS_AttackCheck를 재사용하되,
스킬마다 히트박스 크기가 다르므로 OnAttackCheckStart를 오버라이드:

```
OnAttackCheckStart() [Override]
├─ Get Current Skill Data (AvailableSkills[CurrentSkillIndex])
├─ AttackHitbox → Set Sphere Radius (SkillData.HitboxRadius)
│  └─ 또는 Box용이면 Box Extent 설정
├─ Clear HitActorsThisSwing
├─ AttackHitbox → Set Collision Enabled (Query Only)
├─ AttackDamage = SkillData.Damage (이번 공격의 데미지 세팅)
```

---

## 7. 플레이어 스태미너

### BP_ToonCharacter 수정

#### 변수 추가
| 이름 | 타입 | 기본값 |
|------|------|--------|
| CurrentStamina | Float | 200 |
| MaxStamina | Float | 200 |
| StaminaRegenRate | Float | 30 |
| StaminaRegenDelay | Float | 1.5 |
| bCanRegenStamina | Boolean | true |
| StaminaRegenTimerHandle | Timer Handle | - |
| CurrentHP | Float | 500 |
| MaxHP | Float | 500 |

#### Function: ConsumeStamina (Float Amount) → Boolean
```
ConsumeStamina(Amount) → Bool
├─ Branch (CurrentStamina >= Amount)
│  ├─ True
│  │  ├─ CurrentStamina -= Amount
│  │  ├─ Set bCanRegenStamina = false
│  │  ├─ Clear Timer (StaminaRegenTimerHandle)
│  │  ├─ Set Timer by Event (StaminaRegenDelay)
│  │  │  └─ On Timer → Set bCanRegenStamina = true
│  │  └─ Return TRUE
│  └─ False
│     └─ Return FALSE (스태미너 부족)
```

#### Event Tick에서 스태미너 회복
```
Event Tick (DeltaTime)
├─ Branch (bCanRegenStamina AND CurrentStamina < MaxStamina)
│  ├─ True → CurrentStamina = Clamp(CurrentStamina + StaminaRegenRate * DeltaTime, 0, MaxStamina)
│  └─ False → Skip
```

#### 기존 입력에 스태미너 체크 추가
**IA_Dodge 입력 이벤트 수정:**
```
IA_Dodge Triggered
├─ ConsumeStamina(DodgeStaminaCost)
│  ├─ True → (기존 구르기 로직 실행)
│  └─ False → (아무것도 안 함 / 스태미너 부족 UI 피드백)
```

**IA_LightAttack 입력 이벤트 수정:**
```
IA_LightAttack Triggered
├─ ConsumeStamina(LightAttackStaminaCost)
│  ├─ True → (기존 약공격 로직)
│  └─ False → Return
```

**IA_HeavyAttack 입력 이벤트 수정:**
```
IA_HeavyAttack Triggered
├─ ConsumeStamina(HeavyAttackStaminaCost)
│  ├─ True → (기존 강공격 로직)
│  └─ False → Return
```

#### BeginPlay에 DataTable 로드 추가
```
Event BeginPlay
├─ Get Data Table Row (DT_PlayerStats, "Player_Default")
├─ Break S_PlayerStats
│  ├─ Set MaxHP, CurrentHP = MaxHP
│  ├─ Set MaxStamina, CurrentStamina = MaxStamina
│  ├─ Set StaminaRegenRate, StaminaRegenDelay
│  └─ Set DodgeStaminaCost, LightAttackStaminaCost, HeavyAttackStaminaCost
```

---

## 8. 락온 시스템

### Input Action 생성
Content Browser → `Content/BluePrint/Input/Actions/` → 우클릭 → Input → Input Action → `IA_LockOn`
- Value Type: Digital (Bool)

### IMC_Game에 매핑 추가
IA_LockOn → Mouse Middle Button

### BP_ToonCharacter에 락온 로직 추가

#### 변수
| 이름 | 타입 |
|------|------|
| bIsLockedOn | Boolean |
| LockOnTarget | Actor Reference |
| LockOnRange | Float (2000) |
| LockOnWidget | WBP_LockOnMarker Reference |

#### IA_LockOn 이벤트 (토글)
```
IA_LockOn Started
├─ Branch (bIsLockedOn)
│  ├─ True → Call DisableLockOn()
│  └─ False → Call EnableLockOn()
```

#### Function: EnableLockOn
```
EnableLockOn()
├─ SphereOverlapActors
│  ├─ Location: GetActorLocation
│  ├─ Radius: LockOnRange
│  ├─ Object Types: Pawn
│  ├─ Filter: ActorHasTag("Enemy")
│  └─ → OutActors
├─ Branch (OutActors is empty)
│  ├─ True → Return (적 없음)
│  └─ False
│     ├─ BestTarget = None, BestDot = -1
│     ├─ For Each Actor in OutActors
│     │  ├─ Skip if Actor.bIsDead
│     │  ├─ DirectionToEnemy = (Enemy.Location - Camera.Location).Normalize
│     │  ├─ CameraForward = Camera.GetForwardVector
│     │  ├─ DotProduct = Dot(CameraForward, DirectionToEnemy)
│     │  ├─ Branch (DotProduct > BestDot)
│     │  │  └─ BestDot = DotProduct, BestTarget = Actor
│     ├─ Branch (BestTarget valid)
│     │  ├─ True
│     │  │  ├─ Set LockOnTarget = BestTarget
│     │  │  ├─ Set bIsLockedOn = true
│     │  │  ├─ CharacterMovement → bOrientRotationToMovement = false
│     │  │  ├─ CharacterMovement → bUseControllerDesiredRotation = true
│     │  │  ├─ Create Widget (WBP_LockOnMarker) → Attach to LockOnTarget
│     │  └─ False → Return
```

#### Function: DisableLockOn
```
DisableLockOn()
├─ Set bIsLockedOn = false
├─ Set LockOnTarget = None
├─ CharacterMovement → bOrientRotationToMovement = true
├─ CharacterMovement → bUseControllerDesiredRotation = false
├─ Remove LockOn Widget
```

#### Event Tick에서 락온 카메라 업데이트
```
Event Tick
├─ Branch (bIsLockedOn AND LockOnTarget is Valid AND NOT LockOnTarget.bIsDead)
│  ├─ True
│  │  ├─ TargetLocation = LockOnTarget.GetActorLocation + (0, 0, 100)
│  │  ├─ LookAtRotation = Find Look at Rotation (GetActorLocation, TargetLocation)
│  │  ├─ CurrentRotation = GetControlRotation
│  │  ├─ NewRotation = RInterpTo(CurrentRotation, LookAtRotation, DeltaTime, 10.0)
│  │  ├─ SetControlRotation(NewRotation)
│  │  │
│  │  ├─ // 거리 체크 - 너무 멀면 해제
│  │  ├─ Distance = GetDistanceTo(LockOnTarget)
│  │  ├─ Branch (Distance > LockOnRange * 1.5)
│  │  │  └─ True → Call DisableLockOn()
│  │  │
│  └─ False
│     ├─ Branch (bIsLockedOn) → Call DisableLockOn() (타겟 사망/무효)
```

---

## 9. 게임 플로우

### BP_CombatGameMode 생성
Content Browser → `Content/BluePrint/` → 우클릭 → Blueprint Class → Game Mode Base → `BP_CombatGameMode`

#### 변수
| 이름 | 타입 | 기본값 |
|------|------|--------|
| AliveMonsterCount | Integer | 0 |
| TotalMonsterCount | Integer | 3 |
| bBossSpawned | Boolean | false |
| bGameCleared | Boolean | false |
| MonsterClass | Class Reference | BP_MonsterBasic |
| BossClass | Class Reference | BP_BossEnemy |
| MonsterSpawnPoints | Array of Actor References | - |
| BossSpawnPoint | Actor Reference | - |
| MainHUDWidget | WBP_MainHUD Reference | - |

#### Event BeginPlay
```
Event BeginPlay
├─ Get All Actors of Class (BP_MonsterSpawnPoint) → MonsterSpawnPoints
├─ Get All Actors of Class (BP_BossSpawnPoint) → BossSpawnPoint (첫 번째)
├─ Create Widget (WBP_MainHUD) → Add to Viewport → Store as MainHUDWidget
├─ SpawnMonsters()
```

#### Function: SpawnMonsters
```
SpawnMonsters()
├─ For Each SpawnPoint in MonsterSpawnPoints
│  ├─ SpawnActor (MonsterClass, SpawnPoint.GetTransform)
│  ├─ AliveMonsterCount++
├─ Update Mission UI ("몬스터 처치: 0/" + TotalMonsterCount)
```

#### Function: OnEnemyKilled (Actor KilledEnemy)
```
OnEnemyKilled(KilledEnemy)
├─ Branch (KilledEnemy has Tag "Boss")
│  ├─ True → OnBossKilled()
│  └─ False
│     ├─ AliveMonsterCount--
│     ├─ Update Mission UI ("몬스터 처치: " + (Total - Alive) + "/" + Total)
│     ├─ Branch (AliveMonsterCount <= 0 AND NOT bBossSpawned)
│     │  └─ True → SpawnBoss()
```

#### Function: SpawnBoss
```
SpawnBoss()
├─ Set bBossSpawned = true
├─ Update Mission UI ("보스 출현!")
├─ // 간단한 연출
├─ Play Level Sequence (LS_BossIntro) ── 또는 아래 간이 연출
│  ├─ (대안) Set View Target with Blend → BossSpawnPoint (카메라 전환)
│  ├─ Delay (2초)
│  ├─ SpawnActor (BossClass, BossSpawnPoint.GetTransform)
│  ├─ Spawn Emitter at Location (보스 등장 파티클)
│  ├─ Delay (1초)
│  ├─ Set View Target with Blend → Player (카메라 복귀)
├─ Update Mission UI ("보스를 처치하라!")
├─ Show Boss HP Bar in MainHUD
```

#### Function: OnBossKilled
```
OnBossKilled()
├─ Set bGameCleared = true
├─ Update Mission UI ("게임 클리어!")
├─ Delay (2초)
├─ Create Widget (WBP_GameClear) → Add to Viewport
├─ Set Game Paused (true) ── 또는 슬로모션
```

### BP_MonsterSpawnPoint
Content Browser → `Content/BluePrint/` → 우클릭 → Blueprint Class → Actor → `BP_MonsterSpawnPoint`
- BillboardComponent 추가 (에디터에서 보이게)
- 레벨에 3개 배치

### BP_BossSpawnPoint
- 동일하게 생성, 레벨에 1개 배치

### World Settings 설정
레벨 에디터 → World Settings → Game Mode Override = BP_CombatGameMode

---

## 10. UI 시스템

### 10-1. WBP_MainHUD (User Widget)
Content Browser → `Content/BluePrint/UI/` → 우클릭 → User Interface → Widget Blueprint → `WBP_MainHUD`

#### 레이아웃 (Canvas Panel)
```
[Canvas Panel]
├─ [Overlay - Top Left] ── HP/스태미너
│  ├─ [Vertical Box]
│  │  ├─ [HP Bar Section]
│  │  │  ├─ Text "HP"
│  │  │  └─ ProgressBar "HPBar" (Fill: Red, Size: 300x25)
│  │  ├─ [Stamina Bar Section]
│  │  │  ├─ Text "ST"
│  │  │  └─ ProgressBar "StaminaBar" (Fill: Green, Size: 250x18)
│  │
├─ [Overlay - Top Center] ── 미션 목표
│  ├─ Text "MissionText" (Font Size 18, White)
│  │
├─ [Overlay - Top Right] ── 미니맵
│  ├─ [Border] (원형 마스크용, 200x200)
│  │  └─ Image "MinimapImage" (Brush: RT_Minimap)
│  ├─ Image "PlayerArrow" (중앙 화살표)
│  │
├─ [Overlay - Top Center, Y offset] ── 보스 HP (기본 Hidden)
│  ├─ [Vertical Box] "BossHPSection"
│  │  ├─ Text "BossNameText"
│  │  └─ ProgressBar "BossHPBar" (Fill: Red, Size: 500x30)
```

#### 바인딩 (이벤트 그래프)

**Event Construct:**
```
Event Construct
├─ Get Player Character → Cast to BP_ToonCharacter → Store as PlayerRef
├─ Set Timer by Event (0.03초, Looping) → UpdateHUD
```

**Function: UpdateHUD**
```
UpdateHUD()
├─ HPBar → Set Percent (PlayerRef.CurrentHP / PlayerRef.MaxHP)
├─ StaminaBar → Set Percent (PlayerRef.CurrentStamina / PlayerRef.MaxStamina)
```

**Function: UpdateMissionText (String NewText)**
```
MissionText → Set Text (NewText)
```

**Function: ShowBossHP (BP_BossEnemy BossRef)**
```
├─ BossHPSection → Set Visibility (Visible)
├─ BossNameText → Set Text ("BOSS")
├─ Set Timer (0.03초, Looping) → Update BossHPBar Percent
```

**Function: HideBossHP**
```
BossHPSection → Set Visibility (Hidden)
```

### 10-2. WBP_EnemyHP (User Widget)
```
[Canvas Panel]
└─ [ProgressBar] "HPBar" (Size: 100x10, Fill: Red)
```

#### Function: SetHP (Float Percent)
```
HPBar → Set Percent (Percent)
```

### 10-3. WBP_LockOnMarker (User Widget)
```
[Canvas Panel]
└─ [Image] "MarkerIcon" (Diamond shape, White, Size: 40x40)
    └─ 또는 Text "◆" (White, Font Size 30)
```

### 10-4. WBP_GameClear (User Widget)
```
[Canvas Panel]
└─ [Overlay - Center]
    ├─ [Text] "GAME CLEAR" (Font Size: 60, Gold/Yellow)
    └─ Play Animation (FadeIn, 2초)
```

---

## 11. 미니맵

### 11-1. Render Target 생성
Content Browser → `Content/BluePrint/UI/` → 우클릭 → Materials & Textures → Render Target → `RT_Minimap`
- Size X: 512, Size Y: 512

### 11-2. BP_MinimapCapture (Actor)
Content Browser → Blueprint Class → Actor → `BP_MinimapCapture`

#### 컴포넌트
- **SceneComponent** (Root)
- **SceneCaptureComponent2D** "MinimapCamera"
  - Projection Type: Orthographic
  - Ortho Width: 3000 (조절 가능 - 미니맵 줌)
  - Texture Target: RT_Minimap
  - Rotation: (-90, 0, 0) → 아래를 내려다봄
  - Capture Every Frame: true
  - Primitive Render Mode: Render Scene Primitives (기본)
  - Show Flag Settings: Lighting OFF (선택) → 깔끔한 탑뷰

#### Event Tick
```
Event Tick
├─ Get Player Character → GetActorLocation
├─ Set Actor Location (PlayerX, PlayerY, PlayerZ + 5000)
│  └─ 플레이어 위 5000 유닛에서 따라다님
├─ Set Actor Rotation (0, -90, 0) ── 항상 아래를 봄
```

### 11-3. 미니맵 아이콘
간단한 방법: 각 적에게 Static Mesh Component (Plane + 빨간색 머티리얼) 추가
- BP_EnemyBase에 추가
- 머리 위 높은 곳에 배치 (Z + 4000)
- SceneCapture에만 보이도록 설정:
  - 방법 1: 별도 Custom Depth 사용
  - 방법 2: 간단하게 적 머리 위에 작은 빨간 Plane 배치 (높이가 높아서 게임 카메라에선 안 보임)

#### BP_EnemyBase에 추가
```
컴포넌트: StaticMeshComponent "MinimapIcon"
├─ Static Mesh: Plane
├─ Material: 빨간색 Unlit 머티리얼
├─ Relative Location: (0, 0, 4500) ── 미니맵 카메라에만 보임
├─ Scale: (0.3, 0.3, 0.3)
├─ Collision: No Collision
├─ 보스는 크기를 더 크게 (0.5, 0.5, 0.5) + 다른 색 (보라색)
```

### 11-4. WBP_MainHUD에 미니맵 표시
```
MinimapImage → Brush → Image = RT_Minimap
```

원형 마스크 처리 (선택):
- Material에서 RadialGradient로 원형 마스크 적용
- 또는 Border의 Clip을 이용

### 11-5. BP_CombatGameMode BeginPlay에 추가
```
SpawnActor (BP_MinimapCapture, Location: 0,0,5000)
```

---

## 빠른 참조: 전체 파일 생성 목록

### 새로 만들 BP (에디터)
| # | 이름 | 부모 클래스 | 폴더 |
|---|------|------------|------|
| 1 | S_MonsterStats | Structure | Content/Data/ |
| 2 | S_BossStats | Structure | Content/Data/ |
| 3 | S_BossSkills | Structure | Content/Data/ |
| 4 | S_PlayerStats | Structure | Content/Data/ |
| 5 | BP_EnemyBase | Character | Content/BluePrint/Enemy/ |
| 6 | BP_MonsterBasic | BP_EnemyBase | Content/BluePrint/Enemy/ |
| 7 | BP_BossEnemy | BP_EnemyBase | Content/BluePrint/Enemy/ |
| 8 | AIC_Monster | AIController | Content/BluePrint/AI/ |
| 9 | AIC_Boss | AIController | Content/BluePrint/AI/ |
| 10 | BB_Monster | Blackboard | Content/BluePrint/AI/ |
| 11 | BB_Boss | Blackboard | Content/BluePrint/AI/ |
| 12 | BT_Monster | Behavior Tree | Content/BluePrint/AI/ |
| 13 | BT_Boss | Behavior Tree | Content/BluePrint/AI/ |
| 14 | BTTask_RunEnemyAttack | BTTask_BlueprintBase | Content/BluePrint/AI/ |
| 15 | BTTask_SelectBossSkill | BTTask_BlueprintBase | Content/BluePrint/AI/ |
| 16 | BTTask_ExecuteBossSkill | BTTask_BlueprintBase | Content/BluePrint/AI/ |
| 17 | BTTask_UpdateDistance | BTTask_BlueprintBase | Content/BluePrint/AI/ |
| 18 | BP_CombatGameMode | GameModeBase | Content/BluePrint/ |
| 19 | BP_MonsterSpawnPoint | Actor | Content/BluePrint/ |
| 20 | BP_BossSpawnPoint | Actor | Content/BluePrint/ |
| 21 | IA_LockOn | InputAction | Content/BluePrint/Input/Actions/ |
| 22 | WBP_MainHUD | UserWidget | Content/BluePrint/UI/ |
| 23 | WBP_EnemyHP | UserWidget | Content/BluePrint/UI/ |
| 24 | WBP_LockOnMarker | UserWidget | Content/BluePrint/UI/ |
| 25 | WBP_GameClear | UserWidget | Content/BluePrint/UI/ |
| 26 | RT_Minimap | RenderTarget | Content/BluePrint/UI/ |
| 27 | BP_MinimapCapture | Actor | Content/BluePrint/ |
| 28 | ANS_BossAttackHitbox | AnimNotifyState | Content/BluePrint/Enemy/ |

### 수정할 기존 BP
| BP | 수정 내용 |
|----|----------|
| BP_ToonCharacter | 스태미너 시스템, 락온, HP, DataTable 로드 |
| IMC_Game | IA_LockOn 매핑 추가 |
| OpenWorld_Main | SpawnPoint 배치, World Settings |
