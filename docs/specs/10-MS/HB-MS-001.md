---
doc_id: HB-MS-001
type: MS
title: MINISPEC — ImportService
status: draft
upstream: [HB-DOM-002, HB-SEQ-001, HB-API-001]
---

# MINISPEC

## 0. 이 문서가 다루는 것

`hb/service/import_service.py` — 파일 하나를 받아 거래를 만들고 분류해 저장하는 `ImportService`. 클래스 명세 [[HB-DOM-002#ImportService]]의 함수 하나하나.

## 1. 함수 목록

| 함수 | 한 줄 | 근거 |
|---|---|---|
| ImportService.import_file | 파일 바이트 → ImportResult | [[HB-SEQ-001#SEQ-1]] |
| ImportService.recent | 최근 가져오기 10건 | [[HB-API-001#GET/api/imports]] |

## 2. 함수

#### ImportService.import_file 파일 가져오기

**시그니처** `import_file(data: bytes, filename: str) -> ImportResult`

**입력** CSV 파일 바이트와 파일명(로그용). 5MB 초과는 라우터가 먼저 413으로 막는다.

**처리**
1. `CsvReader.detect_profile(data)`로 프로파일을 고른다. None이면 `UnknownProfileError(header)`.
2. `CsvReader.read_rows(data, profile)`로 `(rows, unread_rows)`를 얻는다.
3. 행마다 `Transaction.from_row(row, profile.code)`로 거래를 만든다 — merchant 정규화·fingerprint 계산 포함.
4. `repo.existing_fingerprints(fps)`로 이미 있는 지문을 빼 `new_txs`를 만든다.
5. `ClassifyService.classify(merchant)`를 new_txs마다 불러 category를 붙인다. 규칙표는 서비스가 한 번만 읽는다.
6. `repo.insert_batch(ImportBatch(profile, now, len(new_txs), len(rows)-len(new_txs), unread_rows), new_txs)` — 한 트랜잭션.
7. `ImportResult`를 돌려준다. `unclassified_count`는 new_txs 중 category가 uncategorized인 수.

**출력** `ImportResult(id, profile, imported_at, new_count, skipped_count, unread_rows, unclassified_count)`

**예외** | 조건 | 에러 |
|---|---|
| 프로파일 없음 | `UnknownProfileError(header: list[str])` → 400 unknown_profile |
| 저장 실패 | `sqlite3.Error` 그대로 → 500. 트랜잭션 롤백으로 아무것도 남지 않음 |

**호출하는 것** [[HB-DOM-002#CsvReader]] · [[HB-DOM-002#ClassifyService]] · [[HB-DOM-002#Repository]] · [[HB-DOM-001#Transaction]]

**테스트 관점**
- 은행 A 샘플 → new_count = 행 수, unread 0
- 같은 바이트 두 번 → 둘째는 new 0 · skipped = 행 수
- 알 수 없는 헤더 → UnknownProfileError, DB 행 수 불변
- 날짜가 깨진 행 하나 → unread_rows에 그 번호, 나머지는 저장
- 규칙 「스타벅스→food」가 있으면 스타벅스 행은 food, 없으면 uncategorized

근거: [[HB-SEQ-001#SEQ-1]] · [[HB-API-001#POST/api/imports]] · [[HB-PRD-001#R1]] · [[HB-PRD-001#R2]]

#### ImportService.recent 최근 가져오기

**시그니처** `recent(limit: int = 10) -> list[ImportResult]`

**입력** 개수.

**처리**
1. `repo.list_batches(limit)`를 imported_at 내림차순으로.
2. 각 batch를 ImportResult로 바꾼다(unclassified_count는 batch 저장 시 함께 기록한 값).

**출력** ImportResult 목록.

**예외** | 조건 | 에러 |
|---|---|
| 없음 | — |

**호출하는 것** [[HB-DOM-002#Repository]]

**테스트 관점** 배치 12개 넣고 10개만 최신순으로 오는지.

근거: [[HB-API-001#GET/api/imports]] · [[HB-UI-001#UI-1]]

## 3. 미결사항

- [ ] unclassified_count를 batch 행에 저장할지 매번 셀지(저장 쪽으로 가정)
