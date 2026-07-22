# 프론트엔드 네트워킹: 웹소켓 연결 & 통신

## 어디를 봐야 하나 (핵심 파일)

| 파일 | 역할 |
|------|------|
| `client/src/lib/network/socket.ts` | **NetworkManager** — 웹소켓 연결/재연결, 송수신, 인증 |
| `client/src/lib/network/messageHandlers.ts` | 서버 메시지(ServerMessage) → 스토어/매니저로 **디스패치** |
| `client/src/lib/network/networkTypes.ts` | `ClientMessage`/`ServerMessage` TS 타입 (Rust enum과 1:1) |
| `client/src/lib/utils/networkUtils.ts` | `getDefaultServerUrl()` — `/ws` URL 생성 |
| `client/src/lib/wasm/onlinerpg_shared` | **WASM** — MessagePack 직렬화/역직렬화 (shared 크레이트) |
| `client/src/lib/components/PlayerControl.svelte` | 입력 → `sendPlayerMove`/`sendPlayerAttack` 호출 지점 |
| `client/src/lib/managers/remotePlayerManager.ts` | 다른 플레이어 위치 보간/렌더 |
| `client/src/lib/managers/monsterManager.ts` | 몬스터 상태/렌더 |

`networkManager`는 **싱글톤**(`socket.ts` 맨 아래 `hmrSingleton`)이라 앱 전역에서 하나만 존재.

---

## 1. 연결 (Connect)

`NetworkManager.connect()` (`socket.ts:97`):

```ts
this.socket = new WebSocket(targetUrl)     // targetUrl = getDefaultServerUrl()
this.socket.binaryType = 'arraybuffer'     // 바이너리(MessagePack) 수신
```

### URL은 어떻게 정해지나 — `getDefaultServerUrl()` (`networkUtils.ts:1`)
```ts
const protocol = location.protocol === 'https:' ? 'wss:' : 'ws:'
return `${protocol}//${host}/ws`   // 예: ws://localhost:10004/ws
```
- **직접 서버(10006)에 붙지 않는다.** 브라우저는 자신이 뜬 오리진(`localhost:10004`)의 `/ws`로 접속.
- **Vite dev 프록시**가 `/ws` → `ws://127.0.0.1:10006`으로 넘김 (`client/vite.config.ts`).
- 덕분에 HTTPS/WSS, CORS, 인증서 문제를 프록시가 흡수 → 프론트 코드는 오리진만 신경 쓰면 됨.

### 콜백 4개
```ts
socket.onopen    = () => gameStore.isConnected = true
socket.onclose   = (e) => { isConnected=false; if(e.code!==1000) handleReconnect() }
socket.onerror   = () => handleReconnect()
socket.onmessage = (e) => { ...역직렬화 후 handleServerMessage... }
```

### 재연결 (`handleReconnect`, `socket.ts:166`)
- 최대 5회, `2000ms * 시도횟수` 백오프.
- 재연결 성공 시 **저장해둔 Google 토큰으로 자동 재인증 → 자동 `EnterGame`**.
- 토큰 만료(~1h) 시 `authError` → "세션 만료, 다시 로그인" 안내.

---

## 2. 통신 포맷: MessagePack (WASM 경유)

서버와 동일하게 **JSON이 아니라 MessagePack 바이너리**. 직렬화는 브라우저 JS가 아니라
**shared 크레이트를 WASM으로 빌드한 것**이 담당 (서버와 완전히 같은 로직):

```ts
// socket.ts:16-18
import { serialize_client_message, deserialize_server_message } from '../wasm/onlinerpg_shared'
```

- **보낼 때**: `ClientMessage`(JS 객체) → `serialize_client_message()` → `Uint8Array` → `socket.send()`
- **받을 때**: `ArrayBuffer` → `deserialize_server_message()` → `ServerMessage`(JS 객체)

> 그래서 `npm run dev`가 항상 `build:wasm`을 먼저 돌리는 것. shared 크레이트(Rust)가 프론트·서버 양쪽의 프로토콜 단일 소스.

`ClientMessage`/`ServerMessage` TS 타입(`networkTypes.ts`)은 Rust enum과 1:1. serde의 externally-tagged 방식이라 `{ PlayerMove: { position, ... } }` 형태.

---

## 3. 동작 흐름 (내 이동 → 서버 → 남 화면)

### 보내기 — 입력에서 서버로
```
PlayerControl.svelte (클릭/키보드)
  └─ FSM (player-control/fsm/*)      ← 이동/전투 상태머신
       └─ sendPlayerMove(pos, rot, floor, append)   (PlayerControl.svelte:371)
            └─ networkManager.sendPlayerMove(...)     (socket.ts:321)
                 └─ sendMessage({ PlayerMove: {...} }) → WASM 직렬화 → socket.send
```
공격도 동일: `sendPlayerAttack(monsterId)` → `{ PlayerAttack: { monster_id } }` (`socket.ts:300`).

### 받기 — 서버 메시지 디스패치
`socket.onmessage` → `handleServerMessage(message, ...)` (`messageHandlers.ts`)에서 **거대한 `switch`**로 분기:

```ts
case 'PlayerMoved':      remotePlayerManager.setTargetPosition(id, pos, rot)  // 보간 이동
case 'PlayerAppeared':   remotePlayerManager.initPlayer(...)
case 'PlayerDisappeared':remotePlayerManager.removePlayer(id)
case 'MonsterMoved':     monsterManager.updateMonsterFromNetwork(...)
case 'MonsterDead':      monsterManager ... (죽음 처리/루팅)
case 'GameTimeSync':     → 곧바로 sendMessage('Heartbeat')   // 생존 신호
case 'AuthSuccess':      this.authSuccess.emit(...)            // UI 이벤트
```

- **`PlayerMoved`**: 내 캐릭터면 무시(내 위치는 내가 이미 렌더). 남이면
  `remotePlayerManager.setTargetPosition()`으로 **목표 위치만 갱신** → 매 프레임 부드럽게 보간(interpolation). 서버가 200ms 간격으로 주는 위치 사이를 채워 자연스럽게 걷는 것처럼 보이게 함.
- **Heartbeat**: 서버의 `GameTimeSync`(8초 주기)를 받을 때마다 하트비트로 응답 → 서버가 "살아있음" 판단(`connection.rs`의 heartbeat 타임아웃).

---

## 4. 인증 흐름 (프론트 관점)

```
LoginScreen.svelte
  └─ Google GSI 버튼 → id_token(credential) 획득
       └─ networkManager.connect() + authenticateWithGoogle(id_token)  (socket.ts:529)
            └─ sendAndSerialize({ Authenticate: { google_id_token } })
서버 검증 성공 → ServerMessage::AuthSuccess
  └─ messageHandlers 'AuthSuccess' → authSuccess 이벤트 → 캐릭터 선택 UI
     (동시에 setApiAuthToken(id_token): REST 쓰기용 저장, networkUtils.ts)
```

---

## 요약
프론트는 **오리진의 `/ws`로 웹소켓 접속(→ Vite 프록시 → 서버 10006)**, **WASM으로 MessagePack 직렬화**해 `ClientMessage`를 보내고, `onmessage`에서 역직렬화한 `ServerMessage`를 **`switch`로 매니저/스토어에 디스패치**한다. 남의 이동은 목표 위치를 받아 **클라이언트에서 보간 렌더**한다.
