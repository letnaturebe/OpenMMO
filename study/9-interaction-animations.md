# 인터랙션 애니메이션 전파 (held vs transient)

케이스 스터디: PR #4 "Show the pickup crouch to nearby players" (업스트림 Julian-adv/OpenMMO).
[`5-realtime-sync.md`](./5-realtime-sync.md)의 AOI 브로드캐스트가 애니메이션에 어떻게 적용되는지 보여주는 예.

## 문제
아이템 줍기 **웅크리기(crouch) 애니메이션이 로컬 전용**이었다.
→ 남이 주우면 근처 플레이어 눈에는 캐릭터가 가만히 선 채 **아이템만 사라지는** 어색한 장면.

## 접근: 새 메시지 없이 기존 interaction 브로드캐스트 재사용
`PlayerInteractionChanged` 메시지의 `object_type: Option<String>` 필드에 `"pickup"`을 실어
근처(AOI) 플레이어에게 push. 프로토콜 변경 0.

### 서버 (`server/src/game_state/inventory.rs`)
```rust
async fn broadcast_pickup_animation(&self, player_id, position, floor_level) {
    send_direct_message_to_players_within_position(
        position, floor_level, EVENT_DELIVERY_RADIUS,
        ServerMessage::PlayerInteractionChanged { object_type: Some("pickup".into()), .. },
    )
}
```
- 줍기 성공 경로 2곳에서 호출.
- **transient(일시적)**: 서버가 상태를 **저장하지 않고**, `StopInteraction`도 안 보냄.

### 클라 (`client/src/lib/components/PlayerModel.svelte`)
```
조건: (isCurrentPlayer || interactionAnim === 'pickup')
```
- "애니 끝나면 idle 복귀" 콜백이 원래 내 캐릭터에만 적용 → `|| pickup` 추가.
- 원격 플레이어도 pickup을 **one-shot 재생 후 스스로 idle 복귀** (`StopInteraction` 안 기다림).
- 수신 경로: `messageHandlers.ts:118 handleInteraction` → `remotePlayerManager.handleInteraction` (기존 재사용).
- `"pickup"` 문자열은 클라 catalog fallback이 해당 클립으로 해석.

---

## 핵심 개념: held vs transient interaction

| 종류 | 예 | 서버 상태 저장 | 종료 방식 | 늦게 온 사람 |
|------|----|:-------------:|-----------|-------------|
| **held (지속)** | 벤치 앉기, 화로 사용 | O | 명시적 `StopInteraction` | 그 포즈를 봐야 함 → 저장 필요 |
| **transient (일시적)** | **pickup** | X | 클라 one-shot 자동 종료 | 안 봐도 됨 → 저장하면 오히려 버그 |

**판단 근거**: pickup을 held처럼 저장하면, 늦게 접속한 사람이 이미 끝난 동작의 **"영원히 웅크린" 포즈**를 보게 된다. 순식간에 끝나는 동작은 저장하지 않는 게 맞다.

## 검증
서버 테스트 `pickup_broadcasts_the_pickup_animation()` — 근처 플레이어가 `object_type: "pickup"` 수신하는지 확인.

## 일반 원칙
- 연출성 애니메이션도 "남에게 보여야 하면" → **AOI 브로드캐스트** (전체 아님).
- 지속/일시 구분이 곧 "저장할까 말까"를 가른다. 애매하면 "늦게 온 사람이 이 상태를 봐야 하나?"로 판단.

관련 문서: [`5-realtime-sync.md`](./5-realtime-sync.md), [`7-monster-ai-ownership.md`](./7-monster-ai-ownership.md)

> 참고: 이 PR은 현재 로컬 체크아웃엔 미머지 상태 (열린 PR).
