# OpenMMO 서버 구조

## 한 프로세스, 두 개의 서버

`server/src/main.rs`가 포트 2개를 열어 **서로 다른 두 서버**를 한 프로세스 안에서 같이 띄웁니다.

| 서버 | 포트 | 프로토콜 | 용도 |
|------|------|----------|------|
| **게임 서버** | 10006 | WebSocket (바이너리) | 실시간 플레이 — 이동/전투/채팅/스폰 |
| **REST API 서버** | 10007 | HTTP (Axum) | 지형·주택·NPC 데이터 읽기/쓰기 (맵 에디터·봇) |

> REST API는 `127.0.0.1`에만 바인딩됩니다. 브라우저는 Vite 프록시를 통해 same-origin으로만 접근 → 외부 노출 없음.

---

## ① WebSocket 게임 서버 (핵심)

실시간 게임은 전부 여기서 돕니다. **REST가 아니라 소켓인 이유** = 서버가 클라이언트 요청 없이도 먼저 데이터를 밀어줘야 하기 때문 (다른 플레이어 이동, 몬스터 등장 등). HTTP로는 이런 push가 안 됩니다.

### 연결 1개 = tokio 태스크 1개
```rust
// main.rs — 접속마다
tokio::spawn(async move {
    handle_connection(stream, game_state, ...).await;
});
```
OS 스레드가 아니라 가벼운 async 태스크라, 5,000명 동접이 목표대로 가능한 구조입니다.

### 동작 방식: `select!` 이벤트 루프
`handle_connection`은 연결마다 루프를 돌며 **4가지를 동시에 대기**하다가 먼저 오는 걸 처리합니다:

```
tokio::select! {
  ① 이 클라이언트가 보낸 메시지   (ws_receiver)  → 이동/공격/채팅 처리
  ② 월드 전체 브로드캐스트        (broadcast)    → 남의 이동/몬스터를 이 소켓에 전달
  ③ 나에게만 온 개인 메시지       (direct_rx)    → 강퇴 등
  ④ heartbeat 타이머                            → 죽은 연결 정리
}
```
- **②번 broadcast** = "한 번 보내면 모든 구독자가 각자 받는" pub/sub. 실시간 동기화의 핵심.
- **③번 direct** = 특정 플레이어 1명에게만 가는 mpsc 채널.

### `loop { select! }`는 busy-spin이 아니다

문법만 보면 `while(true)`인데, 실제로는 event-driven입니다.

- **연결 1개 = Task 1개**. `tokio::spawn`은 OS 스레드를 만드는 게 아니라, 워커 스레드풀(기본 = CPU 코어 수)이 나눠 실행할 작업 단위(Task)를 등록하는 것. Task는 특정 스레드에 고정되지 않고, work-stealing 스케줄러가 매번 어느 워커 스레드에서 실행할지 정함.
- `select!`의 각 브랜치(`heartbeat_check.tick()`, `ws_receiver.next()` 등)를 poll 했는데 아무 것도 준비 안 됐으면 → 즉시 `Pending` 반환하고 해당 워커 스레드는 이 Task를 내려놓고 다른 Task를 실행하러 감. **CPU를 스핀하며 확인하는 게 아니라, 이벤트 소스에 waker를 걸어두고 재워두는 것.**
- 소켓 쪽 이벤트 감지는 mio(→ Linux epoll / macOS kqueue / Windows IOCP)가 담당. 데이터 도착하면 커널이 알려주고, 그 fd에 걸린 waker가 호출되어 Task가 다시 실행 큐에 올라감.
- 이 "커널에게 준비된 fd를 물어보고 콜백/Task를 깨운다"는 개념은 Node.js 이벤트 루프의 **poll phase**와 동일한 지점(둘 다 밑단은 epoll류). 차이는 Node는 단일 스레드가 timers→poll→check 같은 정해진 phase 순서를 순환하는 반면, Tokio는 phase 없이 여러 워커 스레드가 각자 리액터 확인 + Task 실행을 병렬로 함.
- 그래서 커넥션 5,000개가 다 이 루프를 돌아도 실제 CPU 사용량은 "그 순간 대기 중인 커넥션 수"가 아니라 "그 순간 실제로 이벤트가 발생해 처리 중인 Task 수"(최대 워커 스레드 수만큼 동시 실행)에 비례함.

### 메시지 포맷
JSON이 아니라 **MessagePack(바이너리)**. `shared/messages.rs`의 Rust enum(`ClientMessage`/`ServerMessage`)을 `serde`가 직렬화. 클라이언트·서버·봇이 **완전히 동일한 프로토콜** 사용 (README의 agent-human parity).

### 인증
소켓 연결 후 첫 메시지로 `Authenticate { google_id_token }` → 서버가 Google 공개키로 토큰 검증 → `sub`으로 계정 조회/생성. (봇은 `AuthenticateNpc` + 공유 토큰)

---

## ② REST API 서버 (Axum)

실시간이 아닌 **정적/대용량 월드 데이터**용. 좌표 기반으로 청크를 읽고 씁니다.

```
GET/PUT  /api/terrain/height/{x}/{z}      높이맵
GET/PUT  /api/terrain/splat/{x}/{z}       텍스처 splatmap
GET/PUT  /api/terrain/grass/{x}/{z}       풀 배치 (최대 16MB)
GET/PUT  /api/terrain/objects/{rx}/{rz}   배치된 오브젝트
GET/PUT  /api/terrain/zones/{rx}/{rz}     구역(마을/스폰)
GET      /api/terrain/trees|river|water   생성 데이터
GET/POST /api/housing, /api/housing/area  주택
GET      /api/npcs                        NPC 스케줄
```
- **읽기(GET)는 public**, **쓰기(PUT/POST/DELETE)는 인증 필요** — `api_auth.rs` 미들웨어가 NPC 토큰 또는 admin 이메일(구글 토큰)을 검사.
- 데이터는 DB가 아니라 **파일**로 저장 (`data/terrain/`, `data/housing/`).

---

## 상태 저장: 메모리가 진짜, DB는 백업

| 데이터 | 어디에 |
|--------|--------|
| 플레이어 위치·몬스터·바닥템 (실시간) | **인메모리** `GameState` (`Arc<RwLock<HashMap>>`) |
| 계정·캐릭터·인벤토리·게임시간 (영속) | **SQLite** `data/game_data.db` |
| 지형·주택·NPC | **JSON/바이너리 파일** |

`GameState`는 이동마다 DB를 안 때립니다. `main.rs`의 8초 틱이 **32초마다 "변경된(dirty)" 것만 SQLite에 배치 저장** → 성능 확보.

### SQLite 테이블 (`server/src/auth.rs`)
- `accounts` — 계정. `google_sub`(구글 고유 ID)로 식별
- `characters` — 캐릭터 스탯/레벨/HP/마지막 위치/gold
- `character_items` — 인벤토리/장비
- `world_time` — 게임 내 시간 (1행)

---

## 백그라운드 틱 (main.rs에서 spawn)

접속과 별개로 도는 주기 태스크들:

| 주기 | 하는 일 |
|------|---------|
| 200ms | 플레이어 이동 시뮬레이션 (서버 권위적 위치) |
| 8s | 게임 시간 진행 + dirty 상태 저장 |
| 10s | 몬스터 스폰 보충 |
| 30s | 던전 리필 / 바닥템 소멸 |

각 틱은 `guard_tick`으로 감싸 패닉이 나도 루프가 안 죽습니다.

---

## 부팅 순서 (main.rs를 위→아래로 읽으면)

```
main.rs
 ├─ AuthService::new()         → SQLite 오픈, 없으면 테이블 생성
 ├─ GoogleAuthVerifier::new()  → GOOGLE_CLIENT_ID 로 토큰 검증기 준비
 ├─ GameState::new()           → 인메모리 월드 상태 생성
 ├─ tokio::spawn(...) × 5      → 백그라운드 틱 태스크들
 ├─ TcpListener::bind(10006)   → WebSocket 게임 서버
 └─ TcpListener::bind(10007)   → Axum REST API
```

---

**요약**: `main.rs` 한 프로세스가 **WS 게임 서버(10006)** + **REST API(10007)**를 함께 띄우고, 실시간은 소켓 `select!` 루프로 pub/sub 하며, 상태는 메모리에 두고 SQLite/파일로 주기 백업하는 구조.

## 관련 문서
- 실시간 이동/전투 브로드캐스트: [`5-realtime-sync.md`](./5-realtime-sync.md)
- 프론트 웹소켓 연결/통신: [`2-client-networking.md`](./2-client-networking.md)

## 코드 진입점 요약
- 서버 부트: `server/src/main.rs`
- 소켓 연결 루프: `server/src/connection.rs` `handle_connection`
- 인메모리 상태: `server/src/game_state/mod.rs` `GameState`
- DB: `server/src/auth.rs` `AuthService`
- REST 라우트: `server/src/{terrain,housing,npc_schedule}/routes.rs`
- 프론트 소켓: `client/src/lib/network/socket.ts` `NetworkManager`
- 프론트 메시지 분기: `client/src/lib/network/messageHandlers.ts`

## 인증 client ID 주의 (로컬 셋업 메모)
- 클라이언트(`client/.env.local` `VITE_GOOGLE_CLIENT_ID`)와 서버(`GOOGLE_CLIENT_ID`)의 client ID가 **반드시 같아야** 토큰 `aud` 검증 통과.
- 서버 쪽 값은 `.cargo/config.toml`의 `[env]`가 주입 (셸 export가 있으면 그게 우선).
