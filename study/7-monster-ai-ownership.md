# 몬스터 AI 소유권(Ownership) 구조

이 게임에서 가장 독특한 설계. 몬스터 AI를 **서버가 아니라 "소유자 클라이언트"가 시뮬레이션**한다.

## 왜 이렇게? — 서버는 지형을 모른다
서버는 지형 데이터(높이·물·풀·길)를 메모리에 들고 있지 않다 (지형은 절차생성 + 파일).
→ 서버는 **지형 인식 경로탐색/스폰 배치를 할 수 없다.**
그래서 지형을 렌더링하며 볼 수 있는 **클라이언트에게 AI 계산을 위임**한다.
효과: CPU 부하를 5000개 클라로 분산 + 서버는 가볍게. 대신 서버가 **검증·권위**는 유지.

---

## 역할 분담

| 항목 | 누가 | 비고 |
|------|------|------|
| 몬스터 존재/생성/제거 | **서버** | 인메모리 `monsters` HashMap |
| 소유권(owner_id) 배정 | **서버** | 스폰 시 요청 플레이어에게 |
| 스폰 상한(max_per_player, max_total) | **서버** | `tick_monster_spawns` |
| 스폰 위치 결정 | **소유자 클라** | 지형 보고 유효 위치 탐색 |
| AI 이동/경로탐색 | **소유자 클라** | `MonsterMove`로 보고 |
| 데미지/명중/죽음 판정 | **서버** | server-authoritative (치트 방지) |
| 근처 전파 | **서버** | AOI로 `MonsterMoved` 중계 |

**핵심 원칙**: "무엇을/얼마나/살았나죽었나 = 서버", "어떻게 움직이나/어디 세우나 = 소유 클라". 신뢰가 필요한 부분만 서버가 쥔다.

---

## 스폰 흐름 (server-driven, client-placed)

```
8s? 아니 10s tick: tick_monster_spawns (monster.rs:263)
  └─ 플레이어별 (owner, type) 살아있는 수 카운트
  └─ max_per_player 미만이면 → 그 플레이어에게 SpawnMonsterRequest { monster_type }   [서버→클라]
       └─ 클라 monsterManager.tryAmbientSpawn: 지형 보고 유효 위치 탐색
            └─ RequestSpawnMonster { type, position, rotation }                        [클라→서버]
                 └─ 서버 validate_spawn_position (no-spawn 존/사거리 검사)
                      └─ spawn_monster(..., owner=요청자)  → MonsterAssigned { monster } [서버→그 클라]
```
- 서버는 "이 플레이어 주변에 고블린 하나 더 필요"까지만 판단.
- **실제 좌표는 클라가** (서버는 지형을 몰라 grassland/water 판정 불가 — 주석에 명시).
- 서버는 존/사거리만 재검증 → 통과하면 소유권 부여.

## 이동 흐름 (client-simulated, server-relayed)

```
소유자 클라: 몬스터 AI 시뮬레이션(추적/배회/공격) → 매 프레임 위치 계산
  └─ MonsterMove { monster_id, position, rotation, state, target_position }   [클라→서버]
       └─ update_monster_position (monster.rs:133)
            └─ monster.is_controllable_by(mover_id)  ← 소유자만 통과 (아니면 무시)
            └─ 인메모리 위치 갱신
            └─ fanout_monster_position_update → 근처(AOI)에 MonsterMoved   [서버→근처, 소유자는 skip]
```
- **소유자만 그 몬스터를 움직일 수 있다** (`is_controllable_by`). 남이 위조 이동을 보내도 거부.
- 소유자는 자기가 이미 아니까 fanout에서 **자신은 제외**(skip_player_id).
- 다른 근처 플레이어는 `MonsterMoved` 받아 `monsterManager.updateMonsterFromNetwork`로 보간 렌더.

## 데미지는 예외 — 서버가 계산
AI는 클라가 하지만 **전투 결과는 서버**. `PlayerAttack`/`MonsterAttack` → 서버가 주사위/HP/죽음 판정
후 근처에 `PlayerAttacked`/`MonsterDead` 통보 (→ [`5-realtime-sync.md`](./5-realtime-sync.md)). 클라가 AI를 굴려도 데미지는 못 속인다.

## 소유자 이탈 시
소유자 접속이 끊기면 그 몬스터들은 방치되지 않고 정리/이관:
- 지상: `remove_monsters_by_owner` 로 제거 (monster.rs:227)
- 던전: 다른 점유자에게 재할당 (없으면 despawn) — 던전은 층 인스턴스라 재구성 처리.

---

## 코드 대응표 (서버 ↔ 프론트)
| 단계 | 서버 | 프론트 |
|------|------|--------|
| 스폰 요청 | `monster.rs:263 tick_monster_spawns` → `SpawnMonsterRequest` | `messageHandlers 'SpawnMonsterRequest'` → `monsterManager.tryAmbientSpawn` |
| 스폰 확정 | `connection.rs:744 RequestSpawnMonster` → `spawn_monster` → `MonsterAssigned` | `messageHandlers 'MonsterAssigned'` → `adoptOwnership` |
| AI 이동 보고 | `connection.rs:772 MonsterMove` → `update_monster_position` | 소유자 `monsterManager` AI → `MonsterMove` 송신 |
| 이동 중계 | `monster.rs:174 fanout_monster_position_update` → `MonsterMoved` | `messageHandlers 'MonsterMoved'` → `updateMonsterFromNetwork` |
| 죽음 | `combat.rs broadcast_player_attack` → `MonsterDead` | `messageHandlers 'MonsterDead'` |

관련 문서: [`5-realtime-sync.md`](./5-realtime-sync.md), [`4-server-ticks.md`](./4-server-ticks.md), [`1-server-architecture.md`](./1-server-architecture.md)
