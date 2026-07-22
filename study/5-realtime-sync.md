# 실시간 동기화: 이동 & 전투 브로드캐스트

## 큰 그림: "전체 브로드캐스트"가 아니라 AOI(Area of Interest)

실시간 이벤트(이동, 공격, 몬스터 죽음 등)는 **모든 접속자에게 뿌리지 않는다.**
**근처(같은 층 + 반경 `EVENT_DELIVERY_RADIUS`) 플레이어에게만** 각자의 웹소켓으로 push.

- 전파 채널 = 플레이어별 **mpsc `direct_channel`** (broadcast 아님)
- 근처 판별 = **spatial cell(격자)** 인덱스 `player_spatial_cells`
- 이유 = 5000명 동접에서 "모두에게 모든 이벤트"는 O(N²)로 터짐. 10km 밖 플레이어는 알 필요 없음.

> 진짜 전체가 알아야 하는 것(예: 게임 내 시간)만 global `broadcast_tx` 사용.

---

## 캐릭터 이동 전파

```
클라이언트 PlayerMove ─▶ update_player_position (인메모리 갱신)
                          │
      200ms 이동 틱 ──────┘  tick_player_movement: 목표로 조금씩 이동 (서버 권위적)
                          │
                          ▼
              fanout_player_position_update   (player.rs:995) ← 핵심
```

`fanout_player_position_update`가 하는 일:
1. 이동 **전** 반경 안 플레이어 = `old_visible`
2. 이동 **후** 반경 안 플레이어 = `new_visible`
3. 차집합/교집합으로 분류:
   - `left`  (멀어짐)   → 서로에게 `PlayerDisappeared`
   - `entered`(가까워짐) → 서로에게 `PlayerAppeared`
   - `stayed`(계속 근처) → `PlayerMoved`

```rust
self.send_direct_message(player_id, update_msg.clone())    // 자기 자신
self.send_direct_message_to_players(&stayed, update_msg)   // 근처에 남은 사람들
```

이동 시 소속 격자도 갱신: `move_player_spatial_cell`.
같은 팬아웃에서 근처 **몬스터/바닥아이템**의 등장/사라짐(`MonsterSpawned`/`MonsterRemoved`, `GroundItemAppeared`/`GroundItemRemoved`)도 함께 계산해 보냄.

---

## 공격 / 몬스터 데미지 전파 (같은 메커니즘)

`broadcast_player_attack` (combat.rs:82):
1. 몬스터 생존/거리/층 검증 (다른 층·사거리 밖이면 무시)
2. **서버에서 데미지 계산** — 주사위(`roll_attack`), STR 보정, 무기 dice, 인챈트 (+N)
3. 결과를 **근처 플레이어에게만** 통보:

```rust
self.send_direct_message_to_players_within_position(
    &monster_position, monster_floor_level, EVENT_DELIVERY_RADIUS,
    ServerMessage::PlayerAttacked { player_id, monster_id, hit, roll, damage },
    None,
).await;
```

4. 명중 시 몬스터 HP 차감(인메모리) → 0이면 `MonsterDead`도 같은 반경에 전송, 루팅/월드드롭 생성.

**서버 권위적(server-authoritative)**: 클라이언트는 "때렸다"만 보내고 실제 수치는 서버가 확정.
→ 근처 모든 플레이어가 **동일한 데미지 숫자**를 동시에 봄. 클라이언트 조작 불가.

---

## 이동 vs 공격 대조

| | 이동 | 공격/데미지 |
|--|------|------------|
| 상태 갱신 | 인메모리 `players` | 인메모리 `monsters` |
| 전파 대상 | 근처 플레이어 (AOI) | 근처 플레이어 (AOI) |
| 전파 채널 | `direct_channel` (mpsc) | `direct_channel` (mpsc) |
| 반경 | `EVENT_DELIVERY_RADIUS` | `EVENT_DELIVERY_RADIUS` |
| 권위 | 서버 (200ms 틱이 위치 계산) | 서버 (데미지 계산) |

**결론**: 둘 다 "인메모리 갱신 → 근처(AOI) 플레이어의 웹소켓으로만 push"라는 동일 패턴.

## 관련 코드 (서버)
- `server/src/game_state/player.rs` — `tick_player_movement`, `fanout_player_position_update`, `player_ids_within_position`, `move_player_spatial_cell`
- `server/src/game_state/combat.rs` — `broadcast_player_attack`, `broadcast_monster_attack`
- `server/src/game_state/mod.rs` — `broadcast`(global), `subscribe`, `EVENT_DELIVERY_RADIUS`
- `server/src/connection.rs` — `select!` 루프에서 direct/broadcast 수신, `PlayerMove`/`PlayerAttack` 처리(703, 700번대)

## 코드 레벨 대응표 (서버 ↔ 프론트)

### 이동
| 단계 | 서버 | 프론트 |
|------|------|--------|
| 입력→송신 | — | `PlayerControl.svelte:371 sendPlayerMove` → `socket.ts:321` → `{ PlayerMove }` |
| 수신·검증 | `connection.rs:703 PlayerMove` → `update_player_position` | — |
| 위치 계산 | `player.rs:509 tick_player_movement` (200ms) | — |
| 팬아웃 | `player.rs:995 fanout_player_position_update` → `PlayerMoved` | — |
| 남 화면 반영 | — | `messageHandlers.ts:280 'PlayerMoved'` → `remotePlayerManager.setTargetPosition` (보간) |

### 공격/데미지
| 단계 | 서버 | 프론트 |
|------|------|--------|
| 입력→송신 | — | `PlayerControl.svelte:600 sendPlayerAttack` → `socket.ts:300` → `{ PlayerAttack }` |
| 데미지 계산 | `combat.rs:82 broadcast_player_attack` (`roll_attack`) | — |
| 결과 통보 | `send_direct_message_to_players_within_position` → `PlayerAttacked`/`MonsterDead` | — |
| 화면 반영 | — | `messageHandlers.ts 'PlayerAttacked'/'MonsterDead'` → `monsterManager` |

> 상세 프론트 웹소켓 구조는 [`2-client-networking.md`](./2-client-networking.md) 참고.
