# Plan — Daily Briefing에 외국어 소스 요약 섹션 추가

상태: 설계 완료(인터뷰 1건 반영), 구현 대기.

## 0. 인터뷰 결정 사항

사용자가 등록한 뉴스 소스 중 서로 다른 언어의 비영어 소스가 동시에 여러 개
감지될 경우(예: 프랑스어 소스 + 일본어 소스), **감지된 언어마다 별도 섹션을
하나씩** 브리핑 맨 끝에 붙이기로 확정했다. 소스가 한두 개 늘어난다고 정보가
누락되지 않는 쪽을 택한 것이다.

## 1. 요청 요약

사용자가 Sources & Topics에 등록한 뉴스 소스 중 영어가 아닌 소스가 있으면,
Daily Briefing 화면 맨 끝에 그 소스의 원래 언어로 쓰인 요약 섹션을 추가해
달라는 요청이다. 요약 범위는 지금 활성화된 Legal Topics와 Business Sectors
(사용자가 "legal topics and business topics"라고 표현한 것 — 이 앱에서는
Business Sectors가 그 역할을 한다)로 한정한다.

## 2. 현재 코드베이스 파악 결과

`generateBriefing()`(`index.html:904`)이 Daily Briefing 생성의 핵심 함수다.
흐름은 다음과 같다.

- `S.urls`(사용자가 등록한 소스 도메인 목록, `index.html:611`)를 `/api/search-news`
  (`server.js:82`)로 보내면, 서버의 `fetchSiteArticles(domain)`
  (`server.js:46`)이 각 도메인의 RSS/Atom 피드를 파싱해 `{title,link,domain}`
  형태의 기사 목록을 반환한다. 이 시점엔 언어 정보가 전혀 없다 — 그냥 제목
  텍스트뿐이다.
- 가져온 기사 제목들은 `articleCtx`(`index.html:940-943`)로 문자열화되어
  `system` 프롬프트(`index.html:947-951`)에 그대로 포함된 뒤, `/api/chat`
  (`server.js:218`, Claude Haiku 호출)에 보내진다. AI는 `{"sections":[...]}`
  형태의 JSON만 반환하도록 지시받는다.
- 응답을 파싱해 `S.context`(`index.html:998-1003`, 검색·Trend 분석·Legal
  Q&A 컨텍스트 로드에 재사용되는 평문 문자열)를 만들고, 같은 `parsed.sections`
  배열로 `#brief-out`의 HTML(`index.html:1022-1048`)을 렌더링한다.
- 완성된 `entry`(html/context/articles 포함, `index.html:1052-1061`)가
  `localStorage`와 Firestore Archive(`saveArchiveEntryToCloud`)에 저장된다.

이 전체 구조가 이미 "AI에게 기사 제목만 주고 JSON 스키마로 합성시킨다"는
패턴이라, 이번 기능도 같은 패턴을 그대로 확장하는 것이 가장 이 코드베이스와
일관된 방식이다 — 별도 언어 감지 라이브러리나 새 서버 엔드포인트를 추가하지
않고, 이미 호출하고 있는 같은 `/api/chat` 요청 안에서 AI가 제목 언어까지
같이 판단하게 한다.

관련해서 이미 존재하는 다른 흐름들도 확인했다.

- **Podcast(Listen) 기능**(`listenBriefing()`, `index.html:1523`)은 DOM에서
  `#brief-out .brief-section .prose`를 그대로 긁어 TTS 스크립트를 만든다.
  즉 새 섹션도 같은 `.brief-section`/`.prose` 클래스 구조로 렌더링하면 별도
  코드 추가 없이 자동으로 podcast 스크립트에 포함된다.
- **Trend 분석**(`runTrendAnalysis()`, `index.html:1184-1204`)의 실제 필터링
  로직을 다시 확인했다.
  ```js
  const blocks = inRange.flatMap(e => {
    const ctxBlocks = (e.context || '').split('\n\n');
    return selectedLabels.length
      ? ctxBlocks.filter(b => selectedLabels.some(l => b.startsWith('[' + l + ']')))
      : ctxBlocks;
  });
  ```
  `selectedLabels.length`가 0일 때(= 사용자가 토픽을 하나도 안 골라서 "전체
  포함" 모드일 때, UI 힌트 문구 "Leave all topics unselected to include
  everything"가 가리키는 바로 그 상태) `ctxBlocks`를 필터링 없이 통째로
  쓴다 — 즉 이 분기에서는 `[French Summary]` 같은 새 블록도 그대로 Trend
  분석 프롬프트에 섞여 들어간다. 처음 작성한 plan에서 "라벨이 안 맞아서
  자동으로 빠진다"고 썼던 건 `selectedLabels.length > 0`인 경우에만 맞는
  얘기였고, 기본값(전체 포함) 상태에서는 틀린 주장이었다 — 3-6에서
  `runTrendAnalysis()` 자체에 한 줄 필터를 추가해서 바로잡는다.
- **Firestore 저장**(`firestore.rules:93-96`)은 `html`(≤200,000자)/`context`
  (≤50,000자) 필드 크기 제한만 있고 내용 형식은 검증하지 않는다 — 이번
  추가 텍스트는 이 여유 범위 안에 충분히 들어가므로 `firestore.rules` 변경이
  필요 없다.
- **Archive 검색**(`filterArchive()`, `index.html:1138`)은 `e.context`
  전체를 대소문자 무시 검색하므로, 외국어 요약도 자동으로 검색 대상에
  포함된다.

## 3. 구현 — `generateBriefing()` 프롬프트/스키마 확장

### 3-1. 시스템 프롬프트에 언어 감지 지시 추가

`hasArticles`가 true인 분기(`index.html:948`)에만 아래 지시를 추가한다 —
실제로 가져온 기사 제목이 있어야 AI가 언어를 판단할 근거가 있고, 기사가
하나도 없으면(`hasArticles`가 false) 애초에 판단할 데이터가 없으므로 이
분기에서는 언급하지 않고 클라이언트에서 빈 배열로 기본 처리한다(3-2 참고).

```js
// hasArticles가 true인 분기 안에서만 스플라이스되므로(아래 system 템플릿
// 참고) 여기서 다시 hasArticles를 조건으로 걸 필요는 없다 — 중복 가드를
// 피하려고 처음 작성한 plan의 삼항연산자를 제거하고 단일 문자열로 뒀다.
const foreignInstr = ` Additionally, inspect the article list above by their (domain) prefix — if any domain's article titles are written in a language other than English, identify each such distinct language. For each one, write a 3-5 sentence summary paragraph, written ENTIRELY in that language, covering only the legal and business developments implied by that domain's titles, scoped to topics: ${topicNames}${activeSectors.length ? ' and business sectors: ' + activeSectors.join(', ') : ''}. Add one entry per distinct non-English language to a "foreignSummaries" array: {"language":"<language name in English, e.g. French>","text":"<summary written in that language>"}. If every source domain's titles are in English, return "foreignSummaries": [].`;
```

`system` 템플릿(`index.html:947-951`)의 JSON 스키마 예시 줄도 함께 갱신한다.

```js
const system = `You are a senior legal news analyst. Generate a concise daily legal briefing for ${today}. Topics: ${topicNames}.${kwExtra}${bizInstr}
${hasArticles ? articleCtx + `\n\nUsing ONLY the articles listed above, write 2-3 bullet summaries per section. Each bullet MUST reference one article by its number using "ref": <integer>. Do NOT copy URLs into the JSON.${foreignInstr}` : 'No live articles available — generate a plausible briefing based on current legal trends. Omit "ref" from bullets.'}
Reply ONLY in valid JSON, no markdown fences:
{"sections":[{"topic":"ip","bullets":[{"text":"one-sentence summary of the article","ref":1}],"prose":"...","biz":[{"type":"opportunity","company":"","sector":"","text":"..."}]}],"foreignSummaries":[{"language":"French","text":"..."}]}
Only include these topics: ${activeTopics.join(',')}.`;
```

### 3-2. 응답 파싱 — 안전한 기본값

`parsed.sections.forEach(...)`(`index.html:985`) 다음 줄에 아래를 추가해서,
AI가 필드를 아예 생략하거나 배열이 아닌 값을 보내는 경우까지 방어한다.

```js
const foreignSummaries = Array.isArray(parsed.foreignSummaries) ? parsed.foreignSummaries : [];
```

### 3-3. `S.context`에 병합

`S.context = parsed.sections.map(...).join('\n\n')`(`index.html:998-1003`)
뒤에 이어서 붙인다.

```js
if (foreignSummaries.length) {
  S.context += '\n\n' + foreignSummaries.map(fs => `[${fs.language} Summary]\n${fs.text}`).join('\n\n');
}
```

대괄호 헤더 형식(`[Label]`)을 기존 topic 블록과 동일하게 맞춰서, Archive
키워드 검색이나 Legal Q&A 컨텍스트 로드에서 자연스럽게 같은 방식으로
취급되게 한다. Legal Q&A(`index.html:1476`)는 `S.context` 전체를
"Loaded briefing" 배경 지식으로 시스템 프롬프트에 그대로 넣는데, 여기엔
외국어 요약 블록이 섞여 들어가도 문제없다 — Claude는 다국어 컨텍스트를
읽는 데 문제가 없고, 오히려 사용자가 그 요약 내용에 대해 질문할 수도
있으므로 의도적으로 그대로 둔다. 반면 Trend 분석은 위에서 확인한 대로
"전체 포함" 모드에서 실제로 새어 들어가는 문제가 있어 3-6에서 별도로
막는다.

### 3-4. 렌더링 — 맨 끝에 섹션 추가

`out.innerHTML = parsed.sections.map(...).join('')`(`index.html:1022-1048`)
바로 다음 줄에서, 기존 `.brief-section` 템플릿과 같은 구조로 이어붙인다.

```js
if (foreignSummaries.length) {
  out.innerHTML += foreignSummaries.map(fs => `
    <div class="brief-section">
      <hr class="divider">
      <span class="topic-tag" data-t="foreign">${t('brief.foreign_summary_label', {lang: fs.language})}</span>
      <p class="prose">${fs.text}</p>
    </div>`).join('');
}
```

처음 작성한 plan에서는 `fs.language`를 `esc()`로 감쌌는데, 다시 확인해보니
`esc()`(`index.html:1639`, `function esc(s){return (s||'').replace(/\\/g,'\\\\').replace(/'/g,"\\'")}`)
는 `onclick='...'` 같은 **JS 문자열 리터럴 attribute** 안에 안전하게 넣기
위한 이스케이프이지, `<span>` 태그의 텍스트 콘텐츠로 들어가는 이 자리에는
맞는 함수가 아니다(HTML 이스케이프를 전혀 안 함 — `<`/`>`/`&`는 그대로
통과). 다만 같은 함수 안 기존 bullet 렌더링(`index.html:1030-1038`)도
`${text}`를 HTML 이스케이프 없이 그대로 `<li>${text}</li>`에 꽂고 있어서,
AI가 생성한 텍스트를 HTML 이스케이프 없이 innerHTML에 넣는 건 이
`generateBriefing()` 함수 전체에 이미 있는 기존 관행이다. 이번 기능만
새로 이스케이프를 넣으면 오히려 이 함수 안에서 일관성이 깨지고, 전체
함수의 이스케이프 정책을 다시 설계하는 건 이번 요청 범위를 넘어서므로
`esc()`를 빼고 기존 관행 그대로 `${fs.language}`를 직접 넣는다 — 이건
새로 만드는 문제가 아니라 기존에 이미 받아들여진 리스크의 연장선이라는
점을 5번 리스크 섹션에도 명시해 둔다.

`data-t="foreign"`을 새로 붙인 이유는 CSS 때문이다. `.topic-tag`
(`index.html:81`)는 배경색이 없고, `data-t` 값별로 별도 규칙
(`index.html:82-86`, `ip`/`reg`/`lit`/`corp`/`biz` 다섯 개만 정의됨)이
색을 입힌다. 처음 작성한 plan은 `data-t` 속성 자체를 빼먹어서, 실제로
구현했다면 이 라벨만 배경색·글자색 없이 밋밋하게(사실상 스타일 없이)
떴을 것이다. 3-6에서 새 CSS 규칙을 하나 추가한다.

bullets나 biz 카드 없이 `.prose` 문단 하나만 두는 이유는 요청 문구가
"a last section on the summary"(단일 요약 문단)였기 때문이다 — 기존
topic 섹션처럼 불릿/기회·리스크까지 복제하면 "요약"의 범위를 넘어선다고
판단했다. `.brief-section`/`.prose` 클래스를 그대로 재사용했기 때문에
`listenBriefing()`의 DOM 스캔에 별도 코드 추가 없이 자동으로 포함된다 —
즉 팟캐스트 재생에도 이 외국어 문단이 자연스럽게 들어간다(OpenAI TTS
`nova` 보이스가 해당 언어 발음을 완벽하게 처리 못 할 수 있지만, 이 앱의
기존 방식(무엇이든 있는 그대로 재생)과 일관되게 별도 예외 처리를 넣지
않기로 결정했다).

### 3-5. i18n — 새 키 1개 추가

`I18N` 객체(`index.html:1690` 부근)의 6개 언어 블록 각각에
`brief.foreign_summary_label` 키를 추가한다. `{lang}` 자리에는 AI가 반환한
언어 이름(영어 표기, 예: "French")이 그대로 들어간다 — 이건 UI 크롬 텍스트가
아니라 AI가 생성한 값이라 번역 대상이 아니다(기존 `spec_site_restructure_i18n.md`
6-2에서 확정한 "AI 생성 콘텐츠는 번역 범위 밖" 원칙과 같은 경계).

| 언어 | 값 |
|---|---|
| en | `🌐 Summary in {lang}` |
| ko | `🌐 {lang} 요약` |
| es | `🌐 Resumen en {lang}` |
| fr | `🌐 Résumé en {lang}` |
| zh | `🌐 {lang}摘要` |
| ja | `🌐 {lang}要約` |

### 3-6. CSS — `topic-tag` 새 variant

`:root` 변수 블록(`index.html:11-15`)에 다섯 번째 태그 색과 나란히 하나
추가한다.

```css
--tag-foreign:#F1F5F9; --tag-foreign-t:#334155;
```

`.topic-tag[data-t="biz"]` 규칙(`index.html:86`) 바로 다음 줄에 매칭 규칙을
추가한다.

```css
.topic-tag[data-t="foreign"] {background:var(--tag-foreign); color:var(--tag-foreign-t)}
```

기존 5색(indigo/orange/green/purple/emerald)과 겹치지 않는 중립 슬레이트
톤을 골랐다 — 외국어 요약은 법률 토픽이나 비즈니스 섹터 색 체계에 속하지
않는 별개 성격의 섹션이라, 기존 색상 중 하나를 억지로 재사용하지 않고
새 variant를 만드는 쪽이 맞다고 판단했다.

### 3-7. `runTrendAnalysis()` — 외국어 블록 항상 제외

2번에서 확인한 대로, `selectedLabels.length`가 0일 때(토픽 전체 포함
모드) 필터링이 아예 안 걸려서 외국어 요약 블록이 Trend 분석 프롬프트에
섞여 들어간다. `index.html:1198-1204`의 블록을 아래로 교체한다.

```js
const KNOWN_LABELS = [...Object.values(TOPIC_META).map(m => m.label), ...Object.values(SECTOR_META)];
const selectedLabels = S.trendConfig.topics;
const blocks = inRange.flatMap(e => {
  const ctxBlocks = (e.context || '').split('\n\n')
    .filter(b => KNOWN_LABELS.some(l => b.startsWith('[' + l + ']')));
  return selectedLabels.length
    ? ctxBlocks.filter(b => selectedLabels.some(l => b.startsWith('[' + l + ']')))
    : ctxBlocks;
});
```

먼저 `KNOWN_LABELS`(legal topic 4개 + business sector 9개 라벨)와 매칭되는
블록만 남기는 필터를 앞에 하나 더 걸어서, 토픽을 선택했든 안 했든 두
분기 모두에서 외국어 요약 블록(그리고 앞으로 추가될 수 있는 다른 어떤
비-topic 블록도)이 걸러지게 한다. 기존 필터를 대체하는 게 아니라 앞에
한 단계를 더 끼워 넣는 구조라 `selectedLabels.length > 0`일 때의 기존
동작(사용자가 고른 토픽만 남기기)은 그대로 유지된다.

## 4. 테스트 plan

- 기본 소스(law360.com, courthousenews.com, 둘 다 영어)만 있는 상태로
  브리핑을 생성해서 외국어 섹션이 전혀 안 뜨는지 확인(오탐 없는지가
  핵심 — 실제로 영어인데 AI가 다른 언어로 착각해 섹션을 만들어내면 안 됨).
- 프랑스어 법률 뉴스 RSS 소스(예: `dalloz-actualite.fr`)를 하나 추가하고
  Renew — 브리핑 맨 끝에 "🌐 Summary in French"(현재 UI 언어에 따라 라벨은
  바뀌고 본문은 프랑스어 그대로) 섹션이 뜨는지, 내용이 실제로 프랑스어인지
  확인.
- 서로 다른 언어의 비영어 소스 두 개(예: 프랑스어 + 일본어)를 동시에 등록하고
  Renew — 인터뷰에서 확정한 대로 섹션이 두 개(언어별로 하나씩) 뜨는지 확인.
- 등록은 했지만 그날 RSS가 기사를 하나도 못 가져온 외국어 소스가 있을 때
  해당 언어 섹션이 조용히 빠지는지(에러 문구 없이) 확인.
- 외국어 섹션이 포함된 브리핑에서 Listen 버튼을 눌러 팟캐스트 생성 시 에러
  없이 재생되는지(발음 품질 자체는 평가 대상 아님, 크래시 여부만 확인).
- 그 브리핑을 Archive에 저장한 뒤 키워드 검색창에 외국어 요약에만 있는
  단어를 입력해서 검색되는지 확인.
- 외국어 소스가 활성화된 상태에서 Trend 분석을 **토픽을 아무것도 선택 안 한
  기본(전체 포함) 모드**로 돌려서, 3-7에서 고친 필터가 실제로 외국어 요약
  블록을 걸러내는지 확인 — 처음 plan에서 놓쳤던 지점이라 별도 항목으로
  명시해 둔다. 토픽을 하나 이상 선택한 모드에서도 마찬가지로 안 섞이는지
  확인.
- 외국어 요약 섹션의 라벨(`🌐 Summary in French` 등)이 3-6에서 추가한
  CSS로 실제 배경색·글자색이 입혀져서 뜨는지 육안으로 확인(스타일 없이
  밋밋하게 뜨면 CSS 누락 회귀).
- UI 언어를 6개 다 돌려가며 라벨("🌐 Summary in French" 등)의 앞부분만
  번역되고 `{lang}` 자리와 본문(`fs.text`)은 그대로 원래 언어로 남아있는지
  확인 — 이번 기능의 핵심 경계(UI 크롬 vs AI 콘텐츠)가 실제로 지켜지는지
  보는 테스트다.

## 5. 리스크 / 엣지케이스

AI가 언어를 잘못 판별할 가능성(예: 스페인어를 포르투갈어로 착각)은 낮지만
존재한다 — 서버 쪽에 별도 언어 검증 로직을 두지 않기로 했으므로(2번 참고,
기존 topic/bullet 생성도 전부 AI 판단을 그대로 신뢰하는 방식이라 일관성
유지) 이 리스크는 이 앱의 기존 신뢰 모델 연장선에서 받아들인다. 실사용
중 오분류가 잦으면 그때 도메인별 언어 힌트(예: `.fr`/`.jp` TLD)를 프롬프트에
추가하는 걸 후속 개선으로 고려할 수 있다.

외국어 소스를 아주 많이(4개 이상) 등록하면 언어별 섹션이 그만큼 늘어나
브리핑 페이지가 길어질 수 있다 — 인터뷰에서 이 트레이드오프를 알고
"언어별 섹션" 방식을 택했으므로 별도 상한을 두지 않는다.

`fs.language`/`fs.text`는 3-4에서 확인한 대로 HTML 이스케이프 없이
innerHTML에 직접 들어간다. AI가 악의적인 HTML/스크립트 조각을 그 필드에
끼워 넣도록 프롬프트 인젝션에 성공하면 이론적으로 XSS 표면이 된다 — 다만
이건 `generateBriefing()`의 bullet/prose 렌더링(`index.html:1038`,
`index.html:1041`)에도 이미 동일하게 존재하는, 이 기능 이전부터 있던
리스크이고 이번 변경이 새로 만든 문제는 아니다. 전체 함수의 이스케이프
정책을 이번 기능 범위에서 같이 고치지는 않기로 했다(오버 방지, 6번 참고).

## 6. Self-review

**베스트 plan인지** — 새 서버 엔드포인트, 새 언어 감지 라이브러리, 새
Firestore 필드 없이 기존 `/api/chat` 요청 한 번과 기존 렌더링 패턴만으로
구현된다. Podcast/Archive/검색/Trend 분석 네 기능 모두 이미 있는 재사용
지점(`​.brief-section`/`.prose` 클래스, `S.context`의 `[Label]` 블록 규칙)에
자연스럽게 올라타서 추가 코드가 최소화됐다.

**빠진 게 있는지** — 이 기능을 끄고 켜는 별도 설정 토글은 만들지 않았다.
비영어 소스가 없으면 섹션 자체가 안 뜨는 자동 동작이라 토글이 없어도
사용자가 원치 않는 걸 억지로 보게 되는 상황이 없다고 판단해서 뺐다 —
필요해지면 나중에 Sources & Topics에 체크박스 하나 추가하는 정도로 쉽게
확장 가능하다.

**오버한 게 있는지** — 서버 쪽 결정론적 언어 감지(예: `franc` 같은 npm
패키지 추가)를 검토했지만, 이미 AI 판단에 크게 의존하는 기존 코드베이스
철학과 어긋나고 새 의존성을 추가하는 것도 이번 범위에서는 과하다고 봐서
빼고 AI 단일 판단으로 결정했다. 외국어 섹션에 기존 topic 섹션처럼 불릿·
기회/리스크 카드까지 복제하는 것도 요청 문구("summary")를 넘어서는 확장이라
넣지 않았다. `generateBriefing()` 전체의 HTML 이스케이프 정책을 이번
기회에 같이 손보고 싶은 유혹이 있었지만, 그건 이번 요청과 무관한 기존
코드 전체의 리팩터라 범위를 넘어선다고 보고 5번에 리스크로만 기록하고
손대지 않았다.

**테스트 충분한지** — 4번에 오탐 방지, 다중 언어, 빈 피드, Podcast, Archive
검색, 6개 언어 UI 크롬까지 포함했다. 검증 과정에서 Trend 분석의 "토픽
전체 포함" 기본 모드가 실제로는 필터링이 안 걸려서 외국어 블록이 새는
버그를 찾아 3-7로 고쳤고, 그 회귀를 잡는 테스트 항목도 4번에 추가했다 —
이 부분은 처음 plan에서는 아예 없던 케이스였다. 다만 AI가 실제로 생성하는
외국어 문장이 자연스러운지는 자동화 테스트로 검증할 수 없는 영역이라,
구현 후 실제 프랑스어/일본어 등 원어민이 아니어도 기계번역 수준으로
확인하는 정도의 육안 검수가 필요하다.

## 7. 사용자 결정 필요 항목

없음 — 유일한 트레이드오프였던 다중 외국어 처리 방식은 0번에서 이미
인터뷰로 확정됐다.
