---
doc_id: HB-API-001
type: API
title: 가계부 자동 분류 — REST API
status: approved
upstream: [HB-UC-001, HB-DOM-001, HB-UI-001]
---

# API 명세 REST

## 0. 이 문서가 다루는 것

브라우저 화면(HB-UI-001)이 부르는 REST 엔드포인트 전부. MCP 도구는 없다 — 에이전트가 부를 일이 없는 개인 도구다. 서버는 `127.0.0.1:8765`, 인증 없음([[HB-INFRA-001#C3]]).

## 1. 규칙

- 경로는 `/api/` 아래. JSON 요청·응답. 금액은 원 단위 정수, 날짜는 `YYYY-MM-DD`, 월은 `YYYY-MM`.
- 항목은 코드(`food` 등)로 주고받는다([[HB-DOM-001#Category]]).
- 실패는 `{ "error": "<코드>", "message": "<사람이 읽는 문장>", "detail": {...} }`.
- 목록에 페이지는 없다. 한 달 300건, 규칙 수백 개 규모다.

## 2. 에러

| HTTP | error | 언제 |
|---|---|---|
| 400 | `unknown_profile` | 헤더가 어느 프로파일과도 맞지 않음. detail.header에 헤더 행 |
| 400 | `bad_category` | 없는 항목 코드 |
| 400 | `bad_month` | 월 형식 오류 |
| 404 | `not_found` | 거래 id 없음 |
| 413 | `too_large` | 파일 5MB 초과 |
| 500 | `internal` | 그 외 |

## 3. 엔드포인트 (또는 도구)

#### POST/api/imports 파일 가져오기

화면 [[HB-UI-001#UI-1]] · 유스케이스 [[HB-UC-001#UC-H1]] · 서비스 `ImportService.import_file`

```yaml
/api/imports:
  post:
    summary: CSV 파일 하나를 가져온다
    requestBody:
      content:
        multipart/form-data:
          schema:
            type: object
            properties:
              file: { type: string, format: binary }
    responses:
      "200":
        description: 가져오기 결과
        content:
          application/json:
            schema: { $ref: "#/components/schemas/ImportResult" }
      "400":
        description: unknown_profile
```

#### GET/api/imports 최근 가져오기 목록

화면 [[HB-UI-001#UI-1]] · 유스케이스 [[HB-UC-001#UC-H1]] · 서비스 `ImportService.recent`

```yaml
/api/imports:
  get:
    summary: 최근 가져오기 10건
    responses:
      "200":
        content:
          application/json:
            schema:
              type: array
              items: { $ref: "#/components/schemas/ImportResult" }
```

#### GET/api/unclassified 미분류 묶음

화면 [[HB-UI-001#UI-2]] · 유스케이스 [[HB-UC-001#UC-H2]] · 서비스 `ClassifyService.unclassified_groups`

```yaml
/api/unclassified:
  get:
    summary: 미분류 거래를 merchant로 묶어 합계 절댓값 내림차순으로
    responses:
      "200":
        content:
          application/json:
            schema:
              type: object
              properties:
                total: { type: integer, description: 미분류 거래 수 }
                groups:
                  type: array
                  items: { $ref: "#/components/schemas/MerchantGroup" }
```

#### PUT/api/rules/{merchant} 가맹점 규칙 정하기

화면 [[HB-UI-001#UI-2]] · [[HB-UI-001#UI-4]] · 유스케이스 [[HB-UC-001#UC-H2]] · [[HB-UC-001#UC-H3]] · 서비스 `RuleService.set_rule`

```yaml
/api/rules/{merchant}:
  put:
    summary: merchant → category 규칙을 만들거나 갱신하고, 미확정 거래를 다시 분류한다
    parameters:
      - { name: merchant, in: path, required: true, schema: { type: string } }
    requestBody:
      content:
        application/json:
          schema:
            type: object
            required: [category]
            properties:
              category: { type: string, example: food }
    responses:
      "200":
        content:
          application/json:
            schema:
              type: object
              properties:
                rule: { $ref: "#/components/schemas/Rule" }
                reclassified: { type: integer, description: 바뀐 거래 수 }
      "400":
        description: bad_category
```

#### PATCH/api/transactions/{id} 거래 한 건 항목 바꾸기

화면 [[HB-UI-001#UI-4]] · 유스케이스 [[HB-UC-001#UC-H3]] · 서비스 `ClassifyService.confirm_one`

```yaml
/api/transactions/{id}:
  patch:
    summary: 이 건만 항목을 바꾸고 확정한다. 규칙은 건드리지 않는다
    parameters:
      - { name: id, in: path, required: true, schema: { type: integer } }
    requestBody:
      content:
        application/json:
          schema:
            type: object
            required: [category]
            properties:
              category: { type: string }
    responses:
      "200":
        content:
          application/json:
            schema: { $ref: "#/components/schemas/Transaction" }
      "404":
        description: not_found
```

#### GET/api/months/{ym} 월 요약

화면 [[HB-UI-001#UI-3]] · 유스케이스 [[HB-UC-001#UC-H4]] · 서비스 `SummaryService.month`

```yaml
/api/months/{ym}:
  get:
    summary: 한 달의 항목별 합계·전월 대비·거래 목록·미분류 건수
    parameters:
      - { name: ym, in: path, required: true, schema: { type: string, example: "2026-10" } }
      - { name: category, in: query, required: false, schema: { type: string }, description: 거래 목록을 이 항목으로 거른다 }
    responses:
      "200":
        content:
          application/json:
            schema: { $ref: "#/components/schemas/MonthlySummary" }
      "400":
        description: bad_month
```

#### GET/api/months 거래가 있는 달 목록

화면 [[HB-UI-001#UI-3]] · 유스케이스 [[HB-UC-001#UC-H4]] · 서비스 `SummaryService.months`

```yaml
/api/months:
  get:
    summary: 거래가 1건 이상 있는 달을 최신순으로
    responses:
      "200":
        content:
          application/json:
            schema:
              type: array
              items: { type: string, example: "2026-10" }
```

#### GET/api/categories 항목 목록

화면 [[HB-UI-001#UI-2]] · [[HB-UI-001#UI-4]] · 유스케이스 [[HB-UC-001#UC-H2]] · 서비스 `RuleService.categories`

```yaml
/api/categories:
  get:
    summary: 항목 11개(코드·이름·kind)
    responses:
      "200":
        content:
          application/json:
            schema:
              type: array
              items: { $ref: "#/components/schemas/Category" }
```

## 4. 스키마 (또는 에이전트 순서)

```yaml
components:
  schemas:
    ImportResult:
      type: object
      properties:
        id: { type: integer }
        profile: { type: string, example: card_b }
        imported_at: { type: string, format: date-time }
        new_count: { type: integer }
        skipped_count: { type: integer }
        unread_rows: { type: array, items: { type: integer } }
        unclassified_count: { type: integer, description: 이번에 들어온 것 중 미분류 }
    Transaction:
      type: object
      properties:
        id: { type: integer }
        source: { type: string }
        occurred_at: { type: string, format: date }
        amount: { type: integer }
        raw_memo: { type: string }
        merchant: { type: string }
        category: { type: string }
        confirmed: { type: boolean }
    MerchantGroup:
      type: object
      properties:
        merchant: { type: string }
        count: { type: integer }
        total: { type: integer }
        samples: { type: array, items: { type: string }, description: raw_memo 최대 3개 }
        ids: { type: array, items: { type: integer } }
    Rule:
      type: object
      properties:
        merchant: { type: string }
        match: { type: string, enum: [exact, contains] }
        category: { type: string }
        updated_at: { type: string, format: date-time }
    Category:
      type: object
      properties:
        code: { type: string }
        name: { type: string }
        kind: { type: string, enum: [expense, income, transfer, uncategorized] }
    MonthlySummary:
      type: object
      properties:
        month: { type: string }
        totals: { type: object, properties: { expense: { type: integer }, income: { type: integer }, transfer: { type: integer } } }
        by_category:
          type: array
          items:
            type: object
            properties:
              category: { type: string }
              total: { type: integer }
              prev_diff: { type: integer, nullable: true }
        unclassified_count: { type: integer }
        transactions: { type: array, items: { $ref: "#/components/schemas/Transaction" } }
```

## 5. 미결사항

- [ ] 가져오기를 되돌리는 `DELETE /api/imports/{id}`는 첫 버전에서 뺀다
- [ ] 규칙 목록·삭제 화면이 없으므로 `GET/DELETE /api/rules`도 뺀다 — 필요해지면 UI와 같이 추가
