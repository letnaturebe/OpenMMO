# 오디오 (BGM & SFX)

## 한 줄 요약
오디오는 **100% 프론트(클라이언트) 전용**. 서버는 오디오를 전혀 모른다.
각 클라가 **자기 로컬 게임 상태**를 보고 독립적으로 재생한다.

## 왜 프론트 전용?
음악/효과음은 게임 로직에 영향 없는 **순수 연출**. 사람마다 볼륨·타이밍이 달라도 무방.
서버는 "몬스터가 죽었다" 같은 **사실**만 보내고, 소리를 낼지는 클라가 결정.

---

## BGM (배경음악) — `client/src/lib/managers/bgmManager.ts`

### 평상시 (ambient)
- `BGM_FILES`(~50곡)를 **셔플 재생**.
- 한 곡 끝 → 짧은 gap(쉼) → 다음 곡. 리스트 끝나면 다시 셔플(`shufflePlaylist`).

### 전투 (battle)
- 로컬 플레이어가 몬스터를 **타겟팅**하면 배틀곡으로 전환.
- 전투 종료 시 즉시 끄지 않고 **linger(잠깐 유지) → crossfade(페이드)** 로 ambient 복귀.
  (`battleLingerTimer` / `battleFadeTimer` / `battleQuietTimer`)

### 트리거는 로컬 판정 (서버 아님)
```
CombatController (combatController.ts)
  beginCombat(monsterId)  → wasInCombat 아니면 startBattleMusic()   (line 46)
  cancelCombat()          → wasInCombat 이면 stopBattleMusic()      (line 54)
  isInCombat = (_targetMonsterId !== null)
```
→ "내가 몬스터를 노리고 있나"가 곧 전투 상태. **각 클라가 스스로 판단** → 플레이어마다 독립적으로 배틀 음악이 켜짐.

---

## SFX (효과음) — `client/src/lib/managers/sfxManager.ts`
- 칼 명중/빗나감 등 **로컬 이벤트**에 반응 (`playSwordHitSound`, `playSwordMissSound`).
- **풀(pool)** 방식: 오디오 객체 여러 개를 미리 만들어 돌려씀 → 연타 시 소리가 겹쳐도 안 끊김.
- 자재별 히트음 등 URL 파라미터로 교체 가능.

---

## 공통
- 에셋: `/bgm/…mp3`, `/sfx/…` 같은 **정적 파일**(public)에서 로드.
- 설정: 볼륨/온오프는 `SettingsPanel.svelte`, 개별 오디오에 `applyAudioSettings`.
- 브라우저 정책: **첫 사용자 제스처(클릭) 전엔 재생 불가** → play() 실패를 조용히 무시하고 이후 재시도.

## 코드 위치
- `bgmManager.ts` — ambient 플레이리스트 + battle 전환/crossfade
- `sfxManager.ts` — 효과음 풀
- `combatController.ts` — 전투 상태 → BGM 트리거
- `SettingsPanel.svelte` — 볼륨/설정 UI

> 자산 출처/라이선스는 `doc/ASSETS.md` 참고. 서버 코드에는 오디오가 없음.
