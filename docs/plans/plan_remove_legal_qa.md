# Plan — Personal Tool에서 Legal Q&A 제거

상태: 범위 파악 완료(grep으로 전수 확인), 인터뷰 불필요, 구현 대기.

## 1. 요청 요약

Personal Tool(`panel-thoughts`)의 sub-nav 3개(Search Stories / Legal
Q&A / Compare) 중 Legal Q&A를 완전히 제거한다.

## 2. 제거 대상 전수 조사 (grep으로 확인)

Legal Q&A는 여러 곳에 걸쳐 있고, 특히 **Archive 카드의 "Use in Legal
Q&A →" 버튼**이 이 기능으로 들어가는 유일한 진입점 중 하나라 같이
정리해야 한다 — Q&A 패널이 없어지면 그 버튼은 존재하지 않는 패널로
전환을 시도하는 죽은 링크가 된다.

- **nav 버튼** — `index.html:418`, `sub-btn-qa`.
- **패널 전체** — `index.html:441-464`, `sub-qa` div(채팅 UI 전체:
  컨텍스트 배지, "Load from archive" 버튼, 채팅 영역, 입력창, 안내
  문구까지 전부 이 블록 안에 있다).
- **`showSubPanel()`의 배열** — `index.html:1138`,
  `['search','qa','compare']` → `['search','compare']`로.
- **Archive → Q&A cross-link(제거 대상)** — `index.html:1167`, 카드
  footer의 "Use in Legal Q&A →" 버튼(`onclick="loadArchiveToQA(${i})"`).
- **`loadArchiveToQA(idx)`**(`index.html:1306-1318`)와 하위호환용
  `loadArchive(idx)`(`index.html:1320`) — 둘 다 Q&A 패널 진입이
  유일한 목적이라 통째로 제거.
- **채팅 관련 함수 5개**(`index.html:1475-1554`,
  `// (c) LEGAL Q&A` 섹션 전체) — `chatKey()`, `ts()`, `addMsg()`,
  `renderLinks()`, `sendMessage()`. grep으로 확인한 결과 이 5개는
  전부 Q&A 안에서만 쓰이고 Search Stories나 Compare 등 다른 기능이
  재사용하는 곳이 없다 — 안전하게 통째로 제거 가능.
- **`S.chatHist` 상태 필드** — `index.html:617`, Q&A 채팅 히스토리
  전용이라 같이 제거.
- **i18n 키 9개 × 6개 언어(54줄)** — `personal.qa_tab`,
  `personal.no_briefing`, `personal.load_from_archive`,
  `personal.qa_welcome`, `personal.now`, `personal.qa_ph`,
  `personal.send`, `personal.qa_note`, `personal.use_in_qa`.

## 3. 구현

### 3-1. nav 버튼 + 패널 HTML 제거

`index.html:418`(`sub-btn-qa` 버튼)과 `index.html:441-464`(`sub-qa`
패널 전체)를 삭제한다. Search Stories와 Compare 두 sub-panel은 그대로
둔다 — sub-nav 자체는 2개 탭으로도 계속 유지한다(Daily Briefing의
Today/Archive 토글도 2개 탭으로 같은 UI 패턴을 쓰고 있어 일관된다).

### 3-2. Archive 카드의 cross-link 제거

`index.html:1167`의 `<button class="btn-outline" onclick="loadArchiveToQA(${i})">${t('personal.use_in_qa')}</button>`
줄을 통째로 삭제한다. Archive 카드 footer엔 더 이상 버튼이 안 남으므로,
`arc-card-footer` div 자체도 비게 되면 같이 제거한다(빈 div가 여백만
차지하지 않도록).

### 3-3. `showSubPanel()` 배열 수정

`index.html:1138`을 `['search','compare']`로 바꾼다.

### 3-4. 관련 함수 6개 + 상태 필드 제거

`loadArchiveToQA()`, `loadArchive()`(`index.html:1306-1320`)와
`// (c) LEGAL Q&A` 섹션 전체(`chatKey`/`ts`/`addMsg`/`renderLinks`/
`sendMessage`, `index.html:1472-1554`)를 삭제한다. `S` 객체 리터럴
(`index.html:617`)에서 `chatHist: [],` 줄을 제거한다.

### 3-5. i18n 키 9개 × 6개 언어 제거

2번에 나열한 9개 키를 `I18N` 객체의 6개 언어 블록 전부에서 삭제한다.

## 4. 테스트 plan

- Personal Tool 진입 시 sub-nav가 Search Stories / Compare 2개만
  뜨는지, 첫 화면이 여전히 Search Stories인지 확인.
- Archive 탭에서 저장된 브리핑 카드를 펼쳐봐서 footer에 아무 버튼도
  안 남았는지(또는 빈 여백이 안 남았는지) 확인.
- 콘솔 에러 없이 페이지가 로드되는지(특히 `showSubPanel()` 배열
  변경 후 Compare 탭 전환이 정상 동작하는지 — `qa` 항목이 빠지면서
  인덱스나 로직이 깨지지 않는지).
- Search Stories, Compare 두 기능이 이번 제거로 영향받지 않고 그대로
  정상 동작하는지(공유 함수가 없다는 2번의 grep 결과를 실제 클릭
  테스트로 재확인).
- UI 언어 6개를 돌려가며 다른 화면 문구가 깨지지 않는지(삭제한 키가
  혹시 다른 곳에서도 재사용되고 있었다면 그 자리가 빈 텍스트로 뜨는지
  확인 — 2번 grep으로 이미 Q&A 전용임을 확인했지만 육안으로 한 번 더
  본다).

## 5. 리스크 / 엣지케이스

`/api/chat`, `/api/legal-search` 같은 서버 엔드포인트는 그대로 둔다 —
`/api/legal-search`는 Q&A 전용이라 이번엔 안 쓰이게 되지만, 서버
엔드포인트 자체를 지우는 건 이번 요청(Personal Tool의 화면 요소 제거)
범위를 넘어서는 별도 정리 작업이라 포함하지 않았다(죽은 엔드포인트가
남아있는 것 자체는 무해하다 — 호출하는 클라이언트 코드가 없으면 그냥
안 쓰일 뿐이다).

## 6. Self-review

**베스트 plan인지** — grep으로 Q&A 관련 코드를 전수 조사해서 빠짐없이
나열했고, Archive cross-link처럼 놓치기 쉬운 연결점도 미리 찾아뒀다.

**빠진 게 있는지** — `addMsg`/`renderLinks`/`ts`가 다른 기능에서
재사용되고 있는지 별도로 grep 검증해서 안전하게 삭제 가능함을
확인했다. `loadArchive()` 하위호환 별칭까지 놓치지 않고 포함했다.

**오버한 게 있는지** — 서버 쪽 `/api/legal-search` 엔드포인트는 안
건드렸다 — 요청은 "Personal Tool에서 Legal Q&A 삭제"였지 서버 API
정리가 아니다.

**테스트 충분한지** — 화면 확인, 콘솔 에러, 인접 기능(Search/Compare)
회귀, 6개 언어 확인까지 포함했다.

## 7. 사용자 결정 필요 항목

없음.
