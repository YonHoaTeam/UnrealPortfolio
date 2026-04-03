# Phase G: 락온 + 달리기 시스템 구현 가이드

> 키 매핑: LockOn = Ctrl, Dodge = Alt, Sprint = Shift

---

## 진행 체크리스트

| 단계 | 작업 | 상태 |
|------|------|------|
| G-1 | IA_Sprint 생성 | ☐ |
| G-2 | IMC_Game 매핑 변경 (LockOn→Ctrl, Dodge→Alt, Sprint→Shift) | ☐ |
| G-3 | BP_ToonCharacter 달리기 변수 + 로직 | ☐ |
| G-4 | BP_ToonCharacter 락온 변수 추가 | ☐ |
| G-5 | WBP_LockOnMarker 위젯 생성 | ☐ |
| G-6 | EnableLockOn / DisableLockOn 함수 | ☐ |
| G-7 | IA_LockOn 이벤트 연결 | ☐ |
| G-8 | Event Tick에 락온 카메라 + 달리기 스태미너 추가 | ☐ |
| G-9 | 테스트 | ☐ |

---

## G-1. IA_Sprint 생성

`Content/BluePrint/Input/Actions/` → 우클릭 → **Input** → **Input Action** → `IA_Sprint`
- Value Type: **Digital (Bool)**

---

## G-2. IMC_Game 매핑 변경

`Content/BluePrint/Input/IMC_Game` 열기

### 변경할 매핑

| Input Action | 키 |
|-------------|-----|
| IA_LockOn | **Left Ctrl** (기존 Mouse Middle Button 삭제) |
| IA_Dodge | **Left Alt** (기존 키 삭제 후 변경) |
| IA_Sprint | **Left Shift** (새로 추가) |

**방법:**
1. 기존 IA_LockOn 행 → 키 클릭 → **Left Ctrl** 선택
2. 기존 IA_Dodge 행 → 키 클릭 → **Left Alt** 선택
3. **+** 버튼 → IA_Sprint 추가 → **Left Shift** 선택

**Save**

---

## G-3. 달리기 시스템 (BP_ToonCharacter)

### 변수 추가

| 변수명 | 타입 | 기본값 | 설명 |
|--------|------|--------|------|
| `bIsSprinting` | Boolean | `false` | 현재 달리기 중 |
| `SprintSpeed` | Float | `800` | 달리기 속도 |
| `WalkSpeed` | Float | `500` | 기본 이동 속도 |
| `SprintStaminaCost` | Float | `15` | 초당 스태미너 소모량 |

> WalkSpeed는 기존 CharacterMovement → MaxWalkSpeed와 같은 값으로 설정

### IA_Sprint 이벤트

```
IA_Sprint Started (Shift 누름)
 │
 ├─ Branch (CurrentStamina > 0)
 │   ├─ True
 │   │   ├─ Set bIsSprinting = true
 │   │   └─ CharacterMovement → Set Max Walk Speed (SprintSpeed)
 │   └─ False → 무시
```

```
IA_Sprint Completed (Shift 뗌)
 │
 ├─ Set bIsSprinting = false
 └─ CharacterMovement → Set Max Walk Speed (WalkSpeed)
```

### Event Tick에 달리기 스태미너 소모 추가

> 기존 Event Tick의 스태미너 회복 로직 **앞**에 추가

```
Event Tick
 │
 ├─ Branch (bIsSprinting AND 캐릭터가 실제 이동 중)
 │   │
 │   │  ※ 이동 판정: GetVelocity → VectorLength > 10
 │   │
 │   ├─ True (달리는 중)
 │   │   ├─ NewStamina = CurrentStamina - (SprintStaminaCost * DeltaTime)
 │   │   ├─ Set CurrentStamina = Max(NewStamina, 0)
 │   │   ├─ Branch (CurrentStamina <= 0)
 │   │   │   ├─ True → 스태미너 바닥
 │   │   │   │   ├─ Set bIsSprinting = false
 │   │   │   │   └─ CharacterMovement → Set Max Walk Speed (WalkSpeed)
 │   │   │   └─ False → 계속 달리기
 │   │   └─ Set StaminaRegenTimerHandle → Clear (회복 딜레이 리셋)
 │   │
 │   └─ False → (기존 스태미너 회복 로직 실행)
 │
 ├─ [기존 스태미너 회복 로직] ← bIsSprinting == false 일 때만 실행되도록
 │   ...
```

**핵심 포인트:**
- 달리면서 가만히 서있으면 스태미너 안 빠지게 (GetVelocity 체크)
- 스태미너 0 되면 자동으로 걷기로 전환
- 달리는 동안에는 스태미너 회복 안 됨

---

## G-4. 락온 변수 추가 (BP_ToonCharacter)

| 변수명 | 타입 | 기본값 |
|--------|------|--------|
| `bIsLockedOn` | Boolean | `false` |
| `LockOnTarget` | Actor Reference | None |
| `LockOnRange` | Float | `2000` |
| `LockOnWidget` | WBP_LockOnMarker Reference | None |

---

## G-5. WBP_LockOnMarker 위젯 생성

`Content/BluePrint/UI/` → 우클릭 → **User Interface** → **Widget Blueprint** → `WBP_LockOnMarker`

### Designer 탭

```
[Canvas Panel]
 └─ Text "◆" (White, Font Size 30, Anchor: Center)
```

**Save**

---

## G-6. 락온 함수 (BP_ToonCharacter)

### Function: EnableLockOn

```
EnableLockOn()
 │
 ├─ SphereOverlapActors
 │   - Location: GetActorLocation
 │   - Radius: LockOnRange
 │   - Object Types: Pawn
 │   - Filter: ActorHasTag("Enemy")
 │   → OutActors 배열
 │
 ├─ Branch (OutActors is empty)
 │   ├─ True → Return (주변에 적 없음)
 │   └─ False ↓
 │
 ├─ [로컬 변수] BestTarget = None, BestDot = -1
 │
 ├─ For Each Actor in OutActors
 │   ├─ Cast to BP_EnemyBase → 실패 시 Skip
 │   ├─ Branch (bIsDead) → True: Skip
 │   ├─ DirectionToEnemy = (Enemy.Location - Camera.Location).Normalize
 │   ├─ CameraForward = GetPlayerCameraManager → Get Forward Vector
 │   ├─ DotProduct = Dot(CameraForward, DirectionToEnemy)
 │   └─ Branch (DotProduct > BestDot)
 │       └─ True → BestDot = DotProduct, BestTarget = Actor
 │
 ├─ Branch (BestTarget is Valid)
 │   ├─ True
 │   │   ├─ Set LockOnTarget = BestTarget
 │   │   ├─ Set bIsLockedOn = true
 │   │   ├─ CharacterMovement → Set bOrientRotationToMovement = false
 │   │   ├─ CharacterMovement → Set bUseControllerDesiredRotation = true
 │   │   └─ Create Widget (WBP_LockOnMarker)
 │   │       → Attach to Component (LockOnTarget → Mesh, Z offset +100)
 │   │       → Set LockOnWidget
 │   └─ False → Return
```

> **ActorHasTag("Enemy")** 사용하려면 BP_EnemyBase의 BeginPlay에서 `Tags → Add "Enemy"` 해줘야 합니다.
> 이미 되어있다면 스킵.

### Function: DisableLockOn

```
DisableLockOn()
 │
 ├─ Set bIsLockedOn = false
 ├─ Set LockOnTarget = None
 ├─ CharacterMovement → Set bOrientRotationToMovement = true
 ├─ CharacterMovement → Set bUseControllerDesiredRotation = false
 ├─ Branch (LockOnWidget is Valid)
 │   ├─ True → LockOnWidget → Remove from Parent
 │   └─ Set LockOnWidget = None
```

---

## G-7. IA_LockOn 이벤트 (BP_ToonCharacter)

```
IA_LockOn Started (Ctrl 누름)
 │
 ├─ Branch (bIsLockedOn)
 │   ├─ True → Call DisableLockOn()
 │   └─ False → Call EnableLockOn()
```

토글 방식: Ctrl 한번 누르면 ON, 다시 누르면 OFF

---

## G-8. Event Tick — 락온 카메라 + 달리기 통합

> 기존 Event Tick에 아래 로직을 추가합니다.
> 순서: **달리기 스태미너 → 락온 카메라 → 기존 스태미너 회복**

```
Event Tick (DeltaTime)
 │
 │  ======= [1] 달리기 스태미너 소모 =======
 │
 ├─ Branch (bIsSprinting)
 │   ├─ True
 │   │   ├─ Velocity = GetVelocity → VectorLength
 │   │   ├─ Branch (Velocity > 10)
 │   │   │   ├─ True → 실제 이동 중
 │   │   │   │   ├─ CurrentStamina -= SprintStaminaCost * DeltaTime
 │   │   │   │   ├─ CurrentStamina = Max(CurrentStamina, 0)
 │   │   │   │   ├─ Branch (CurrentStamina <= 0)
 │   │   │   │   │   └─ True → Set bIsSprinting = false
 │   │   │   │   │          → CharacterMovement.MaxWalkSpeed = WalkSpeed
 │   │   │   │   └─ (스태미너 회복 딜레이 리셋 — Clear Timer)
 │   │   │   └─ False → 서있기 (소모 안 함)
 │   └─ False → (달리기 안 하는 중)
 │
 │  ======= [2] 락온 카메라 추적 =======
 │
 ├─ Branch (bIsLockedOn AND LockOnTarget is Valid AND NOT LockOnTarget.bIsDead)
 │   ├─ True
 │   │   ├─ TargetLocation = LockOnTarget.GetActorLocation + (0, 0, 100)
 │   │   ├─ LookAtRotation = Find Look at Rotation(GetActorLocation, TargetLocation)
 │   │   ├─ NewRotation = RInterpTo(GetControlRotation, LookAtRotation, DeltaTime, 10.0)
 │   │   ├─ SetControlRotation(NewRotation)
 │   │   │
 │   │   ├─ Distance = GetDistanceTo(LockOnTarget)
 │   │   ├─ Branch (Distance > LockOnRange * 1.5)
 │   │   │   └─ True → Call DisableLockOn()  (거리 벗어남)
 │   │   │
 │   └─ False
 │       └─ Branch (bIsLockedOn)
 │           └─ True → Call DisableLockOn()  (타겟 사망/무효)
 │
 │  ======= [3] 스태미너 자연 회복 (기존 로직) =======
 │
 ├─ Branch (NOT bIsSprinting)  ← 달리기 중이면 회복 안 함
 │   └─ True → [기존 스태미너 회복 로직 그대로]
```

---

## 락온 + 달리기 동시 사용 시 동작

| 상황 | 동작 |
|------|------|
| 락온 + 달리기 | 타겟 방향 바라보면서 달리기 가능 |
| 달리기 중 스태미너 0 | 자동으로 걷기 전환, 락온은 유지 |
| 락온 중 적 사망 | 자동 해제, 달리기는 유지 |
| 달리기 중 공격 | 공격 시 bIsSprinting = false 처리 권장 |

### 공격 시 달리기 해제 (권장)

기존 공격 로직(IA_LightAttack, IA_HeavyAttack)의 **몽타주 재생 직전**에 추가:

```
(공격 시작 직전)
 ├─ Branch (bIsSprinting)
 │   └─ True
 │       ├─ Set bIsSprinting = false
 │       └─ CharacterMovement → Set Max Walk Speed (WalkSpeed)
 └─ (기존 공격 로직 계속)
```

---

## G-9. 테스트 체크리스트

- [ ] **Shift** 누르면 캐릭터 속도 증가 (800)
- [ ] Shift 뗌 → 기본 속도 복귀 (500)
- [ ] 달리기 중 스태미너 서서히 감소
- [ ] 스태미너 0 → 자동 걷기 전환
- [ ] 가만히 서서 Shift → 스태미너 안 빠짐
- [ ] 달리기 멈추면 딜레이 후 스태미너 회복
- [ ] **Ctrl** → 가장 가까운 정면 적에 락온 (◆ 마커 표시)
- [ ] 다시 Ctrl → 락온 해제
- [ ] 락온 중 카메라가 적을 부드럽게 추적
- [ ] 적 사망 시 자동 해제
- [ ] 거리 벗어나면 자동 해제
- [ ] **락온 + 달리기 동시** 작동 확인
- [ ] **Alt** → 구르기 정상 작동
