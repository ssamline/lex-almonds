# Plan — Archive 검색 완성(빈 결과 안내 + 원문 기사 제목까지 검색)

상태: 설계 완료, 인터뷰 불필요(세부 사항 위임받음), 구현 대기.

## 1. 현재 상태 파악 — 이미 있는 것

Archive 검색은 이미 존재한다. `#archive-search` 입력창(`index.html:383`)이
`oninput`으로 `filterArchive(q)`(`index.html:1178`)를 호출하고, 이 함수는
각 저장된 브리핑(`S.archive[i]`)의 `date`(날짜) + `topics`(그날 활성화된
토픽 라벨, 사실상 "제목" 역할) + `context`(AI가 생성한 본문 요약 전체
텍스트)를 합쳐서 소문자 비교로 부분 일치하면 카드를 보여주고 아니면
`display:none`으로 숨긴다. 즉 "제목이나 본문에 특정 단어가 포함된
daily briefing만 필터링"이라는 요청의 핵심은 이미 동작하고 있다.

빠진 게 두 가지다.

- 검색어와 매칭되는 카드가 하나도 없을 때 화면에 아무 안내가 없다 —
  그냥 전부 `display:none`이 되면서 텅 빈 흰 화면만 남는다. 사용자
  입장에선 이게 "결과 없음"인지 "로딩 중"인지 "고장"인지 구분이 안 된다.
- 검색 대상이 AI가 요약한 `context`까지만 포함하고, 그날 실제로 가져온
  기사 원문 제목(`e.articles`, RSS에서 스크랩한 `{title,link,domain}`
  배열, `generateBriefing()`이 저장 시 같이 넣어 둔다)은 검색 대상이
  아니다. AI 요약 불릿에 안 뽑힌 기사 제목이나 회사명은 지금 검색으로
  못 찾는다.

## 2. 이번 plan의 범위

1번의 두 gap만 고친다. 검색 하이라이트(매칭 단어 강조 표시)나 "N개 중
M개 표시" 같은 카운터는 넣지 않는다 — 요청하신 핵심(제목·본문 검색)은
이미 있고, 이번엔 그 위에 진짜 빠진 것만 채우는 선에서 멈춘다(6번
참고).

## 3. 구현

### 3-1. `filterArchive()` — 원문 기사 제목까지 검색 대상에 포함 + 매칭 수 추적

`index.html:1178-1186`을 아래로 교체한다.

```js
function filterArchive(q) {
  q = q.trim().toLowerCase();
  let matched = 0;
  document.querySelectorAll('#archive-cards .arc-card').forEach((card, i) => {
    const e = S.archive[i];
    if (!e) return;
    const articleTitles = (e.articles || []).map(a => a.title).join(' ');
    const hay = `${e.date} ${e.topics} ${e.context || ''} ${articleTitles}`.toLowerCase();
    const isMatch = !q || hay.includes(q);
    card.style.display = isMatch ? '' : 'none';
    if (isMatch) matched++;
  });
  const noMatchEl = document.getElementById('archive-no-match');
  if (noMatchEl) noMatchEl.style.display = (q && matched === 0 && S.archive.length > 0) ? 'block' : 'none';
}
```

`e.articles`는 이미 각 archive entry에 저장돼 있는 필드다(`generateBriefing()`의
`entry` 객체, `index.html` — `articles: S.articles`) — Firestore
읽기/쓰기 스키마 변경이 전혀 필요 없다, 이미 있는 데이터를 검색
하이스택에 한 줄 추가하는 것뿐이다.

### 3-2. `renderArchive()` — 재렌더 후 현재 검색어 재적용

`index.html:1149-1169`의 `renderArchive()` 맨 끝(`el.innerHTML = ...`
다음)에 한 줄 추가한다.

```js
function renderArchive() {
  const el = document.getElementById('archive-cards');
  if (!S.archive.length) {
    el.innerHTML = `<div class="hint">${t('brief.no_saved')}</div>`;
    return;
  }
  el.innerHTML = S.archive.map((e, i) => `...`).join('');
  filterArchive(document.getElementById('archive-search')?.value || '');
}
```

이유 — `renderArchive()`는 `loadArchiveFromCloud()`(로그인 시)나 새
브리핑 저장 시에도 다시 호출되는데, 그때마다 `#archive-cards`의
`innerHTML`이 통째로 새로 그려지면서 이전에 입력해 둔 검색어와 무관하게
모든 카드가 다시 보이는 상태로 리셋된다. 검색창에 뭔가 입력해 둔 채로
Archive가 재렌더되는 상황(예: 다른 탭에서 새 브리핑을 생성한 직후
Archive로 돌아옴)에서 검색 결과가 조용히 풀려버리는 걸 막는다.

### 3-3. HTML — 빈 결과 안내 엘리먼트

`index.html:403-405`(`#archive-cards`) 바로 다음에 형제 엘리먼트로
추가한다 — `#archive-cards` 안에 넣으면 `renderArchive()`가 매번
`innerHTML`을 통째로 교체할 때 같이 지워지므로 반드시 밖에 둬야 한다.

```html
<div id="archive-cards">
  <div class="hint">No saved briefings yet. Generate your first briefing on the Today tab.</div>
</div>
<div id="archive-no-match" class="hint" style="display:none" data-i18n="archive.search_no_results">No briefings match your search.</div>
```

### 3-4. i18n — 새 키 1개, 6개 언어

`archive.search_ph` 옆에 추가한다.

| 언어 | `archive.search_no_results` |
|---|---|
| en | No briefings match your search. |
| ko | 검색어와 일치하는 브리핑이 없어요. |
| es | Ningún resumen coincide con tu búsqueda. |
| fr | Aucun résumé ne correspond à votre recherche. |
| zh | 没有符合搜索条件的简报。 |
| ja | 検索条件に一致するブリーフィングがありません。 |

## 4. 테스트 plan

- 저장된 브리핑이 여러 개 있는 상태에서 아무 검색어도 매칭 안 되는
  키워드를 입력 — 카드가 전부 사라지고 "검색어와 일치하는 브리핑이
  없어요" 안내가 뜨는지 확인.
- 검색어를 지우면(빈 문자열) 안내가 사라지고 카드가 전부 다시 보이는지.
- AI 요약 불릿에는 안 뽑혔지만 그날 실제로 가져온 기사 원문 제목에만
  있는 단어로 검색 — 이번에 새로 검색 대상이 된 `e.articles` 덕분에
  해당 브리핑이 정상적으로 걸리는지(3-1에서 추가한 부분의 핵심 검증).
- 기존 회귀 — 날짜, 토픽 라벨, `context`(요약 본문) 키워드로 검색했을
  때 기존처럼 정상 필터링되는지.
- 검색어를 입력해 둔 상태에서 Today 탭에서 새 브리핑을 하나 생성한 뒤
  다시 Archive 탭으로 돌아왔을 때, 검색 결과가 풀리지 않고 그대로
  유지되는지(3-2에서 고친 부분의 핵심 검증).
- Archive가 완전히 비어 있는 상태(브리핑 0개)에서는 기존
  "저장된 브리핑이 아직 없어요" 안내만 뜨고, `archive-no-match`
  안내는 안 뜨는지(두 안내가 동시에 안 겹치는지 — `S.archive.length > 0`
  조건으로 이미 분리해 뒀다).
- UI 언어 6개를 돌려가며 새 안내 문구가 정확히 번역되는지.

## 5. 리스크 / 엣지케이스

`e.articles`가 없는(과거 버전에서 생성된, 또는 필드 자체가 비어있는)
아주 오래된 archive entry가 있어도 `(e.articles || []).map(...)`가
안전하게 빈 배열로 처리하므로 에러 없이 넘어간다.

## 6. Self-review

**베스트 plan인지** — 새 Firestore 필드나 서버 API 없이, 이미 저장돼
있는 `e.articles` 데이터를 검색 하이스택에 포함시키는 것과 빈 결과
안내 엘리먼트 하나 추가하는 게 전부다. 최소 변경으로 실제로 빠져있던
두 가지만 채운다.

**빠진 게 있는지** — `renderArchive()`가 여러 곳에서 재호출되는 걸
확인해서, 검색어가 재렌더 후에도 유지되도록 챙겼다. 빈 결과 안내
엘리먼트가 `#archive-cards` 밖에 있어야 재렌더 때 안 지워진다는 것도
명시했다.

**오버한 게 있는지** — 매칭 단어 하이라이트, 결과 개수 카운터, 검색
결과 정렬 옵션 모두 넣지 않았다 — 요청하신 핵심(제목·본문 검색)은
이미 있었고, 이번엔 진짜 빠진 부분(빈 결과 안내, 원문 기사 제목 검색)
만 채우는 선에서 의도적으로 멈췄다.

**테스트 충분한지** — 신규 기능(원문 기사 검색, 빈 결과 안내, 재렌더
후 검색어 유지) 세 가지와 기존 검색 회귀, 언어 전환까지 포함했다.

## 7. 사용자 결정 필요 항목

없음 — 세부 사항을 이미 위임받았다.
