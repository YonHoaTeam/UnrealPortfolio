# Phase H: 강공격 넉백 + 이펙트 시스템 구현 가이드

---

## 진행 체크리스트

| 단계 | 작업 | 상태 |
|------|------|------|
| H-1 | DT_HeavyAttack Damage Type 생성 | ☐ |
| H-2 | 강공격 Apply Damage에 DT_HeavyAttack 적용 | ☐ |
| H-3 | 피격 이펙트 준비 (Niagara/Cascade) | ☐ |
| H-4 | BP_EnemyBase Event AnyDamage 수정 (강공격 분기) | ☐ |
| H-5 | 넉백 처리 (Launch Character) | ☐ |
| H-6 | 이펙트 Spawn | ☐ |
| H-7 | 카메라 쉐이크 (선택) | ☐ |
| H-8 | 테스트 | ☐ |

---

## H-1. DT_HeavyAttack Damage Type 생성

`Content/BluePrint/` → 우클릭 → **Blueprint Class** → 부모: **Damage Type** → `DT_HeavyAttack`

---

## H-2. 강공격 Apply Damage에 적용

기존 강공격 데미지 주는 곳 (Apply Damage 노드):
- **Damage Type Class** 핀에 `DT_HeavyAttack` 선택
- 약공격은 기본 Damage Type 그대로

---

## H-3. 피격 이펙트 준비

Niagara 또는 Cascade 이펙트 에셋 준비
- 기존 에셋 활용 또는 마켓플레이스/엔진 기본 이펙트 사용

---

## H-4. BP_EnemyBase Event AnyDamage 수정

```
Event AnyDamage (Damage, DamageType, InstigatedBy, DamageCauser)
 │
 ├─ (기존 데미지 계산 로직)
 │
 ├─ Branch (DamageType → Cast to DT_HeavyAttack → 성공 여부)
 │
 │   ├─ True (강공격)
 │   │   ├─ Play Anim Montage (AM_HeavyReact 또는 기존 React)
 │   │   ├─ 넉백 (H-5)
 │   │   └─ 이펙트 (H-6)
 │   │
 │   └─ False (약공격)
 │       └─ Play Anim Montage (AM_React) 기존 피격 몽타주
```

---

## H-5. 넉백 처리

```
DamageCauser → Get Actor Forward Vector
 → Multiply (Float: 800)  ← 넉백 거리 조절 가능
 → Make Vector (X: 위결과.X, Y: 위결과.Y, Z: 150)
    → Launch Character
       XY Override: true
       Z Override: true
```

- DamageCauser의 Forward Vector = 때린 방향
- X/Y 800: 밀려나는 거리
- Z 150: 살짝 뜨는 느낌

---

## H-6. 이펙트 Spawn

### Niagara 사용 시
```
Spawn System at Location
 - System Template: 원하는 Niagara System
 - Location: Self → GetActorLocation + (0, 0, 50)
 - Rotation: DamageCauser → Get Actor Forward Vector → Rotation From X Vector
 - Auto Destroy: true
```

### Cascade 사용 시
```
Spawn Emitter at Location
 - Emitter Template: 원하는 Particle System
 - Location: Self → GetActorLocation + (0, 0, 50)
 - Rotation: 위와 동일
```

---

## H-7. 카메라 쉐이크 (선택)

```
Get Player Controller → Client Start Camera Shake
 - Shake Class: 원하는 CameraShake BP
```

타격감 강화용, 필수는 아님.

---

## H-8. 테스트 체크리스트

- [ ] 강공격 시 적이 때린 방향으로 밀려남
- [ ] 약공격 시 넉백 없이 React만 재생
- [ ] 이펙트가 적 위치에서 올바른 방향으로 Spawn
- [ ] 넉백 거리/높이 적절한지 확인
- [ ] 카메라 쉐이크 작동 (적용 시)

---

## 동시 동작 참고

| 상황 | 동작 |
|------|------|
| 강공격 + 락온 중 | 넉백으로 거리 벌어지면 락온 자동 해제 가능 |
| 강공격 + 적 사망 | 사망 판정이 우선, 넉백은 무시 권장 |
