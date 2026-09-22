---
doc_id: HB-CODE-001
type: CODE
title: 가계부 자동 분류 — 구현 계획
status: approved
upstream: [HB-MS-001, HB-MS-002, HB-MS-003, HB-MS-004, HB-MS-005, HB-SEQ-001, HB-DOM-003]
---

# 구현 계획

## 0. 이 문서가 다루는 것

첫 버전을 슬라이스 다섯으로 나눈다. A는 기반, B1~B4는 시나리오 하나씩. 슬라이스 하나가 PR 하나. 완료 행에 커밋을 적는다.

## 1. 슬라이스

#### A 기반

| 항목 | 내용 |
|---|---|
| 근거 | [[HB-INFRA-001#C2]] · [[HB-INFRA-001#C3]] · [[HB-DOM-003#transactions]] · [[HB-DOM-003#rules]] |
| 구현 | pyproject · `hb serve` 진입 · SQLite 스키마와 마이그레이션 · 저장소 · 정적 파일 서빙 · 항목 상수 |
| 테스트 | 빈 DB로 기동 → `/api/categories`가 11개 · 스키마 버전 1 |
| 선행 | 없음 |
| 완료 | — |

#### B1 첫 달 — 파일 넣고 미분류 정리

| 항목 | 내용 |
|---|---|
| 근거 | [[HB-SCN-001#S1]] |
| 구현 함수 | [[HB-MS-001#ImportService.import_file]] · [[HB-MS-005#CsvReader.detect_profile]] · [[HB-MS-005#CsvReader.read_rows]] · [[HB-MS-002#ClassifyService.classify]] · [[HB-MS-002#ClassifyService.unclassified_groups]] · [[HB-MS-003#RuleService.set_rule]] |
| API | [[HB-API-001#POST/api/imports]] · [[HB-API-001#GET/api/unclassified]] · [[HB-API-001#PUT/api/rules/{merchant}]] |
| 화면 | [[HB-UI-001#UI-1]] · [[HB-UI-001#UI-2]] |
| 테스트 | 구현 함수의 테스트 관점 전부 + S1 E2E(샘플 CSV 셋 → 묶음 정리 → 미분류 0) |
| 선행 | A |
| 완료 | — |

#### B2 둘째 달 — 중복 배제와 자동 분류

| 항목 | 내용 |
|---|---|
| 근거 | [[HB-SCN-001#S2]] |
| 구현 함수 | [[HB-MS-001#ImportService.import_file]](지문 건너뛰기) · [[HB-MS-004#SummaryService.month]] · [[HB-MS-004#SummaryService.months]] |
| API | [[HB-API-001#GET/api/months/{ym}]] · [[HB-API-001#GET/api/months]] · [[HB-API-001#GET/api/imports]] |
| 화면 | [[HB-UI-001#UI-3]] |
| 테스트 | 같은 파일 두 번 → 새로 0 · 규칙 있는 가맹점 100% 분류 · 전월 증감 계산 · S2 E2E |
| 선행 | B1 |
| 완료 | — |

#### B3 예외 한 건 고치기

| 항목 | 내용 |
|---|---|
| 근거 | [[HB-SCN-001#S3]] |
| 구현 함수 | [[HB-MS-002#ClassifyService.confirm_one]] · [[HB-MS-003#RuleService.set_rule]](확정 건 제외 확인) |
| API | [[HB-API-001#PATCH/api/transactions/{id}]] |
| 화면 | [[HB-UI-001#UI-4]] |
| 테스트 | 이 건만 → 규칙 불변·확정 표시 · 전부 → 규칙 변경·확정 건 불변 · S3 E2E |
| 선행 | B2 |
| 완료 | — |

#### B4 취소 거래

| 항목 | 내용 |
|---|---|
| 근거 | [[HB-SCN-001#S4]] |
| 구현 함수 | [[HB-MS-005#CsvReader.read_rows]](취소 행 부호 뒤집기) |
| API | [[HB-API-001#POST/api/imports]] |
| 화면 | [[HB-UI-001#UI-3]] |
| 테스트 | 취소 행이 음수로 들어오고 원 거래와 같은 항목 · 월 합계 상쇄 · S4 E2E |
| 선행 | B2 |
| 완료 | — |

## 2. 통합 테스트

| 시나리오 | 슬라이스 | 검증하는 것 |
|---|---|---|
| [[HB-SCN-001#S1]] | B1 | 세 파일 → 275건 · 묶음 정리 → 규칙 생성 · 미분류 0 |
| [[HB-SCN-001#S2]] | B2 | 규칙 있는 가맹점 자동 분류 · 같은 파일 재투입 시 새로 0 · 전월 증감 |
| [[HB-SCN-001#S3]] | B3 | 이 건만 / 전부의 차이 · 확정 건 보호 |
| [[HB-SCN-001#S4]] | B4 | 취소 상쇄 |
| 네트워크 차단 | 전부 | 소켓 차단 상태에서 전 시나리오 통과([[HB-PRD-001#R6]]) |

## 3. 커밋·PR 목록

슬라이스 카드의 `완료` 행에 기록.

## 4. 미결사항

- [ ] 카드 2곳 실제 샘플 CSV를 tests/fixtures에 넣기 전까지 B1은 은행 A 샘플로만 검증한다
