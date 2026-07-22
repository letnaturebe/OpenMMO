# 게임 진입 라이프사이클: 로그인 → 캐릭터 → 게임 입장

이동/전투는 "게임 안"이었다면, 이 문서는 **그 앞단** — 소켓 연결 후 월드에 들어가 초기 상태를 받기까지.

## 전체 시퀀스

```
연결(WS) ─▶ Authenticate ─▶ [RollStats ─▶ CreateCharacter]* ─▶ EnterGame ─▶ 게임 중
             (계정 확정)       (캐릭터 준비, 최초 1회)          (월드 입장 + 초기 스냅샷)
```

핸들러는 전부 `server/src/connection.rs`의 `handle_client_message` `match` 안에 있음.
**인증 전에는 `Authenticate`/`AuthenticateNpc`만 허용** (connection.rs:370 게이트).

---

## 1. Authenticate — 계정 확정 (connection.rs:394)

```rust
verifier.verify(&google_id_token)      // Google 공개키로 id_token 검증 (aud/iss/exp/서명)
  → claims.sub                          // 구글 사용자 고유 ID
auth_service.login_google(&claims.sub)  // accounts 테이블에서 sub로 조회/생성 → account_name
state.admin_eligible = is_admin(claims) // 이메일이 admin 허용목록이면 true
finish_auth(...)                        // AuthSuccess { account_name, characters[] } 반환
```
- 결과 `ServerMessage::AuthSuccess`에 **그 계정의 캐릭터 목록**을 실어 보냄 → 클라가 캐릭터 선택 UI 렌더.
- 봇은 `AuthenticateNpc`(NPC 토큰) 경로, 계정 자동 생성.

## 2. RollCharacterStats + CreateCharacter — 캐릭터 준비 (최초 1회)

```rust
RollCharacterStats  → 4d6-drop-lowest + 클래스 보정 → state.pending_character_attributes 에 보관
                      → CharacterStatsRolled { attributes, maxHp }
CreateCharacter     → pending_attributes 사용해 auth_service.create_character() (characters 테이블 INSERT)
                      → CharacterCreated { character }
```
- 스탯은 **서버가 굴려** `pending_character_attributes`에 잠깐 들고 있다가 생성 시 확정 → 클라가 스탯 조작 불가.
- 이미 캐릭터가 있으면 이 단계 건너뛰고 바로 EnterGame.

## 3. EnterGame — 월드 입장 (connection.rs:554) ★핵심

```rust
get_character_for_account(account, character_id)   // DB에서 캐릭터 로드 (권한 확인 포함)
state.is_admin = admin_eligible && character.admin_role > 0
kick_player_by_name(character.name)                // 유니크 이름 → 같은 캐릭터 기존 세션 강퇴(중복 로그인 방지)

let mut player = new_player(... last_x/y/z, last_rotation ...)  // DB의 마지막 위치/방향 복원
player.health = saved_health;  player.floor_level = character.floor_level  // HP/층 복원
if floor_level < 0 { rehydrate_dungeon_player(...) }           // 던전 안에서 로그아웃했으면 그 던전 재구성

state.direct_rx = register_direct_channel(&id)     // 이 플레이어 전용 mpsc 채널 개설
register_player_character(id, xp, attributes, gold) // 스탯/골드 인메모리 등록
load_player_inventory(id, character_id, auth)      // DB → 메모리로 인벤토리 로드
```

### 서버가 새 플레이어에게 돌려주는 "입장 패키지"
```rust
responses = [
  JoinSuccess { player, is_admin },   // "너는 이 캐릭터다" (자기 자신)
  GameTimeSync { is_night, datetime },// 현재 게임 시간
  NoSpawnZones { zones },             // 마을 등 스폰 금지 구역
  InventoryState { inventory },       // 인벤토리/장비
  GuardUpdated { guard },             // 방어치
  GoldUpdate { gold },                // 소지금
]
```

### add_player — 월드에 등록 + 초기 스냅샷 (player.rs:256)
```rust
players.insert(id, player)                    // 인메모리 월드에 추가
insert_player_spatial_cell(...)               // AOI 격자에 등록
send_..._except(nearby, PlayerJoined)         // 근처 사람들에게 "새 플레이어 등장" 알림
// 그리고 나에게만 보낼 초기 스냅샷 구성 (내 반경 AOI):
GameState {
  players:      근처 다른 플레이어들,
  monsters:     같은 층 + 반경 안 몬스터들,
  ground_items: 같은 층 + 반경 안 바닥 아이템들,
}
```
→ 즉 입장 순간 **내 주변만** 받는다(전 세계 아님). 이후 이동하면 `fanout_player_position_update`가 Appear/Disappear로 시야를 갱신 (→ [`5-realtime-sync.md`](./5-realtime-sync.md)).

---

## 4. 클라이언트 수신 (messageHandlers.ts)

```ts
'AuthSuccess'  → authSuccess 이벤트 → 캐릭터 선택 화면
'JoinSuccess'  → 내 캐릭터 스폰 (gameStore.currentPlayer)
'GameState'    → remotePlayerManager/monsterManager 로 근처 객체 일괄 스폰 (messageHandlers.ts:350)
'InventoryState','GoldUpdate','GameTimeSync' → 각 UI 스토어 갱신
```
클라 진입 트리거: `LoginScreen` → `socket.ts:711 requestEnterGame(characterId)`.

---

## 영속성 연결고리
- **입장 시 복원**: 위치(last_x/y/z)·방향·HP·floor·골드·인벤토리·스탯을 **DB에서 로드**.
- **게임 중 저장**: dirty 추적 → 32초마다 배치 저장 (→ [`1-server-architecture.md`](./1-server-architecture.md) 상태 저장 절).
- **퇴장 시**: 마지막 상태가 DB에 남아 다음 로그인 때 그 자리에서 재개. 던전 안 로그아웃(음수 floor)도 복원됨.

---

## 코드 대응표 (서버 ↔ 프론트)
| 단계 | 서버 | 프론트 |
|------|------|--------|
| 로그인 | `connection.rs:394 Authenticate` → `auth.rs login_google` | `socket.ts:529 authenticateWithGoogle` |
| 캐릭터 목록 | `AuthSuccess { characters }` | `messageHandlers 'AuthSuccess'` → 선택 UI |
| 스탯 굴림 | `connection.rs:534 RollCharacterStats` | `CharacterStatsRolled` 수신 |
| 캐릭터 생성 | `connection.rs:451 CreateCharacter` → `create_character` | `CharacterCreated` 수신 |
| 입장 | `connection.rs:554 EnterGame` → `add_player` | `socket.ts:711 requestEnterGame` |
| 내 캐릭터 | `JoinSuccess` | `messageHandlers 'JoinSuccess'` |
| 초기 스냅샷 | `player.rs:325 GameState` | `messageHandlers.ts:350 'GameState'` |

관련 문서: [`5-realtime-sync.md`](./5-realtime-sync.md), [`2-client-networking.md`](./2-client-networking.md), [`1-server-architecture.md`](./1-server-architecture.md)
