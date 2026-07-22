# 건축 & 월드 편집: 저장과 전파

## 한 줄 요약
집/지형 편집은 **REST(10007)로 저장(청크 단위 파일) + 근처엔 WebSocket 즉시 push, 멀리/미래엔 REST lazy 로딩**.
이동/전투(소켓 실시간)와 **다른 경로**다. 그리고 **SQLite DB가 아니라 파일**에 저장된다.

---

## 저장 위치 정정: DB가 아니라 청크 파일

```
data/housing/{chunk_x}_{chunk_z}/{house_id}.json     (housing/mod.rs:183)
```
- 32km² 월드를 **청크(격자)** 로 쪼개 폴더로 관리 → "이 지역 집들"만 파일 몇 개로 조회.
- `game_data.db`(SQLite)는 **플레이어 데이터**(계정/캐릭터/인벤토리/시간) 전용.
- 집·지형 같은 **월드 데이터**는 **파일**. (경량·대용량·청크 접근에 유리)

---

## 건축 경로: REST (소켓 아님)

```
클라 housingManager.createHouse()          (client/src/lib/managers/housingManager.ts:112)
  └─ POST /api/housing                      (10007, 인증 필요: admin 이메일 or NPC 토큰)
       └─ create_house                       (server/src/housing/routes.rs:87)
```
집 데이터는 크고 초당 스트림이 아니라 **가끔 일어나는 CRUD** → 요청/응답형 REST가 적합.

### 서버가 하는 일 (create_house)
```rust
validate_house(...)                    // 모양/범위 검증
validate_house_neighbors(...)          // 옆집 충돌 검사 (이웃 청크 스캔)
housing.write_house(&house)            // ① 디스크 파일 저장 (durable, 진실의 원천)
game_state.passability_add_house(...)  // ② 인메모리 충돌맵 (서버 이동 막기)
remove_house_trees(...)                // ③ 밑 나무 제거 (지형 파일 수정)
broadcast_house_change(...)            // ④ 근처 플레이어 알림 (WS)
```
저장 = **3곳**: 디스크 파일(영속) + 인메모리 passability(충돌) + 지형(나무).

---

## 전파: 어떻게 "모든 사용자"에게 퍼지나 — 2단계

전체 브로드캐스트는 **안 함**. 대신:

**① 지금 근처 플레이어 → WebSocket 즉시 push (AOI, floor 0)**
```rust
// broadcast_house_change (routes.rs:286)
send_direct_message_to_players_within_position(&house.origin, 0, EVENT_DELIVERY_RADIUS,
    ServerMessage::HouseSpawned { house }, None)
```
→ 클라 `messageHandlers.ts:726 'HouseSpawned'` → `housingManager.handleRemoteHouseSpawned` → 즉시 렌더.

**② 나중에 오는/새 접속자 → REST lazy 로딩**
```ts
fetch(`/api/housing/area/${cx}/${cz}`)   // housingManager.ts:95 → 디스크 파일 읽기
```
→ 파일에 이미 저장돼 있으니 청크 진입 시 누구든 받음. 재접속·서버 재시작 후에도 파일이 진실.

| 대상 | 방식 | 시점 |
|------|------|------|
| 지금 근처 플레이어 | WebSocket `HouseSpawned`/`HouseUpdated`/`HouseRemoved` (AOI) | 즉시 |
| 나중에 오는/새 접속자 | REST `/api/housing/area/{cx}/{cz}` (파일 읽기) | 청크 진입 시 |

**설계 의도**: "근처는 실시간 push, 멀리/미래는 저장 후 필요 시 pull" → 광활한 월드에서 O(N²) 없이 결국 일관되게 전파.

---

## 지형 편집(맵 에디터)도 같은 패턴
높이/텍스처(splat)/오브젝트 편집 = `PUT /api/terrain/*` REST 저장 + 근처엔 `TreeTilesInvalidated` 등 WS 알림.
→ **월드 편집 = REST 저장(파일) + AOI WS 알림 + REST lazy 로딩** 이라는 공통 뼈대.

---

## 코드 대응표 (서버 ↔ 프론트)
| 단계 | 서버 | 프론트 |
|------|------|--------|
| 건축 요청 | `housing/routes.rs:87 create_house` | `housingManager.ts:112 createHouse` (POST) |
| 파일 저장 | `housing/mod.rs:337 write_house` | — |
| 충돌맵 | `game_state passability_add_house` | — |
| 근처 알림 | `routes.rs:286 broadcast_house_change` → `HouseSpawned` | `messageHandlers.ts:726` → `handleRemoteHouseSpawned` |
| 지역 로딩 | `routes.rs:52 get_houses_in_chunk` (파일 읽기) | `housingManager.ts:95 fetch /api/housing/area` |
| 삭제 | `routes.rs:204 delete_house` → `HouseRemoved` | `messageHandlers.ts:740` → `handleRemoteHouseRemoved` |

관련 문서: [`1-server-architecture.md`](./1-server-architecture.md), [`5-realtime-sync.md`](./5-realtime-sync.md), [`2-client-networking.md`](./2-client-networking.md)
