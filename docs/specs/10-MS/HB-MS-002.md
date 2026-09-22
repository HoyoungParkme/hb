---
doc_id: HB-MS-002
type: MS
title: MINISPEC — ClassifyService
status: approved
upstream: [HB-DOM-002, HB-SEQ-001, HB-API-001]
---

# MINISPEC

## 0. 이 문서가 다루는 것

`hb/service/classify_service.py` — 거래에 항목을 붙이는 `ClassifyService`. 클래스 명세 [[HB-DOM-002#ClassifyService]].

## 1. 함수 목록

| 함수 | 한 줄 | 근거 |
|---|---|---|
| ClassifyService.normalize | 적요 원문 → merchant | [[HB-UC-001#UC-S1]] |
| ClassifyService.classify | merchant → category | [[HB-UC-001#UC-S1]] |
| ClassifyService.unclassified_groups | 미분류를 merchant로 묶는다 | [[HB-API-001#GET/api/unclassified]] |
| ClassifyService.confirm_one | 한 건만 바꾸고 확정 | [[HB-SEQ-001#SEQ-3]] |

## 2. 함수

#### ClassifyService.normalize 적요 정규화

**시그니처** `normalize(raw_memo: str) -> str`

**입력** CSV 적요 원문.

**처리**
1. 앞뒤 공백 제거, 연속 공백 하나로, 소문자화.
2. 괄호와 그 안 내용 제거 — `(주)`, `(강남점)`.
3. 꼬리의 숫자·하이픈 덩어리 제거 — 승인번호·전화번호.
4. 「점」으로 끝나는 마지막 토큰 제거 — `강남역점`.
5. 결과가 빈 문자열이면 1의 결과를 그대로 쓴다.

**출력** merchant 문자열.

**예외** | 조건 | 에러 |
|---|---|
| 없음 | — |

**호출하는 것** 없음(순수 함수)

**테스트 관점** `스타벅스 강남역점` · `스타벅스(주)삼성점` · `STARBUCKS 1234-5678` → 전부 `스타벅스`/`starbucks`로 수렴하는지 · 빈 문자열 → 원문

근거: [[HB-UC-001#UC-S1]] · [[HB-DOM-001#Transaction]]

#### ClassifyService.classify 규칙으로 분류

**시그니처** `classify(merchant: str, rules: RuleIndex | None = None) -> str`

**입력** merchant와 선택적 규칙 인덱스(가져오기가 300건을 돌 때 한 번만 만들어 넘긴다).

**처리**
1. rules가 None이면 `RuleService.index()`로 만든다.
2. `rules.exact.get(merchant)`가 있으면 그 category. 끝.
3. `rules.contains` 중 `r.merchant in merchant`인 것을 merchant 길이 내림차순으로 보고 첫 것의 category. 끝.
4. 없으면 `uncategorized`.

**출력** category 코드.

**예외** | 조건 | 에러 |
|---|---|
| 없음 | — |

**호출하는 것** [[HB-DOM-002#RuleService]]

**테스트 관점** exact가 contains보다 이김 · contains 둘이면 긴 것 · 규칙 없으면 uncategorized · 빈 규칙표에서도 동작

근거: [[HB-UC-001#UC-S1]] · [[HB-PRD-001#R3]]

#### ClassifyService.unclassified_groups 미분류 묶음

**시그니처** `unclassified_groups() -> tuple[int, list[MerchantGroup]]`

**입력** 없음.

**처리**
1. `repo.list_unclassified()`로 미분류 거래 전부.
2. merchant로 묶어 count·total(amount 합)·samples(raw_memo 최대 3개, 중복 제거)·ids.
3. `abs(total)` 내림차순 정렬.

**출력** (전체 건수, 묶음 목록)

**예외** | 조건 | 에러 |
|---|---|
| 없음 | — |

**호출하는 것** [[HB-DOM-002#Repository]]

**테스트 관점** 12건 3가맹점 → 3묶음, 합계 큰 순 · 미분류 0건 → (0, [])

근거: [[HB-API-001#GET/api/unclassified]] · [[HB-UC-001#UC-H2]]

#### ClassifyService.confirm_one 한 건 확정

**시그니처** `confirm_one(tx_id: int, category: str) -> Transaction`

**입력** 거래 id, 항목 코드.

**처리**
1. category가 항목표에 없거나 uncategorized면 `BadCategoryError`.
2. `repo.get_transaction(tx_id)` 없으면 `NotFoundError`.
3. `repo.update_category(tx_id, category, confirmed=True)`.
4. 갱신된 거래를 돌려준다.

**출력** Transaction

**예외** | 조건 | 에러 |
|---|---|
| 항목 없음 | BadCategoryError → 400 |
| 거래 없음 | NotFoundError → 404 |

**호출하는 것** [[HB-DOM-002#Repository]]

**테스트 관점** 확정 후 같은 merchant 규칙을 바꿔도 이 건은 불변 · 없는 id → 404 · uncategorized로 되돌리기 거부

근거: [[HB-SEQ-001#SEQ-3]] · [[HB-API-001#PATCH/api/transactions/{id}]] · [[HB-PRD-001#R4]]

## 3. 미결사항

- [ ] 정규화 4단계(「점」 제거)가 「편의점」 같은 단어를 깎지 않도록 예외 목록이 필요한지
