---
doc_id: HB-MS-004
type: MS
title: MINISPEC — SummaryService
status: draft
upstream: [HB-DOM-002, HB-SEQ-001, HB-API-001]
---

# MINISPEC

## 0. 이 문서가 다루는 것

`hb/service/summary_service.py` — 월 요약을 계산하는 `SummaryService`. 클래스 명세 [[HB-DOM-002#SummaryService]]. 저장하지 않는다.

## 1. 함수 목록

| 함수 | 한 줄 | 근거 |
|---|---|---|
| SummaryService.month | 한 달 요약 | [[HB-SEQ-001#SEQ-4]] |
| SummaryService.months | 거래 있는 달 목록 | [[HB-API-001#GET/api/months]] |

## 2. 함수

#### SummaryService.month 월 요약

**시그니처** `month(ym: str, category: str | None = None) -> MonthlySummary`

**입력** `YYYY-MM`, 선택적 항목 코드(거래 목록 필터).

**처리**
1. ym 형식 검증 — 아니면 `BadMonthError`.
2. `cur = repo.sum_by_category(ym)` — {category: total}.
3. `prev = repo.sum_by_category(prev_month(ym))`. 비어 있으면 None.
4. totals: expense = kind가 expense인 항목 합, income, transfer 각각. uncategorized는 어디에도 넣지 않는다.
5. by_category: cur의 항목마다 `(category, total, prev_diff)`. prev가 None이면 prev_diff는 None, 아니면 `cur - prev.get(category, 0)`. 절댓값 내림차순.
6. `transactions = repo.list_transactions(ym, category)` 날짜 내림차순.
7. `unclassified_count = repo.count_unclassified(ym)`.

**출력** MonthlySummary

**예외** | 조건 | 에러 |
|---|---|
| 형식 오류 | BadMonthError → 400 |

**호출하는 것** [[HB-DOM-002#Repository]]

**테스트 관점** by_category 합 = 거래 합 · 전월 없음 → prev_diff None · 취소 건이 합계를 상쇄 · 이체가 expense에 안 들어감 · category 필터

근거: [[HB-SEQ-001#SEQ-4]] · [[HB-API-001#GET/api/months/{ym}]] · [[HB-PRD-001#R5]] · [[HB-PRD-001#R7]]

#### SummaryService.months 달 목록

**시그니처** `months() -> list[str]`

**입력** 없음.

**처리**
1. `repo.distinct_months()` 최신순.

**출력** `["2026-10", "2026-09", …]`

**예외** | 조건 | 에러 |
|---|---|
| 없음 | — |

**호출하는 것** [[HB-DOM-002#Repository]]

**테스트 관점** 거래 없으면 빈 목록 · 순서

근거: [[HB-API-001#GET/api/months]] · [[HB-UI-001#UI-3]]

## 3. 미결사항

- [ ] prev_diff를 「전월 없음」과 「전월에 그 항목 0」으로 구분해 보여줄지(지금은 후자를 0으로)
