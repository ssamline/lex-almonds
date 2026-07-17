# Plan: Search Stories 강화 — 스토리 직접 조사 (URL/이미지 업로드 + 조사 유형 선택)

## 0. 배경 및 이전 작업 되돌리기

이전 턴에서 `research-template`(별도 정적 리포트 배포 파이프라인)을 설치하고 `research/index.html` 목록 페이지와 상단 Research 메뉴를 추가했으나, 사용자가 방향을 명확히 바꿨다. 새 Research 페이지 대신 기존 **Personal Tool → Search Stories** 탭을 강화해서, 로그인된 사용자가 사이트 안에서 바로(별도 배포 과정 없이) 스토리 하나를 조사할 수 있게 만드는 것이 목표다.

**되돌릴 대상** (커밋 `f236be6`에서 추가된 것들):

- `research/` 폴더 전체 삭제 — `research/index.html`, `research/report-style.css`, `research/lightbox-init.js`, `research/examples/nintendo-order-history.pdf`. 정적 리포트 배포 방식 자체를 쓰지 않기로 했으므로 자산이 무의미해짐.
- `index.html` 상단 nav의 `🔎 Research` 버튼 제거, 그리고 그 버튼 하나 때문에 추가했던 모바일 반응형 미디어쿼리(`@media (max-width:600px)`, `@media (max-width:420px)` 블록)도 원상복구 — nav 항목이 다시 4개로 줄어들면 그 오버플로 대응 코드가 불필요해짐.
- 6개 언어 i18n 딕셔너리에서 추가했던 `nav.research` 키 6줄 제거.
- 루트 `CLAUDE.md`를 삭제하지 않고 **재작성**한다. 기존 파일은 "로컬 Claude Code로 폴더 만들고 markdown 쓰고 git push로 배포"하는 워크플로우를 문서화하고 있는데, 이는 이번에 만들 "사이트에서 바로 무료로 도는" 워크플로우와 완전히 다르다. 옛 파일을 그대로 두면 나중에 이 프로젝트에서 작업할 Claude 세션이 잘못된 워크플로우를 따라갈 위험이 있어서, 실제 구현된 기능(스토리 조사 유형, 프롬프트 구성 규칙, rate limit)을 반영하는 내용으로 교체한다. **판단 근거**: 사용자가 "워크플로우나 의도는 좋다"고 한 것은 폐기가 아니라 그 의도를 실제 구현에 맞게 옮겨달라는 뜻으로 해석 — CLAUDE.md 자체를 없애는 대신 실제 내용으로 갱신.

## 1. 관련 코드 조사 결과 (요약)

- `index.html`의 `#panel-thoughts` → `#sub-search`가 현재 Search Stories UI. `searchStories()`가 `/api/research-search`를 호출해 사용자가 설정한 뉴스 소스(`S.urls`)와 CourtListener 판례를 키워드로 검색하고, `renderSearchResults()`가 결과를 `.result-card` 리스트로 그린다. 각 결과의 제목을 클릭하면 `openArticle(url, summary)`가 `/api/fetch-article`을 호출해 본문을 스크랩하고 모달에 띄운다.
- `server.js`의 `/api/fetch-article`은 이미 URL → HTML fetch → `<title>`/`<meta description>`/`<p>` 태그 추출 로직을 갖고 있다. 이 추출 로직을 재사용할 수 있다.
- `/api/compare-companies`가 이미 서버에서 Claude API(`claude-haiku-4-5-20251001`)를 직접 호출하는 패턴을 갖고 있다: 컨텍스트를 서버에서 문자열로 구성 → system 프롬프트에 "JSON만 응답" 지시 → 응답을 정규식으로 JSON 블록만 추출해 파싱. 새 기능도 이 패턴을 그대로 따른다.
- `/api/chat`은 클라이언트가 보낸 body를 그대로 Anthropic에 전달하는 범용 프록시라서 모델/프롬프트를 클라이언트가 완전히 통제한다 — 새 기능은 이 방식 대신 `/api/compare-companies`처럼 **서버가 프롬프트를 고정**하는 전용 엔드포인트로 만든다(비용/프롬프트 인젝션 통제 목적).
- 인증: 앱 전체가 `#start-gate`(닉네임+비밀번호)로 막혀 있고, `enterApp()` 성공 후에만 `#app-root`가 보인다(`index.html:2597`). 즉 이 앱은 이미 "로그인 안 하면 아무 기능도 못 쓴다" 구조라서, 이번 기능을 위해 별도 인증을 추가할 필요가 없다 — 지난 턴에 논의했던 "로그인 게이트로 익명 인터넷 노출을 막는다"는 안전장치가 이미 충족된 상태.
- Settings(`#panel-settings`)의 `S.companies`(추적 기업)와 `S.sectors`(업종, `data-sector` 토글)가 "내가 Sources에 선택한 회사나 industry"에 해당. `SECTOR_META`에 업종 코드 → 라벨 매핑이 이미 존재.
- `package.json`은 `express`만 의존성으로 갖고 있고 rate-limit 관련 패키지가 없다 — 새 의존성 추가 없이 순수 JS로 in-memory rate limiter를 작성한다.
- `server.js`에 `app.set('trust proxy', ...)` 설정이 없다. Render는 리버스 프록시 뒤에서 앱을 서빙하므로, 이 설정이 없으면 `req.ip`가 실제 클라이언트 IP가 아니라 프록시 IP로 잡혀 IP 기준 rate limit이 사실상 "전체 사이트 공동 한도"가 돼버린다. 이번 작업에서 `trust proxy` 설정을 추가해야 한다.

## 2. 범위

- **프론트엔드**: `index.html`의 `#sub-search` 안에 새 카드 섹션 추가(스토리 URL 붙여넣기 또는 이미지 업로드 → 조사 유형 버튼 2개 → 결과 렌더링). 각 기존 검색 결과 카드에 "🔬 이 스토리 조사하기" 링크를 추가해 URL을 새 섹션에 미리 채워주는 연결도 포함. 6개 언어 i18n 키 추가.
- **백엔드**: `server.js`에 새 엔드포인트 `POST /api/story-research` 추가. 기존 `/api/fetch-article`의 스크랩 로직을 공용 함수로 추출해 재사용. IP 기준 in-memory rate limiter 추가. `trust proxy` 설정 추가.
- **정리**: 위 0번 항목의 되돌리기 작업(`research/` 삭제, nav 되돌리기, i18n 6줄 삭제, `CLAUDE.md` 재작성).
- **범위 제외**: 이미지/URL을 저장하는 히스토리 기능, 조사 결과를 서버나 Firestore에 저장하는 기능(요청 시 그때그때 렌더링만, 페이지 새로고침하면 사라짐 — 기존 Search Stories 결과도 저장 안 되는 것과 동일한 수준으로 맞춤), 요약/영향조사 외 세 번째 조사 유형.

## 3. 작업 순서

### 3-1. 정리(되돌리기)

1. `research/` 폴더 전체 삭제.
2. `index.html`: nav의 `<a class="nav-btn" id="nav-research" ...>` 줄 삭제, `.nav`/`.nav-btn` 미디어쿼리 2블록 삭제(5개 항목 대응용이었으므로), 6개 언어 dict의 `'nav.research': '...'` 줄 삭제.
3. 루트 `CLAUDE.md`를 3-4단계에서 실제 구현될 내용 기준으로 재작성(간결하게: 이 프로젝트의 스토리 조사 기능이 무엇이고, 조사 유형을 추가/수정할 때 지킬 규칙 — 서버에서 프롬프트 고정, 비용 상한, rate limit 유지 등).

### 3-2. 백엔드 — `server.js`

1. `app.set('trust proxy', true)`를 `const app = express();` 바로 아래에 추가.
2. `/api/fetch-article` 안에 있는 HTML → `{title, desc, paragraphs}` 추출 로직을 `extractArticleText(url)`라는 별도 함수로 뽑아내고, `/api/fetch-article`은 이 함수를 호출하도록 변경(동작은 동일, 재사용을 위한 리팩터).
3. 아주 단순한 in-memory rate limiter 추가:
   ```js
   const RATE_LIMIT = { windowMs: 60 * 60 * 1000, max: 8 }; // IP당 시간당 8회
   const rateHits = new Map(); // ip -> { count, resetAt }
   function checkRateLimit(ip) {
     const now = Date.now();
     const entry = rateHits.get(ip);
     if (!entry || now > entry.resetAt) {
       rateHits.set(ip, { count: 1, resetAt: now + RATE_LIMIT.windowMs });
       return true;
     }
     if (entry.count >= RATE_LIMIT.max) return false;
     entry.count++;
     return true;
   }
   ```
   메모리 누수 방지를 위해 `setInterval`로 만료된 엔트리를 주기적으로(예: 10분마다) 청소하는 코드도 같이 추가한다.
4. 새 엔드포인트:
   ```js
   app.post('/api/story-research', async (req, res) => {
     const ip = req.ip;
     if (!checkRateLimit(ip)) {
       return res.status(429).json({ error: 'Too many research requests. Please try again later.' });
     }

     const { researchType, url, image, companies = [], sectors = [] } = req.body;
     if (!['summary', 'impact'].includes(researchType)) {
       return res.status(400).json({ error: 'Invalid research type.' });
     }
     if (!url && !image) {
       return res.status(400).json({ error: 'Provide a URL or an image.' });
     }
     if (url && image) {
       return res.status(400).json({ error: 'Provide either a URL or an image, not both.' });
     }
     if (researchType === 'impact' && companies.length === 0 && sectors.length === 0) {
       return res.status(400).json({ error: 'Add a tracked company or business sector first.' });
     }

     const apiKey = process.env.ANTHROPIC_API_KEY;
     if (!apiKey) return res.status(500).json({ error: 'ANTHROPIC_API_KEY not set.' });

     // 1. 소스 콘텐츠 준비
     let sourceTitle = '';
     let userContent; // Anthropic content 배열

     if (url) {
       if (!/^https?:\/\//.test(url)) return res.status(400).json({ error: 'Invalid URL.' });
       try {
         const { title, desc, paragraphs } = await extractArticleText(url);
         sourceTitle = title || url;
         const text = (paragraphs.length ? paragraphs.join('\n\n') : desc).slice(0, 6000);
         if (!text.trim()) return res.status(422).json({ error: 'Could not extract readable content from that URL.' });
         userContent = [{ type: 'text', text: `Article title: ${sourceTitle}\n\n${text}` }];
       } catch (e) {
         return res.status(502).json({ error: `Could not fetch that URL: ${e.message}` });
       }
     } else {
       const { data, mediaType } = image || {};
       const allowed = ['image/png', 'image/jpeg', 'image/webp', 'image/gif'];
       if (!data || !allowed.includes(mediaType)) return res.status(400).json({ error: 'Unsupported image.' });
       const approxBytes = data.length * 0.75; // base64 → bytes 근사
       if (approxBytes > 5 * 1024 * 1024) return res.status(413).json({ error: 'Image too large (5MB limit).' });
       sourceTitle = 'Uploaded image';
       userContent = [
         { type: 'image', source: { type: 'base64', media_type: mediaType, data } },
         { type: 'text', text: 'This image shows a news story, legal document, or announcement. Analyze it.' }
       ];
     }

     // 2. 조사 유형별 system 프롬프트
     let system, maxTokens;
     if (researchType === 'summary') {
       maxTokens = 700;
       system = `You are a legal analyst. Given a news story about a law, regulation, or legal/regulatory trend, explain what it covers and why it matters.
   Respond ONLY as valid JSON (no markdown fences):
   {"headline":"one-line description of the law/trend covered","summary":"3-5 sentence plain-language explanation of what it is and why it matters","keyPoints":["point 1","point 2","point 3"]}`;
     } else {
       maxTokens = 1500;
       const targets = [...companies, ...sectors];
       system = `You are a legal intelligence analyst. Given a news story about a law, regulation, or legal/regulatory trend, analyze its potential impact specifically on these companies/industries: ${targets.join(', ')}. If a target is not clearly affected, say so plainly rather than inventing a connection.
   Respond ONLY as valid JSON (no markdown fences):
   {"summary":"2-3 sentence overview of what the story covers","impacts":[{"target":"company or industry name","riskLevel":"High|Medium|Low|None","impact":"1-3 sentence explanation specific to this target"}]}`;
     }

     // 3. Claude 호출
     try {
       const claudeRes = await fetch('https://api.anthropic.com/v1/messages', {
         method: 'POST',
         headers: { 'Content-Type': 'application/json', 'x-api-key': apiKey, 'anthropic-version': '2023-06-01' },
         body: JSON.stringify({
           model: 'claude-haiku-4-5-20251001',
           max_tokens: maxTokens,
           system,
           messages: [{ role: 'user', content: userContent }]
         }),
         signal: AbortSignal.timeout(30000)
       });
       if (!claudeRes.ok) return res.status(500).json({ error: 'Claude API error.' });
       const claudeData = await claudeRes.json();
       const raw = (claudeData.content?.[0]?.text || '').trim();
       const m = raw.match(/\{[\s\S]*\}/);
       const result = m ? JSON.parse(m[0]) : { summary: raw };
       res.json({ result, sourceTitle, researchType });
     } catch (e) {
       res.status(500).json({ error: e.message });
     }
   });
   ```
   (위 코드는 plan 단계의 초안이며, 실제 구현 시 기존 코드 스타일 — 세미콜론 사용 여부, 들여쓰기 — 을 그대로 맞춘다.)

### 3-3. 프론트엔드 — `index.html`

1. `#sub-search` 카드 아래, `search-results` 밖에 새 카드를 추가한다(`.card`, `.card-head`, `.card-body` 기존 패턴 그대로):
   - 헤더: "🔬 스토리 직접 조사" (`personal.direct_research_title`)
   - 설명 한 줄: "링크를 붙여넣거나 이미지를 올리면 어떤 조사를 할지 선택할 수 있어요." (`personal.direct_research_desc`)
   - URL 입력 필드 (`#research-url-input`, `.url-input-row`와 동일 스타일)
   - "또는" 구분선 (`personal.direct_research_or`)
   - 파일 입력 (`#research-image-input`, `type="file" accept="image/png,image/jpeg,image/webp,image/gif"`) + 업로드된 이미지 썸네일 미리보기 영역(`#research-image-preview`, 최대 폭 160px)
   - 조사 유형 버튼 두 개, 기본은 `display:none`이고 URL 입력 또는 이미지 선택이 감지되면 나타남(`#research-type-row`):
     - `📝 요약` → `runStoryResearch('summary')`
     - `📊 내 관심 기업/산업 영향 조사` → `runStoryResearch('impact')` — `S.companies.length===0 && 활성 sector 없음`이면 버튼 대신 힌트 텍스트("Settings에서 기업이나 업종을 먼저 추가하세요")를 보여준다(`#compare-hint`와 동일한 UX 패턴).
   - 스피너(`#research-spin`, 기존 `.hint`+`.spinner` 패턴 재사용)
   - 결과 영역(`#research-result`) — `summary` 결과는 `.biz-card` 스타일로, `impact` 결과는 `.compare-card`/`.risk-badge` 스타일로 렌더링(기존 CSS 클래스 재사용, 새 CSS 거의 추가 안 함).
   - "✕ 초기화" 링크로 입력/결과 리셋.
2. 각 `renderSearchResults()`의 `.result-card` 안에 있는 "Open original ↗" 링크 옆에 "🔬 이 스토리 조사하기" 링크를 추가 — 클릭하면 `prefillResearch(url)`가 `#research-url-input`에 URL을 채우고 이미지 입력을 비운 뒤 조사 유형 버튼을 보여주고 새 카드로 스크롤한다.
3. JS 함수 추가:
   - `handleResearchUrlInput()` — URL 입력에 `input` 이벤트로 연결, 값이 있으면 이미지 입력/미리보기를 지우고 유형 버튼을 보여줌.
   - `handleResearchImageInput(inputEl)` — `change` 이벤트, `FileReader.readAsDataURL`로 base64 인코딩(5MB 초과 시 alert 후 무시), URL 입력을 지우고 미리보기 썸네일 표시, 유형 버튼 표시.
   - `updateResearchButtonsState()` — `S.companies`/활성 `S.sectors` 유무에 따라 영향조사 버튼 vs 힌트 텍스트 토글.
   - `runStoryResearch(type)` — `#research-url-input` 값 또는 저장된 base64/mediaType 중 있는 쪽으로 body 구성, `S.companies`와 활성 sector 라벨(`SECTOR_META` 사용)을 담아 `/api/story-research` POST. 호출 시작 시 두 조사 유형 버튼을 모두 `disabled`로 바꾸고 스피너를 표시해 중복 클릭(그리고 불필요한 rate-limit 소모)을 막고, 응답 성공/실패 어느 쪽이든 `finally`에서 버튼을 다시 활성화한다.
   - `renderStoryResearchResult(researchType, result, sourceTitle)` — type별로 다른 마크업 생성(요약: headline/summary/keyPoints, 영향조사: 각 target별 risk-badge 카드).
   - `resetStoryResearch()` — 입력/미리보기/결과/버튼 상태 초기화.
   - `prefillResearch(url)` — 위 2번 항목.
4. 6개 언어 dict에 새 i18n 키 추가(간결하게 라벨 위주): `personal.direct_research_title`, `_desc`, `_or`, `_upload_hint`, `_summary_btn`, `_impact_btn`, `_impact_hint`, `_reset`, `_researching`, `_research_this`(결과 카드용 링크 라벨).

## 4. 기존 동작에 미치는 영향 / 하위 호환성

- `/api/fetch-article`은 리팩터(추출 로직을 함수로 뽑는 것) 외에는 응답 형식이 그대로라 기존 `openArticle()` 흐름에 영향 없음.
- 기존 Search Stories 키워드 검색·Compare 탭·나머지 모든 패널은 변경 없음.
- `trust proxy` 추가는 Express의 `req.ip`/`req.ips` 계산 방식에 전역으로 영향을 준다 — 다른 엔드포인트가 `req.ip`를 안 쓰고 있으므로 부작용 없음(확인됨: 현재 `req.ip`를 참조하는 코드가 이번에 추가하는 rate limiter 외엔 없음).

## 5. 테스트 plan

- **수동 테스트(로컬 `node server.js`)**:
  - 실제 뉴스 기사 URL을 붙여넣고 "요약" → JSON이 정상 파싱되어 headline/summary/keyPoints가 렌더링되는지 확인.
  - `S.companies`에 기업 2개, 활성 sector 1개를 설정한 상태에서 같은 URL로 "영향 조사" → 각 target별 risk-badge 카드가 렌더링되는지 확인.
  - `S.companies`와 활성 sector가 모두 없는 상태에서는 영향조사 버튼이 힌트 텍스트로 대체되는지 확인.
  - 스크린샷(이미지) 업로드로 "요약" 실행 → Claude가 이미지 vision 입력으로 정상 응답하는지 확인.
  - 5MB 넘는 이미지 업로드 시도 → 클라이언트에서 즉시 차단되는지 확인.
  - 잘못된 URL(예: `not-a-url`) 입력 후 조사 시도 → 400 에러가 사용자에게 보이는 문구로 표시되는지 확인.
  - 같은 IP로 9번 연속 요청 → 9번째(또는 설정한 한도+1번째) 요청이 429로 막히는지 확인.
  - 검색 결과 카드의 "🔬 이 스토리 조사하기" 클릭 → URL이 새 섹션에 채워지고 스크롤되는지 확인.
  - 6개 언어 전환 후 새 UI 텍스트가 모두 번역되어 나오는지 확인(특히 한국어/영어).
- **회귀 테스트**: 기존 키워드 검색, Compare 탭, Daily Briefing, Sharing Thoughts가 변경 전과 동일하게 동작하는지 육안 확인.
- 이 프로젝트에 별도 테스트 러너(pytest 등)가 없으므로, 위 수동 확인이 유일한 검증 수단이다 — 배포 전 로컬에서 `node server.js` 띄우고 브라우저로 위 항목을 실제로 눌러본다.

## 6. 위험 / 엣지케이스

- **긴 기사 본문으로 비용 급증**: `extractArticleText`가 최대 30개 문단을 반환할 수 있어 문단 하나당 최대 2000자면 이론상 6만 자에 달할 수 있다 → 서버에서 6000자로 자르는 로직으로 방지(3-2 §4).
- **paywall·스크랩 실패**: 기존 `/api/fetch-article`도 같은 문제를 겪는 사이트가 있으므로, `paragraphs`와 `desc`가 모두 비어있으면 422로 명확히 실패 처리하고 사용자에게 "본문을 가져올 수 없습니다" 안내.
- **rate limit이 서버 재시작 시 초기화됨**: in-memory라서 배포마다 리셋되지만, 이 앱 규모에선 실용적으로 충분하다고 판단(방문자가 많지 않은 개인/팀 도구).
- **`researchType==='impact'`인데 스토리가 대상과 무관**: 프롬프트에 "명확히 관련 없으면 억지로 연결 짓지 말고 그렇게 말하라" 지시를 넣어 환각성 영향 서술을 줄인다(`riskLevel: "None"` 옵션 포함).
- **이미지가 스토리와 무관한 사진일 경우**: Claude가 스스로 판단해서 "이 이미지에서 법률/트렌드 관련 내용을 찾을 수 없습니다" 류의 답을 내도록 시스템 프롬프트에 이미 암묵적으로 유도되지만, 명시적으로 처리하진 않음 — 결과 그대로 사용자에게 보여주는 것으로 충분(요약/영향조사 모두 자유 텍스트 필드가 있어 자연스럽게 표현 가능).

## 7. Self-review

- **베스트인지**: 새 정적 페이지·배포 파이프라인 없이 기존 SPA 패널과 기존 서버 패턴(`/api/compare-companies`와 동일한 "서버가 프롬프트 고정 + JSON 강제" 구조)을 그대로 재사용하므로 유지보수 부담이 낮고, 기존 로그인 게이트를 그대로 활용해 별도 인증 구현이 필요 없다.
- **빠진 거 없는지**: 요청하신 두 가지 조사 유형(요약, 영향조사), URL과 이미지 두 입력 방식, 기존 Search Stories와의 연결(🔬 이 스토리 조사하기), 비용/악용 방지(rate limit, 텍스트/이미지 크기 상한)를 모두 포함했다.
- **오버한 거 없는지**: 세 번째 조사 유형이나 결과 저장·히스토리 기능은 요청받지 않았으므로 추가하지 않았다. Redis 등 외부 rate-limit 저장소나 큐 시스템도 이 규모에는 과하다고 판단해 제외했다.

## 8. 사용자 결정이 필요한 항목

없음 — 위 5번 각주에서 이미 사용자가 준 정보(조사 유형 2가지, 강화 대상 tool, 새 페이지 삭제, 이전 답변 참고)로 모든 아키텍처 결정이 가능했다. rate limit 수치(시간당 8회)와 이미지 크기 상한(5MB)은 조정 가능한 파라미터이며, 필요시 구현 후 쉽게 바꿀 수 있다.
