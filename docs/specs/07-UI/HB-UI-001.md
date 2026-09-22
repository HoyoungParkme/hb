---
doc_id: HB-UI-001
type: UI
title: 가계부 자동 분류 — 화면 설계·와이어프레임
status: approved
upstream: [HB-UC-001, HB-DOM-001]
---

# 화면 설계·와이어프레임 — 가계부 자동 분류

## 0. 이 문서가 다루는 것

브라우저에서 보이는 화면 넷의 목록·흐름·배치·요소·규칙. 한 문서에 화면 설계와 와이어프레임을 같이 쓴다. 화면은 서버가 주는 정적 HTML 하나에 탭처럼 붙는다([[HB-INFRA-001#C3]]). 디자인(색·폰트)은 다루지 않는다.

## 1. 유스케이스 대응

| 유스케이스 | 화면 |
|---|---|
| [[HB-UC-001#UC-H1]] | UI-1 |
| [[HB-UC-001#UC-H2]] | UI-2 |
| [[HB-UC-001#UC-H3]] | UI-3 · UI-4 |
| [[HB-UC-001#UC-H4]] | UI-3 |

## 2. 화면 목록

| 화면 | 경로 | 한 줄 목적 |
|---|---|---|
| UI-1 가져오기 | `/` | CSV를 놓고 결과 건수를 본다 |
| UI-2 미분류 정리 | `/#unclassified` | 가맹점 묶음마다 항목을 고른다 |
| UI-3 월 요약 | `/#month/{ym}` | 항목별 합계·증감·목록을 본다 |
| UI-4 거래 항목 바꾸기 | (UI-3 위의 대화상자) | 한 건을 고치거나 규칙째 바꾼다 |

## UI-1 가져오기

| 항목 | 내용 |
|---|---|
| 경로 | `/` |
| 주 유스케이스 | [[HB-UC-001#UC-H1]] |
| 진입 / 이탈 | 도구를 켜면 첫 화면 / 결과의 「미분류 K건」을 누르면 UI-2, 상단 탭으로 UI-3 |

### 배치
```html
<div data-el="1" class="topbar">
  <span data-el="2">가계부 자동 분류</span>
  <nav data-el="3">[가져오기] [미분류 (K)] [월 요약]</nav>
</div>
<main>
  <div data-el="4" class="dropzone" style="border:2px dashed #999;padding:48px;text-align:center">
    CSV 파일을 여기에 놓으세요<br/>
    <button data-el="5">파일 고르기</button>
  </div>
  <section data-el="6" class="result" style="margin-top:16px">
    <div data-el="7">카드 B 형식 · 새로 들어온 150건 · 건너뛴 0건 · 읽지 못한 행 0건</div>
    <div data-el="8"><a href="#unclassified">미분류 41건 정리하기 →</a></div>
  </section>
  <section data-el="9" class="error" style="display:none;border:1px solid #c00;padding:12px">
    어느 기관 파일인지 모르겠습니다. 헤더 행:
    <pre data-el="10">거래일자,적요,출금액,입금액,잔액</pre>
  </section>
  <section data-el="11" class="history">
    <h3>최근 가져오기</h3>
    <table data-el="12">
      <tr><th>때</th><th>형식</th><th>새로</th><th>건너뜀</th></tr>
      <tr><td>10-02 21:10</td><td>카드 B</td><td>150</td><td>0</td></tr>
    </table>
  </section>
</main>
```

### 요소
| # | 이름 | 종류 | 보여주는 것 | 누르면 |
|---|---|---|---|---|
| 1 | 상단바 | 영역 | 모든 화면 공통 | — |
| 3 | 탭 | 내비 | 가져오기 · 미분류(건수) · 월 요약 | 해당 화면으로 |
| 4 | 놓는 곳 | 드롭존 | 안내 문구 | 파일을 놓으면 즉시 가져오기 |
| 5 | 파일 고르기 | 버튼 | — | 파일 선택창 |
| 6 | 결과 | 영역 | 형식·새로·건너뜀·읽지 못한 행 | — |
| 8 | 미분류 링크 | 링크 | 이번 가져오기의 미분류 건수 | UI-2 |
| 9 | 판별 실패 | 경고 | 메시지와 헤더 행 | — |
| 12 | 최근 가져오기 | 표 | 최근 10건의 ImportBatch | — |

### 규칙
- 파일을 놓으면 확인 없이 바로 가져온다. 중복은 어차피 건너뛴다([[HB-PRD-001#R2]]).
- 9(판별 실패)가 보일 때 6은 숨긴다. 둘이 같이 보이지 않는다.
- 읽지 못한 행이 있으면 7의 그 숫자를 강조한다.

### 시나리오
**S-1 첫 파일 넣기** — [[HB-UC-001#UC-H1]]
1. 4에 파일을 놓는다.
2. 6에 결과가 뜬다. 8의 건수가 0이 아니면 링크가 활성.
3. 8을 눌러 UI-2로.

## UI-2 미분류 정리

| 항목 | 내용 |
|---|---|
| 경로 | `/#unclassified` |
| 주 유스케이스 | [[HB-UC-001#UC-H2]] |
| 진입 / 이탈 | 탭 또는 UI-1의 8 / 묶음이 0이 되면 「월 요약 보기」 링크 |

### 배치
```html
<div data-el="1" class="topbar">…공통…</div>
<main>
  <h2 data-el="2">미분류 41건 · 가맹점 17개</h2>
  <table data-el="3">
    <tr><th>가맹점</th><th>건수</th><th>합계</th><th>항목</th></tr>
    <tr data-el="4">
      <td data-el="5">스타벅스</td>
      <td data-el="6">12</td>
      <td data-el="7">-58,400</td>
      <td>
        <select data-el="8">
          <option>고르기…</option><option>식비</option><option>교통</option><option>…</option>
        </select>
      </td>
    </tr>
    <tr><td colspan="4" data-el="9" style="font-size:12px;color:#666">
      원문: 스타벅스 강남역점 · 스타벅스(주)삼성점 · …  <a data-el="10">펼치기</a>
    </td></tr>
  </table>
  <div data-el="11" style="display:none">다 정리했습니다. <a href="#month/2026-10">10월 요약 보기 →</a></div>
</main>
```

### 요소
| # | 이름 | 종류 | 보여주는 것 | 누르면 |
|---|---|---|---|---|
| 2 | 제목 | 텍스트 | 남은 건수·묶음 수 | — |
| 3 | 묶음 표 | 표 | merchant로 묶은 미분류 거래, 합계 내림차순 | — |
| 8 | 항목 고르기 | 셀렉트 | 항목 10개(미분류 제외) | 고르는 즉시 저장, 행이 사라진다 |
| 9 | 원문 미리보기 | 텍스트 | 묶음 안 raw_memo 세 개 | 10을 누르면 전부 |
| 11 | 끝 안내 | 링크 | 묶음이 0일 때 | UI-3 |

### 규칙
- 8에서 고르면 저장 버튼 없이 즉시 반영. 되돌리기는 UI-3에서 그 거래를 고친다.
- 표는 합계 절댓값 큰 순. 큰 것부터 정리하게.
- 항목 목록에 「미분류」는 없다. 미분류로 되돌리는 것은 지원하지 않는다.

### 시나리오
**S-2 묶음 정리** — [[HB-UC-001#UC-H2]]
1. 4의 8에서 식비를 고른다.
2. 행이 사라지고 2의 숫자가 준다.
3. 0이 되면 11이 보인다.

## UI-3 월 요약

| 항목 | 내용 |
|---|---|
| 경로 | `/#month/{ym}` |
| 주 유스케이스 | [[HB-UC-001#UC-H4]] |
| 진입 / 이탈 | 탭(기본은 가장 최근 달) / 거래 행을 누르면 UI-4 |

### 배치
```html
<div data-el="1" class="topbar">…공통…</div>
<main>
  <div data-el="2">
    <button data-el="3">◀</button> <span data-el="4">2026년 10월</span> <button data-el="5">▶</button>
  </div>
  <div data-el="6" style="border:1px solid #c90;padding:8px">미분류 9건이 있습니다. <a href="#unclassified">정리하기</a></div>
  <div data-el="7">지출 1,842,300 · 수입 3,200,000 · 이체 500,000</div>
  <table data-el="8">
    <tr><th>항목</th><th>합계</th><th>전월 대비</th></tr>
    <tr data-el="9"><td>식비</td><td>-612,400</td><td>+48,000</td></tr>
    <tr data-el="10"><td>의료</td><td>-96,000</td><td>+48,000</td></tr>
    <tr><td>…</td><td>…</td><td>…</td></tr>
  </table>
  <table data-el="11" style="margin-top:12px">
    <tr><th>날짜</th><th>가맹점</th><th>금액</th><th>기관</th><th></th></tr>
    <tr data-el="12"><td>10-03</td><td>스타벅스</td><td>-4,800</td><td>카드 B</td><td><a data-el="13">고치기</a></td></tr>
  </table>
</main>
```

### 요소
| # | 이름 | 종류 | 보여주는 것 | 누르면 |
|---|---|---|---|---|
| 3·5 | 달 이동 | 버튼 | — | 전월·다음 달. 거래 없는 달은 회색 |
| 6 | 미분류 경고 | 경고 | 미분류 건수. 0이면 숨김 | UI-2 |
| 7 | 총계 | 텍스트 | 지출·수입·이체 합계 | — |
| 8 | 항목 표 | 표 | 항목별 합계·전월 대비 | 행을 누르면 11이 그 항목으로 걸러진다 |
| 11 | 거래 목록 | 표 | 고른 항목(없으면 전부)의 거래, 확정 건은 표시 | — |
| 13 | 고치기 | 링크 | — | UI-4 |

### 규칙
- 전월 데이터가 없으면 「전월 대비」 칸은 빈칸([[HB-PRD-001#R5]]).
- 지출 합계에 이체·수입·미분류는 넣지 않는다. 미분류는 6에서만 보인다.
- 취소 거래는 음수·양수 그대로 목록에 둘 다 보인다([[HB-PRD-001#R7]]).

### 시나리오
**S-3 증감 확인** — [[HB-UC-001#UC-H4]]
1. 4가 10월인 상태로 8을 본다.
2. 10(의료)을 누르면 11이 의료 거래만 보인다.

## UI-4 거래 항목 바꾸기

| 항목 | 내용 |
|---|---|
| 경로 | UI-3 위의 대화상자 |
| 주 유스케이스 | [[HB-UC-001#UC-H3]] |
| 진입 / 이탈 | UI-3의 13 / 적용 또는 취소로 UI-3 |

### 배치
```html
<div data-el="1" class="dialog" style="border:1px solid #333;padding:16px;width:360px">
  <div data-el="2">10-03 스타벅스 -4,800 (카드 B)</div>
  <div data-el="3">현재: 식비</div>
  <select data-el="4"><option>식비</option><option>교통</option><option>…</option></select>
  <div>
    <label><input type="radio" name="scope" data-el="5" checked/> 이 건만 (확정)</label><br/>
    <label><input type="radio" name="scope" data-el="6"/> 이 가맹점 전부 — 규칙도 바꾼다</label>
  </div>
  <div data-el="7" style="font-size:12px;color:#666">「전부」를 고르면 확정되지 않은 스타벅스 거래 11건이 함께 바뀝니다.</div>
  <button data-el="8">적용</button> <button data-el="9">취소</button>
</div>
```

### 요소
| # | 이름 | 종류 | 보여주는 것 | 누르면 |
|---|---|---|---|---|
| 2 | 거래 | 텍스트 | 날짜·가맹점·금액·기관 | — |
| 4 | 새 항목 | 셀렉트 | 항목 10개 | — |
| 5 | 이 건만 | 라디오 | 기본값 | — |
| 6 | 전부 | 라디오 | — | 7에 영향 건수 표시 |
| 8 | 적용 | 버튼 | — | 저장 후 닫고 UI-3 갱신 |

### 규칙
- 기본은 「이 건만」. 규칙을 바꾸는 쪽이 더 큰 일이므로 명시적으로 고르게 한다.
- 7의 건수는 확정되지 않은 같은 merchant 거래 수다. 확정 거래는 세지 않는다([[HB-PRD-001#R4]]).

### 시나리오
**S-4 예외 한 건** — [[HB-UC-001#UC-H3]]
1. 4에서 식비를 고르고 5를 둔 채 8.
2. UI-3의 그 행에 확정 표시가 붙는다.

## 3. 공통 틀

상단바(UI-1의 1~3)는 모든 화면 공통. 탭의 「미분류 (K)」 건수는 화면을 바꿀 때마다 새로 읽는다. 화면 전환은 해시 라우팅, 페이지 새로고침 없음. 오류는 화면 상단에 한 줄 띠로 보이고 5초 뒤 사라진다.

## 4. 화면 흐름

```mermaid
flowchart LR
    UI1[UI-1 가져오기] -->|미분류 K건| UI2[UI-2 미분류 정리]
    UI1 -->|탭| UI3[UI-3 월 요약]
    UI2 -->|다 정리함| UI3
    UI3 -->|고치기| UI4[UI-4 항목 바꾸기]
    UI4 -->|적용·취소| UI3
    UI3 -->|미분류 경고| UI2
```

## 5. 미결사항

- [ ] UI-2에서 묶음이 아니라 개별 거래 단위로 고르는 길이 필요한지(같은 merchant인데 다른 항목이어야 하는 경우는 UI-4로 우회)
- [ ] 달 이동(UI-3의 3·5)에서 거래 없는 달을 건너뛸지 회색으로 둘지
