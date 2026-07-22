# OpenMMO 코드베이스 학습 노트

코드 흐름 중심으로 정리한 스터디 문서 모음. **위에서 아래 순서대로** 읽으면 기본 → 심화로 이어짐.
(설계 의도 중심 문서는 저장소 루트 `doc/` 폴더 참고 — 상호 보완됨.)

## 읽는 순서

### 기초: 큰 그림
1. **[1-server-architecture.md](./1-server-architecture.md)** — 서버 전체 구조. 한 프로세스 안 WS 게임서버(10006) + REST API(10007), 인메모리 vs SQLite vs 파일, 백그라운드 tick 개요, 부팅 순서.
2. **[2-client-networking.md](./2-client-networking.md)** — 프론트가 어떻게 붙나. `/ws` 접속(→Vite 프록시→서버), WASM MessagePack 직렬화, `onmessage` → `switch` 디스패치.

### 진입: 게임에 들어가기까지
3. **[3-game-entry-lifecycle.md](./3-game-entry-lifecycle.md)** — 로그인(Google) → 캐릭터 생성/스탯 → EnterGame → 초기 스냅샷(AOI). DB 복원/저장 연결고리.

### 실시간: 게임 중 동작
4. **[4-server-ticks.md](./4-server-ticks.md)** — tick 모델. "매 프레임 전부 계산"이 아니라 주기별 독립 tick + delta + AOI. 이동은 200ms, 서버 권위적.
5. **[5-realtime-sync.md](./5-realtime-sync.md)** — 이동·전투 브로드캐스트. 전체가 아니라 근처(AOI)에게만 push. 서버↔프론트 코드 대응표.

### 시스템별
6. **[6-housing-and-world-edits.md](./6-housing-and-world-edits.md)** — 건축/지형 편집. REST 저장(청크 파일, DB 아님) + 근처 WS 알림 + 원거리 REST lazy 로딩.
7. **[7-monster-ai-ownership.md](./7-monster-ai-ownership.md)** — 몬스터 AI 소유권. AI 시뮬은 소유자 클라, 검증·데미지·권위는 서버. (서버가 지형을 몰라 위임)
8. **[8-audio.md](./8-audio.md)** — BGM/SFX. 100% 프론트 전용, 로컬 게임 상태 기반.
9. **[9-interaction-animations.md](./9-interaction-animations.md)** — 인터랙션 애니메이션 전파 (PR #4 케이스). held vs transient 구분, pickup 크라우치를 근처에 AOI 브로드캐스트.

---

## 핵심 관통 개념 (문서 전반에 반복 등장)
- **인메모리가 진실, DB/파일은 백업**: 실시간 상태는 `GameState` HashMap, 32초마다 dirty 배치 저장.
- **AOI(Area of Interest)**: 전체 브로드캐스트 대신 근처(반경+같은 층) 플레이어에게만 direct 채널로 push. 5000명 확장의 핵심.
- **서버 권위(server-authoritative)**: 위치 전진·데미지·검증은 서버가. 클라는 의도/AI만.
- **채널 2종**: global `broadcast`(모두-예: 시간) vs per-player `direct_channel`(mpsc, AOI/개인).
- **프로토콜 단일 소스**: `shared` 크레이트(Rust) → 서버 네이티브 + 클라 WASM 공용, MessagePack.

## 진입점 빠른 참조
- 서버 부트: `server/src/main.rs`
- 소켓 루프: `server/src/connection.rs` `handle_connection`
- 인메모리 상태: `server/src/game_state/` (`mod.rs`, `player.rs`, `combat.rs`, `monster.rs`)
- DB: `server/src/auth.rs`
- REST: `server/src/{terrain,housing,npc_schedule}/routes.rs`
- 프론트 소켓: `client/src/lib/network/{socket,messageHandlers}.ts`
- 프론트 매니저: `client/src/lib/managers/`

---

## 다음에 볼 후보 (미작성)
- **전투 상세**: 스탯/주사위/데미지 공식 (`server/src/game/combat.rs`, D&D식) + `doc/COMBAT.md`
- **상태 영속화 상세**: dirty 추적 → 배치 저장이 정확히 뭘 저장하나 (`collect_dirty_*`, `save_characters_batch`)
- **던전 인스턴스**: 층 기반 인스턴싱 (`game_state/dungeon.rs`)
- **인벤토리/상거래**: `inventory.rs`, `trading.rs`, `deals.rs`
- **월드 생성**: 절차 지형/강/바이옴 (`terrain` 크레이트 + `doc/TERRAIN_GENERATION.md`)
