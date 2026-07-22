# 서버 tick 구조

## 흔한 오해 3가지 (먼저 정정)
1. ❌ "단일 게임 루프가 매 프레임 전부 다시 계산" → ✅ **주기가 다른 여러 독립 tick**이 각자 돈다.
2. ❌ "매 tick 모든 엔티티 계산" → ✅ **바뀔 여지가 있는 것만** (이동 큐에 있는 플레이어만 등).
3. ❌ "매 tick 전체 상태를 모두에게 push" → ✅ **바뀐 것(delta)만, 근처(AOI)에게만**. 전체 스냅샷은 입장 시 1회.

---

## tick 목록 (server/src/main.rs 에서 tokio::spawn)

| 주기 | 태스크 | 하는 일 |
|------|--------|---------|
| **200ms** (초당 5회) | `tick_player_movement` | 이동 중인 플레이어를 목표로 전진 (서버 권위적 위치) |
| 8s | `time_sync_tick` | 게임 시간 진행, `GameTimeSync` 방송, 32초마다 dirty 배치 저장 |
| 10s | `tick_monster_spawns` | 플레이어별 주변 몬스터를 상한까지 보충 |
| 30s | `tick_dungeons` | 던전 스폰 슬롯 리필 |
| 30s | `tick_ground_item_despawn` | 오래된 바닥 아이템 소멸 |

각 tick은 `guard_tick`으로 감싸 패닉이 나도 루프가 죽지 않음 (`main.rs:39`).
tick들은 서로 **다른 주파수로 독립 실행** — 하나가 느려도 나머지에 영향 최소.

---

## 이동 tick 상세 (`game_state/player.rs:509 tick_player_movement`)

```rust
let mut queues = self.movement_intents.write().await;
if queues.is_empty() { return; }          // 아무도 안 움직이면 즉시 종료 (가만한 5000명은 비용 0)

queues.retain(|player_id, waypoints| {
    // 목표 waypoint 쪽으로 (속도 × dt) 만큼 전진, 충돌 검사(check_collision)
    // 도착하면 waypoint pop, 막히면 큐 비우고 정지
    // 위치가 바뀌면 moved 에 기록
});

for moved_player in moved {
    // 바뀐 사람만 → finish_position_update → 근처(AOI)에게 PlayerMoved
}
```

- **처리 대상 = 이동 intent가 있는 플레이어만.** 정지 상태는 계산 안 함.
- **서버 권위적**: 클라는 "목표 좌표"만 보내고(`PlayerMove` → `movement_intents` 큐), 실제 전진량은 서버가 속도 상한으로 계산 → 순간이동/속도핵 방지.
- 결과는 delta(`PlayerMoved`)로 **근처에만** (→ [`5-realtime-sync.md`](./5-realtime-sync.md)).

---

## 정확한 데이터 흐름

```
클라 입력 → PlayerMove(목표 좌표) → movement_intents 큐에 push       [이벤트-드리븐]
                                            │
200ms tick ──────────────────────────────────┘  큐 속 플레이어만 전진 계산
                                            │
                                            ▼
                       바뀐 사람 → 근처(AOI)에게 PlayerMoved         [델타 push]
```

**이벤트-드리븐(큐에 쌓기) + tick-드리븐(주기적 소비)** 하이브리드.
"매 프레임 전부 다시 + 전체 push"가 아니라서 5000명에서 CPU/대역폭이 안 터진다.

---

## 예외: 몬스터 이동은 tick이 아님
몬스터 AI 이동은 서버 200ms tick이 계산하지 않는다. **소유자 클라이언트가 시뮬레이션**해서
`MonsterMove`로 서버에 보고 → 서버가 근처에 중계하는 **소유권(ownership) 기반** 구조.
(서버가 스폰 슬롯/소유권만 관리, 부하를 클라로 분산) — 별도 문서 후보.

관련 문서: [`1-server-architecture.md`](./1-server-architecture.md), [`5-realtime-sync.md`](./5-realtime-sync.md), [`3-game-entry-lifecycle.md`](./3-game-entry-lifecycle.md)
