# 홀덤 예습/복습 프로그램 아키텍처 (초안 v0.1)

## 1. 목표
- **예습 모드**: 종료된 게임의 각 순간(스냅샷)에서 최선의 선택, 이유, 출처, 실제 이후 전개를 보여준다.
- **복습 모드**: 결과를 숨기고 내가 선택 → 정오답 채점 → 틀린 판단 유형/내 이유/논쟁 여부 기록.
- **지식 보완 루프**: 규칙(내 자료)과 GTO(프리플랍)가 얼마나 일치했는지 횟수를 집계해 자료를 개선.
- **상대 프로파일링**: 핸드 데이터에서 스탯을 계산해 성향(공격/수비 등)으로 분류, 성향별 정답 분기.

## 2. 설계 원칙
1. **결정론적 데이터 우선**: 핸드 히스토리 텍스트 파싱을 1순위, 스크린샷(비전)은 보조 + 사람 검증.
2. **정답은 코드가 아니라 데이터**: 프리플랍 레인지(JSON), 포스트플랍 규칙(규칙 DB)로 분리. 엔진은 조회만 한다.
3. **결과 ≠ 정답**: 한 핸드 승패가 아니라 "규칙/GTO 일치 여부 + EV" 기준으로 집계.
4. **표본 신뢰도 표시**: 플레이어 스탯은 항상 핸드 수와 함께 표시, 표본 적으면 "불확실".
5. **개인 로컬 사용**: SQLite + 로컬 웹앱. GTO Wizard 데이터는 수동 캡처 후 개인 학습용으로만 사용.

## 3. 전체 구조

```
 [입력]                    [처리]                         [저장]              [사용]
 핸드 히스토리(txt) ─┐
 스크린샷(png)  ─────┼─▶ Ingest ─▶ Normalizer ─▶ Situation Builder ─▶ SQLite ─▶ 예습 모드
 수동 입력 ──────────┘     (파서/비전)   (공통 핸드 모델)  (스냅샷+태깅)              │ 복습 모드
                                    │                         ▲                │ 통계/대시보드
 GTO Wizard 캡처 ─▶ Range Importer ─┼─▶ preflop_ranges        │                └ 지식 관리
 내 자료(노트/PDF) ─▶ Rule Authoring ─▶ rules ───────────────┘
                                    └▶ Player Stats Engine ─▶ player_stats / archetypes
```

## 4. 모듈

| 모듈 | 책임 | 비고 |
|---|---|---|
| `ingest/` | 핸드 히스토리 파서(사이트별 어댑터), 스크린샷 → 구조화(비전) + 검증 UI | 어댑터 패턴. 사이트 미정 |
| `domain/` | Hand, Street, Action, Card, Board, Position 모델, 팟/스택/SPR 계산 | 순수 로직, 테스트 용이 |
| `situations/` | 핸드를 의사결정 지점(스냅샷)으로 분해, 태그 부여(포지션, 스트리트, 보드 텍스처, SPR, 팟 유형, 상대 유형) | 예습/복습의 단위 |
| `ranges/` | 프리플랍 레인지 저장/조회 (169핸드 × 액션 빈도), 캡처 → JSON 임포트 | GTO Wizard 수동 캡처 |
| `knowledge/` | 규칙 DB: 조건 → 권장 액션, 출처, 논쟁 여부/이유, 근거 집계 | 포스트플랍 정답의 근거 |
| `advisor/` | 정답 결정: 프리플랍=ranges, 포스트플랍=rules, 상대 유형 보정 → 권장 액션 + 이유 + 출처 | 신뢰도 점수 포함 |
| `profiling/` | VPIP, PFR, 3벳, c-bet, AF, WTSD 등 계산, 성향 분류, 신뢰도 | 베이지안 스무딩 |
| `study/` | 예습 세션, 복습 세션, 출제 로직(유사 상황 매칭, 간격 반복), 채점 | 두 모드의 핵심 |
| `journal/` | 오답 원인 분류, 내 이유 메모, 맞은 이유, 논쟁 기록 | 학습 노트 |
| `api/` + `web/` | FastAPI + 웹 UI | 로컬 실행 |

## 5. 데이터 모델 (SQLite)

```
hands(id, source, site, played_at, game_type, stakes, stack_bb, raw_text, screenshot_path, verified)
hand_players(hand_id, seat, position, nickname, stack_bb, hole_cards?)
actions(id, hand_id, street, order_no, seat, action, amount_bb, pot_before_bb)
boards(hand_id, flop, turn, river)
results(hand_id, seat, net_bb, showdown, won)

situations(id, hand_id, street, action_index, hero_seat, pot_bb, spr, stack_bb,
           hero_cards, board, facing_action, tags_json, actual_action, actual_outcome)

preflop_ranges(id, spot_key, positions, stack_bb, action_seq, grid_json, source, captured_at)
   -- spot_key 예: "BTN_open", "BB_vs_BTN_open", "BTN_open_vs_BB_3bet"

rules(id, title, street, condition_json, recommended_action, rationale,
      source_ref, source_type, disputed BOOL, dispute_note, created_at, updated_at)
rule_evidence(id, rule_id, situation_id, gto_agreed BOOL?, result_bb, note)
   -- 규칙 적용 횟수/일치율/평균 손익을 집계

player_stats(player_key, hands, vpip, pfr, three_bet, fold_to_3bet, cbet_flop,
             fold_to_cbet, af, wtsd, updated_at)
archetypes(id, name, criteria_json)            -- TAG, LAG, 니트, 콜링스테이션 등
player_archetype(player_key, archetype_id, confidence)

study_sessions(id, mode, started_at, ended_at)
attempts(id, session_id, situation_id, chosen_action, correct_action, is_correct,
         answered_at, time_ms, error_type, my_reason, review_note, next_due_at)
```

## 6. 핵심 흐름

**A. 데이터 입력**
핸드 파일 → 파서 → 핸드 저장 → 의사결정 지점 분해(situations) → 태그 부여 → 플레이어 스탯 갱신.

**B. 정답 결정 (advisor)**
1. 프리플랍: spot_key로 `preflop_ranges` 조회 → 내 핸드의 액션 빈도(혼합이면 빈도 표시).
2. 포스트플랍: 조건에 맞는 `rules` 검색 → 우선순위/신뢰도 정렬 → 권장 액션 + 이유 + 출처.
3. 상대 유형 보정: 상대 아키타입에 따라 규칙의 변형 적용(예: 콜링스테이션에게는 블러프 감소).
4. 근거 없으면 "자료 없음"으로 표시하고 자료 보완 목록에 추가.

**C. 예습**
상황 카드 표시 → 정답/이유/출처/실제 전개 공개 → 규칙 근거 기록(evidence) → 상대 성향 퀴즈(스탯 보고 유형 맞히기).

**D. 복습**
출제(예습한 유사 상황 + 오답 우선, 간격 반복) → 선택 → 채점 → 정답/최선과의 차이 → 오답 원인 분류 + 내 이유 입력 → 맞았으면 이유 기록, 논쟁 규칙이면 논쟁 사유 표시.

**E. 지식 보완 루프**
규칙별 적용 횟수, GTO 일치율, 평균 손익, 내 정답률 집계 → 일치율 낮은 규칙을 "검토 필요" 큐에 표시.

## 7. 기술 스택 (제안)
- Python 3.12, FastAPI, SQLite(SQLAlchemy), Pydantic
- 웹 UI: HTMX + Jinja (가볍게) 또는 React. 초기엔 HTMX 추천.
- 비전 추출: Claude API (스크린샷 → JSON), 사람 검증 화면 필수
- 테스트: pytest

## 8. 디렉터리 구조 (제안)
```
holdem-study/
  docs/ARCHITECTURE.md
  data/            # DB, 레인지 JSON, 캡처 이미지 (gitignore)
  src/holdem/
    domain/ ingest/ situations/ ranges/ knowledge/
    advisor/ profiling/ study/ journal/ api/ web/
  tests/
```

## 9. 미결정 사항 (답변 필요)
1. 핸드 데이터 형태: 핸드 히스토리 텍스트 가능 여부, 사이트 이름
2. 보유 자료 형태(PDF/노트/영상 등)와 분량
3. 게임 종류: 6-max 캐시 / 토너먼트 / 헤즈업, 스택 깊이
4. 저장소: 새 저장소 이름
5. 웹 UI 선호(HTMX vs React)

## 10. 해야 할 일 (작업 목록)

### Phase 0. 준비
- [ ] 새 저장소 생성 및 연결
- [ ] 미결정 사항 5개 확정
- [ ] 프로젝트 골격, 린트/테스트 설정

### Phase 1. 도메인 + 입력 (MVP 기반)
- [ ] 도메인 모델(Hand/Action/Position 등) 구현
- [ ] 핸드 히스토리 파서 1개 사이트 어댑터
- [ ] DB 스키마/마이그레이션
- [ ] 의사결정 지점 분해 + 태깅
- [ ] 수동 입력 폼(파서 없는 경우 대비)

### Phase 2. 프리플랍 레인지
- [ ] 캡처 파일명 규칙, JSON 스키마 확정
- [ ] 캡처 → 격자 JSON 변환 + 합계 검증 도구
- [ ] RFI 5개 + 오픈 대응 15개 임포트
- [ ] 레인지 조회 API

### Phase 3. 지식(규칙) DB
- [ ] 규칙 스키마/입력 UI(출처, 논쟁 필드)
- [ ] 내 자료를 규칙으로 변환(초기 20~50개)
- [ ] advisor: 프리플랍 + 포스트플랍 정답 결정
- [ ] rule_evidence 집계(적용 횟수, 일치율, 평균 손익)

### Phase 4. 예습 모드
- [ ] 상황 카드 UI, 정답/이유/출처/실제 전개 표시
- [ ] 규칙 근거 자동 기록
- [ ] 검토 필요 규칙 목록

### Phase 5. 복습 모드
- [ ] 출제 로직(유사 상황 매칭, 오답 우선, 간격 반복)
- [ ] 채점 + 최선과의 차이 표시
- [ ] 오답 원인 분류 + 내 이유 메모 + 논쟁 기록

### Phase 6. 플레이어 프로파일링
- [ ] 스탯 계산(VPIP, PFR, 3벳, c-bet, AF 등) + 신뢰도
- [ ] 아키타입 정의/분류
- [ ] 예습: 성향 맞히기 퀴즈
- [ ] 복습/advisor에 성향별 보정 적용

### Phase 7. 확장
- [ ] 스크린샷 비전 추출 + 검증 UI
- [ ] 통계 대시보드(정답률, 약점 상황, 규칙 일치율)
- [ ] 스택 깊이/스퀴즈/림프 팟 레인지 확장
- [ ] 로컬 솔버 연동(선택)

## 11. 스크린샷 입력 설계 (앱 내장 통계 활용)

### 앱이 제공하는 정보
- 좌석별 닉네임, 스택(BB 단위), 딜러 버튼, SB/BB 칩, 팟, 베팅 칩 금액, 내 홀카드
- 닉네임 옆 `V n` = 프리플랍 참여 빈도(VPIP)로 추정 (확인 필요)
- 앱 내부 기능으로 c-bet, 3벳 등 상세 통계를 제공 → **직접 계산하지 않고 캡처해서 저장**

### 프로파일링 데이터 소스 (2단계)
1. **앱 통계 캡처 (1차)**: 플레이어 상세 통계 화면을 캡처 → 비전으로 추출 → `player_stats`에 `source='app'`, `captured_at`, 표본 수(앱이 표시하면)와 함께 저장
2. **자체 계산 (2차, 선택)**: 핸드 히스토리가 쌓이면 직접 계산해 `source='computed'`로 저장, 앱 값과 비교

앱 통계는 플레이어가 바뀌어도 매번 즉시 얻을 수 있어서 "닉네임별 표본 부족" 문제가 크게 줄어든다.
단, 앱이 통계를 어떤 기간/표본으로 계산하는지 불명확하므로 `captured_at`과 표본 수를 항상 기록한다.

### 스크린샷 종류와 추출 항목
| 종류 | 추출 | 용도 |
|---|---|---|
| 핸드 중 내 차례 | 포지션, 스택, 팟, 베팅, 홀카드, 보드, 좌석 상태, V 값 | 상황 카드, 정답 조회 |
| 플레이어 상세 통계 | VPIP, PFR, 3벳, c-bet, 폴드투c-bet, AF, WTSD 등 | player_stats 저장 |
| 핸드 종료/결과 | 쇼다운 카드, 손익 | 실제 전개/결과 |

### 처리 파이프라인
```
이미지 → 종류 분류 → 비전 추출(JSON) → 자동 검증(팟=칩 합계, 스택 범위, 카드 중복)
      → 확인/수정 UI → 저장(situations / player_stats)
```
- 유효 스택 = min(내 스택, 상대 스택). 가장 가까운 스택 깊이의 레인지를 사용하고 차이를 표시.
- 좌석 순서와 버튼 위치로 포지션 계산, 폴드한 좌석은 비활성 표시로 판단(불확실하면 확인 UI에서 수정).

### 추가 작업 항목
- [ ] 이 앱 화면 샘플 수집: 프리플랍/플랍/턴/리버 내 차례, 통계 상세 화면, 결과 화면
- [ ] `V` 값의 의미와 통계 화면 항목 확정
- [ ] 화면 종류 분류기 + 종류별 추출 프롬프트/스키마
- [ ] 자동 검증 규칙 + 확인/수정 UI
- [ ] `player_stats`에 source/captured_at/sample_size 컬럼 추가

## 12. 리스크
| 리스크 | 대응 |
|---|---|
| 스크린샷 오인식 | 사람 검증 단계, 핸드 히스토리 우선 |
| 규칙이 부족해 정답이 없는 상황 | "자료 없음" 표시 + 보완 큐 |
| 결과 지향 평가 | 승패가 아닌 일치율/EV로 집계 |
| 닉네임 변경으로 표본 부족 | 아키타입 분류 + 신뢰도 표시 |
| GTO Wizard 약관 | 수동 캡처, 개인용 한정, 재배포 금지 |
