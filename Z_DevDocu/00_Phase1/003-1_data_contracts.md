# 003-1 — Data Contracts (P1 SSOT)

| 항목 | 내용 |
|------|------|
| 문서 번호 | 003-1 |
| 목적 | L1(UI)·L2(엔진 Wrapper)·L3(복기 LLM) 구현이 동일한 JSON 계약으로 결합되도록 보장 |
| 선행 문서 | `001-1_product_plan.md`, `002-1_omok_rules.md` |
| 적용 범위 | P1 (엔진 통합·`GameLog` 저장까지) |
| 상태 | 초안 — P1 SSOT |
| 작성일 | 2026-07-31 |

---

## 1) 계약 원칙

- **SSOT:** 데이터 구조는 본 문서가 유일 기준이다.
- **호환성:** 하위 호환 깨짐 시 `schema_version` major 증가.
- **검증:** L1·L2·L3 간 주고받는 모든 JSON은 본 문서의 스키마 규칙을 통과해야 완료로 인정한다.
- **결정론:** 동일 `replay_seed` + 동일 착수 시퀀스면 동일 `GameLog`를 생성해야 한다(`002` §6~8과 결합).
- **계층 분리 (`001-1` §6.2 원칙 계승):** 합법성 판정(002) 필드와 코칭 품질 필드(threat_type 세부, heuristic_score 등, 004에서 확장)를 섞지 않는다 — 본 문서는 **002가 보장하는 최소 계약 + 001-1이 요구하는 코칭 계약의 접합면**만 정의한다.

---

## 2) 공통 규약

### 2.1 JSON 공통 필드

| 필드 | 타입 | 규칙 |
|------|------|------|
| `schema_version` | string | 형식 `major.minor.patch` (예: `1.0.0`) |
| `ruleset_id` | string | `omok_freestyle_15_v1` \| `omok_renju_15_v1` |
| `created_at_est` | string | `YYYY-MM_DD HH:MM:SS EST` |

### 2.2 공통 Enum

| 키 | 값 |
|----|----|
| `player_id` | `P0`(흑, 항상 선), `P1`(백) |
| `stone` | `EMPTY`, `BLACK`, `WHITE` |
| `direction_id` | `H`, `V`, `D1`, `D2` (`002` §3.2) |
| `forbidden_reason` | `none`, `double_three`, `double_four`, `overline` (`002` §9.2) |
| `forbidden_enforcement` | `block_placement`(MVP 기본), `instant_loss`(Post-MVP, `002` §8.9) |
| `event_type` | `SETUP`, `PLACE_STONE`, `INVALID_MOVE`, `FORBIDDEN_REJECTED`, `FIVE_DETECTED`, `DRAW`, `ROUND_END` (`002` §9.1) |
| `end_reason` | `FIVE`, `DRAW` |
| `threat_type` | `open_three`, `open_four`, `double_three`, `double_four`, `four`, `vcf_sequence`, `five` — **정식 카탈로그는 004에서 확정**, 본 문서는 필드 타입만 고정(string enum, 값 추가는 minor 변경) |
| `risk_level` | `low`, `medium`, `high` |
| `critical_code` | `C1`..`C5` (`001-1` §8.2) |
| `persona_id` | `sunsaeng`(기본), `pro`, `short` (`001-1` §9.3) |

### 2.3 `cell_id` 규칙

- 정규식: `^R(0[0-9]|1[0-4])C(0[0-9]|1[0-4])$`
- 의미: `R{RR}C{CC}` (세부는 `002-1` §3.1)

---

## 3) 핵심 엔티티 계약

## 3.1 Stone (좌표 1칸)

```json
{
  "cell_id": "R07C07",
  "row": 7,
  "col": 7,
  "stone": "BLACK"
}
```

검증 규칙:
- `cell_id` 파싱 결과와 `row`/`col` 값이 일치해야 한다.
- `stone == "EMPTY"`인 칸은 `BoardState.stones` 배열에 **포함하지 않는다**(희소 표현, §3.2).

## 3.2 BoardState

```json
{
  "size": 15,
  "stones": [
    { "cell_id": "R07C07", "row": 7, "col": 7, "stone": "BLACK" },
    { "cell_id": "R07C08", "row": 7, "col": 8, "stone": "WHITE" }
  ],
  "move_count": 2
}
```

검증 규칙:
- `size == 15` (P1 범위, `002` §1.3).
- `stones[].cell_id` 중복 금지.
- `move_count == stones.length` (착수 = 배치, 제거 없음).

## 3.3 BoardStateSnapshot (L1 → L2)

```json
{
  "schema_version": "1.0.0",
  "ruleset_id": "omok_renju_15_v1",
  "match_id": "M_2026-07-31_0001",
  "replay_seed": "seed-001",
  "move_index": 12,
  "active_player_id": "P0",
  "board": {
    "size": 15,
    "stones": [],
    "move_count": 12
  },
  "features": {
    "forbidden_enforcement": "block_placement"
  },
  "created_at_est": "2026-07_31 14:00:00 EST"
}
```

검증 규칙:
- `active_player_id`는 `P0`, `P1` 중 하나.
- `board.move_count == move_index`.
- `ruleset_id == omok_freestyle_15_v1`이면 `features.forbidden_enforcement`는 무의미(값은 유지하되 L2가 무시).

## 3.4 MoveEvent (`GameLog` 원소, `002` §9 대응)

```json
{
  "move_index": 12,
  "player_id": "P0",
  "event_type": "PLACE_STONE",
  "cell_id": "R07C08",
  "run_length_by_direction": { "H": 2, "V": 1, "D1": 1, "D2": 3 },
  "is_five": false,
  "is_overline": false,
  "open_three_count": 0,
  "four_count": 0,
  "forbidden_reason": "none",
  "heuristic_score": 18,
  "threat_type": null,
  "risk_level": "low",
  "timestamp_est": "2026-07_31 14:00:03 EST"
}
```

검증 규칙:
- `move_index`는 0 이상 정수, 같은 `match_id` 내 단조 증가.
- `event_type == "FORBIDDEN_REJECTED"`인 경우 **해당 착수는 보드에 반영되지 않는다** — `move_index`는 소비하지 않고(`002` §5.2), 별도 `attempt_log`(§3.4.1, 선택)로만 남긴다.
- `event_type == "PLACE_STONE"`일 때만 `run_length_by_direction`, `is_five`, `is_overline`, `open_three_count`, `four_count`, `forbidden_reason`이 의미를 가진다(그 외 이벤트는 `null` 허용).
- `heuristic_score`, `threat_type`, `risk_level`은 **L2(엔진 Wrapper) 산출 필드** — 002 합법성 판정과 무관, 004에서 채워진다(P1 초기값은 `null` 허용).

### 3.4.1 ForbiddenAttemptEvent (선택, 디버그·코칭용)

```json
{
  "move_index_context": 12,
  "player_id": "P0",
  "attempted_cell_id": "R05C05",
  "forbidden_reason": "double_three",
  "timestamp_est": "2026-07_31 14:00:01 EST"
}
```

검증 규칙:
- `move_index_context`는 **직전 정상 `move_index`** — 이 시도 자체는 턴을 소비하지 않으므로 자체 `move_index`를 갖지 않는다.
- MVP는 이 이벤트를 `GameLog.events`가 아닌 **별도 배열 `GameLog.forbidden_attempts[]`**(선택, 로컬 디버그·코칭 소재용)로 저장한다 — `002` §8.9의 "왜 여기 못 두는지" 코칭에 재사용 가능.

## 3.5 GameLog

```json
{
  "schema_version": "1.0.0",
  "ruleset_id": "omok_renju_15_v1",
  "match_id": "M_2026-07-31_0001",
  "replay_seed": "seed-001",
  "events": [],
  "forbidden_attempts": [],
  "result": {
    "winner_player_id": "P0",
    "end_reason": "FIVE",
    "final_board_move_count": 37
  },
  "created_at_est": "2026-07_31 14:05:00 EST"
}
```

검증 규칙:
- `events`는 최소 1개(`SETUP` 포함).
- 마지막 유효 이벤트는 `FIVE_DETECTED` 또는 `DRAW`를 동반한 `ROUND_END`.
- `result.end_reason == "DRAW"`이면 `result.winner_player_id`는 `null`.
- `result.final_board_move_count`는 `events` 중 `event_type == "PLACE_STONE"` 개수와 일치.

## 3.6 CriticalMoveSummary (`001-1` §8.2 대응)

```json
{
  "move_index": 9,
  "critical_code": "C2",
  "reason_code": "ALLOW_OPEN_FOUR",
  "score_swing": -1,
  "fact_summary": "상대의 열린 3을 방어하지 못해 다음 수 필승 허용",
  "recommended_cell_id": "R06C09"
}
```

검증 규칙:
- `critical_code`는 `C1`..`C5` (`001-1` §8.2).
- `score_swing`은 정수(음수 허용, 부호 의미는 004에서 확정) — P1 초기 구현은 `heuristic_score` 전후 차를 그대로 사용 가능.
- `recommended_cell_id`는 §2.3 `cell_id` 정규식을 따른다.

## 3.7 ReviewPayload (L3 입력, `001-1` §9.2 대응)

```json
{
  "schema_version": "1.0.0",
  "ruleset_id": "omok_renju_15_v1",
  "match_id": "M_2026-07-31_0001",
  "winner_player_id": "P0",
  "end_reason": "FIVE",
  "critical_move_summaries": [],
  "session_context": {
    "session_streak": 1,
    "persona_id": "sunsaeng"
  },
  "created_at_est": "2026-07_31 14:06:00 EST"
}
```

검증 규칙:
- `critical_move_summaries.length` 범위 `1..3` (`001-1` §9.2 "1~3 묶음").
- 전체 직렬화 목표: `<= 2k tokens` (P2 측정, GoStop `001-1` §9.2.1과 동일 정책).
- **전체 `GameLog.events` 원문 포함 금지.**

## 3.8 CoachingPayload (L2 → L2.5, 인게임 전용, `001-1` §9.1 대응)

```json
{
  "schema_version": "1.0.0",
  "recommended_cell_id": "R06C08",
  "reason_code": "BLOCK_OPEN_FOUR",
  "threat_type": "open_four",
  "confidence_level": "high",
  "forbidden_cells": ["R05C05", "R09C03"]
}
```

검증 규칙:
- `reason_code`는 문자열 enum(정식 카탈로그는 006 template-registry 문서에서 확정, `001-1` §19 문서 목록 6번).
- `forbidden_cells`는 `active_player_id == P0` AND `ruleset_id == omok_renju_15_v1`일 때만 채워짐 — 그 외에는 빈 배열(`002` §8.9 배치 차단 UI 연동용).
- `confidence_level`은 `high` \| `medium` (`001-1` §9.1).

---

## 4) 엔진 통합(004) 필수 계약 매핑

| 004 작업 단위 | 생성/소비 계약 |
|----------------|----------------|
| KataGomo 프로세스 Wrapper | `BoardStateSnapshot` → 엔진 입력 변환, 엔진 출력 → `recommended_cell_id`/`heuristic_score` |
| threat classifier | `MoveEvent.threat_type`, `CoachingPayload.threat_type` 채움 |
| 합법성 판정(002 이식) | `MoveEvent.is_five`/`is_overline`/`forbidden_reason` |
| `GameLog` 저장 | `GameLog` 전체 직렬화, `forbidden_attempts[]` 포함 |
| Critical Move 선별 | `CriticalMoveSummary[]`, `ReviewPayload` |

---

## 5) 버전 정책

| 상황 | 조치 |
|------|------|
| 필드 추가(옵션) | minor 증가 (예: `1.1.0`) |
| 필드 삭제/타입 변경 | major 증가 (예: `2.0.0`) |
| 오탈자/설명 보강 | patch 증가 (예: `1.0.1`) |
| `threat_type`·`reason_code` enum 값 추가 | minor 증가(문자열 enum이므로 하위 호환 유지) |

- `002` 룰 변경(예: §8.8 틈형 활삼 정식 지원)으로 판정 필드 의미가 깨지면 `ruleset_version`과 함께 `schema_version` major를 올린다.

---

## 6) Done 정의 (003-1)

- [ ] 004 엔진 통합 구현이 본 계약 필드를 사용한다.
- [ ] 샘플 `GameLog` 10개(자유오목 5·렌주 5)를 스키마 검증기로 통과한다.
- [ ] `ReviewPayload`에 전체 `GameLog.events`가 포함되지 않는다.
- [ ] `schema_version`, `ruleset_id`, `created_at_est`가 모든 루트 객체에 존재한다.
- [ ] `002` 골든 테스트 G01~G10(`002-1` §10)이 본 스키마로 표현 가능함을 확인한다.

---

*본 문서는 L1·L2·L3 간 결합면(Interface) SSOT이다. 판정 로직은 002, 제품 방향은 001-1을 따른다.*
