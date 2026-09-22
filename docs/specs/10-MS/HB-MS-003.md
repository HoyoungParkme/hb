---
doc_id: HB-MS-003
type: MS
title: MINISPEC — RuleService
status: approved
upstream: [HB-DOM-002, HB-SEQ-001, HB-API-001]
---

# MINISPEC

## 0. 이 문서가 다루는 것

`hb/service/rule_service.py` — 분류 규칙과 항목표를 다루는 `RuleService`. 클래스 명세 [[HB-DOM-002#RuleService]].

## 1. 함수 목록

| 함수 | 한 줄 | 근거 |
|---|---|---|
| RuleService.set_rule | 규칙 만들거나 갱신 + 미확정 재분류 | [[HB-SEQ-001#SEQ-2]] |
| RuleService.index | 규칙표를 메모리 인덱스로 | [[HB-MS-002#ClassifyService.classify]] |
| RuleService.categories | 항목 11개 | [[HB-API-001#GET/api/categories]] |

## 2. 함수

#### RuleService.set_rule 규칙 정하기

**시그니처** `set_rule(merchant: str, category: str) -> tuple[Rule, int]`

**입력** 정규화된 merchant, 항목 코드.

**처리**
1. category 검증 — 항목표에 없거나 uncategorized면 `BadCategoryError`.
2. 트랜잭션 시작.
3. `repo.upsert_rule(merchant, "exact", category, now)`.
4. `n = repo.reclassify_unconfirmed(merchant, category)` — `merchant = ? AND confirmed = 0`인 거래의 category를 바꾼다.
5. 커밋. `(rule, n)`.

**출력** 규칙과 바뀐 거래 수.

**예외** | 조건 | 에러 |
|---|---|
| 항목 없음 | BadCategoryError → 400 |

**호출하는 것** [[HB-DOM-002#Repository]]

**테스트 관점** 새 merchant → 규칙 생성, 미분류 12건 전부 변경 · 기존 규칙 → 갱신, 미확정만 변경, 확정 건 불변 · 4에서 실패 시 3도 롤백

근거: [[HB-SEQ-001#SEQ-2]] · [[HB-API-001#PUT/api/rules/{merchant}]] · [[HB-PRD-001#R4]]

#### RuleService.index 규칙 인덱스

**시그니처** `index() -> RuleIndex`

**입력** 없음.

**처리**
1. `repo.list_rules()` 전부.
2. `exact: dict[str, str]`, `contains: list[Rule]`(merchant 길이 내림차순)로 나눈다.

**출력** RuleIndex(exact, contains)

**예외** | 조건 | 에러 |
|---|---|
| 없음 | — |

**호출하는 것** [[HB-DOM-002#Repository]]

**테스트 관점** 규칙 0개 → 빈 인덱스 · contains 정렬 순서

근거: [[HB-INFRA-001#C4]] · [[HB-UC-001#UC-S1]]

#### RuleService.categories 항목 목록

**시그니처** `categories() -> list[Category]`

**입력** 없음.

**처리**
1. 코드 상수 `CATEGORIES`를 그대로 돌려준다.

**출력** Category 11개.

**예외** | 조건 | 에러 |
|---|---|
| 없음 | — |

**호출하는 것** 없음

**테스트 관점** 11개, 코드 유일, uncategorized 포함

근거: [[HB-API-001#GET/api/categories]] · [[HB-DOM-001#Category]]

## 3. 미결사항

- [ ] 기본 제공 contains 규칙(택시→교통 등)을 코드에 둘지 첫 기동 때 rules 테이블에 심을지
