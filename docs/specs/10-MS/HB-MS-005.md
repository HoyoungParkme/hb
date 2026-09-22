---
doc_id: HB-MS-005
type: MS
title: MINISPEC — CsvReader
status: draft
upstream: [HB-DOM-002, HB-SEQ-001]
---

# MINISPEC

## 0. 이 문서가 다루는 것

`hb/infra/csv_reader.py` — 바이트를 프로파일에 따라 행으로 바꾸는 `CsvReader`. 클래스 명세 [[HB-DOM-002#CsvReader]]. 프로파일 정의 자체는 `infra/profiles.py` 상수.

## 1. 함수 목록

| 함수 | 한 줄 | 근거 |
|---|---|---|
| CsvReader.detect_profile | 헤더로 프로파일 판별 | [[HB-SEQ-001#SEQ-1]] |
| CsvReader.read_rows | 행을 읽어 정규 형태로 | [[HB-SEQ-001#SEQ-1]] |

## 2. 함수

#### CsvReader.detect_profile 프로파일 판별

**시그니처** `detect_profile(data: bytes) -> ImportProfile | None`

**입력** 파일 바이트.

**처리**
1. 디코딩 — BOM이 있으면 그 인코딩, 없으면 utf-8 시도, 실패하면 cp949([[HB-INFRA-001#C5]]).
2. 첫 줄을 헤더로 쪼갠다(쉼표, 따옴표 처리는 표준 csv).
3. PROFILES를 순서대로 보며 `header_signature ⊆ set(header)`인 첫 것을 돌려준다.
4. 없으면 None. 호출자가 헤더를 오류에 실을 수 있게 `self.last_header`에 남긴다.

**출력** ImportProfile 또는 None

**예외** | 조건 | 에러 |
|---|---|
| 두 인코딩 모두 실패 | `UnicodeDecodeError` 그대로 → 500 |

**호출하는 것** 없음

**테스트 관점** utf-8 BOM · cp949 · 은행 A/카드 B/카드 C 헤더 각각 · 낯선 헤더 → None

근거: [[HB-SEQ-001#SEQ-1]] · [[HB-PRD-001#R1]] · [[HB-DOM-001#ImportProfile]]

#### CsvReader.read_rows 행 읽기

**시그니처** `read_rows(data: bytes, profile: ImportProfile) -> tuple[list[RawRow], list[int]]`

**입력** 파일 바이트와 프로파일.

**처리**
1. detect_profile과 같은 방법으로 디코딩.
2. DictReader로 행마다 column_map에 따라 `occurred_at`(date_format으로 파싱), `amount`(쉼표·원 기호 제거 후 int; 출금/입금이 두 컬럼이면 출금은 음수), `raw_memo`.
3. profile에 취소 표시 컬럼이 있고 그 값이 취소이면 amount 부호를 뒤집는다([[HB-PRD-001#R7]]).
4. 날짜·금액 파싱에 실패한 행은 `unread_rows`에 행 번호(1부터, 헤더 제외)를 넣고 건너뛴다.

**출력** (RawRow 목록, 읽지 못한 행 번호 목록)

**예외** | 조건 | 에러 |
|---|---|
| 없음 — 행 단위 실패는 unread_rows | — |

**호출하는 것** 없음

**테스트 관점** 세 형식 각각 정상 · 날짜 세 표기 · 금액 「1,234원」 · 취소 행 부호 · 깨진 행 번호

근거: [[HB-SEQ-001#SEQ-1]] · [[HB-UC-001#UC-H1]] · [[HB-PRD-001#R1]] · [[HB-PRD-001#R7]]

## 3. 미결사항

- [ ] 카드 C가 취소를 별도 컬럼이 아니라 적요 앞 「취소」 글자로 표시하는지 확인 필요
