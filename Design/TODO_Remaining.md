# 남은 작업 및 미해결 이슈

---

## 미해결 버그

| 버그 | 설명 | 우선순위 |
|------|------|----------|
| C-3 AttackRange 덮어쓰기 | VariableSet_4가 DetectionRange를 AttackRange에 덮어씀. VariableSet_4 삭제 후 VariableSet_3→VariableSet_5 직접 연결 | 중 |
| 스태미너 중복 소모 | 공격 연타 시 스태미너 중복 차감 | 중 |
| 죽은 플레이어 계속 공격 | Event AnyDamage에 bIsDead 체크 + 캡슐 콜리전 No Collision 처리 미완 | 낮음 (현재 적이 Patrol 복귀로 우회) |

---

## 미완료 기능

### 달리기 스태미너 소모 (보류)
- 현재 Sprint은 스태미너 소모 없이 동작
- 추후 기획 확정 후 Event Tick에 스태미너 소모 로직 추가 예정
- 가이드: Phase_G_LockOn_Sprint_Guide.md G-3, G-8 참고

### 강공격 넉백 + 이펙트 (Phase H)
- 가이드 완료: `Phase_H_HeavyAttack_Knockback_Guide.md`
- DT_HeavyAttack 생성 → Apply Damage 적용 → BP_EnemyBase 분기 처리 → Launch Character + Spawn Effect
- 카메라 쉐이크 선택사항

### Phase F: UI
- WBP_MainHUD (HP바, 스태미너바 등)

### Phase I: 보스
- BP_BossEnemy + 보스 AI

### Phase J: 게임 플로우
- 스테이지 진행, 클리어 조건 등

### Phase K: 미니맵

---

## 현재 완료된 시스템 요약

- 전투 시스템 (약공격/강공격 콤보, 스태미너, 히트박스)
- 적 AI (Patrol → Chase → Attack, BT 기반)
- 락온 시스템 (Ctrl 토글, 카메라 추적, UI 마커, 자동 해제)
- 달리기 (Shift, 스태미너 소모 없음)
- 구르기 (Alt)
- 플레이어 사망 (ABP Dead 상태, GameOver UI, 재시작)
- 피격 시 적 즉시 인지 (블랙보드 직접 세팅)
- 키 매핑: LockOn=Ctrl, Dodge=Alt, Sprint=Shift
