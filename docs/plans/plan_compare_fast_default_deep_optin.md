# Plan: Compare Companies — 빠른 기본 비교 + 선택적 심층 조사

## 1. 문제 정의

Compare Companies가 매번 회사당 Sonnet 5 + web_search/web_fetch 실시간 조사를 돌리는 구조라, effort를 high→medium→low로 낮추고 조사 범위를 30일 + 3개 우선순위로 좁혀도 여전히 사용자가 원하는 속도(60초 근처)와 비용 수준에 못 미친다. 오늘까지 시도한 프롬프트/파라미터 튜닝(effort, max_uses, 우선순위 범위, M&A 추가/제거)으로는 구조적 한계 — 매 비교마다 실시간 웹 검색을 여러 번 도는 것 자체가 걸리는 시간 — 를 깰 수 없다는 게 반복 검증됐다.

또한 effort:low로 낮춘 뒤 사용자가 "한 쪽 결과만 나오네"라고 보고 — 회사 하나의 조사가 실패하는 기존 신뢰도 문제가 effort를 낮출수록 더 자주 나타날 수 있다는 신호.

## 2. 해결 방향 (사용자 승인 완료)

Compare Companies를 두 단계로 나눈다.

- **기본(빠름/거의 무료)**: 지금 이미 무료로 모으고 있는 SEC EDGAR + CourtListener + 사용자 뉴스 소스 데이터를 그대로 Haiku 한 번 호출에 넣어서 비교 결과를 만든다. 실시간 웹 검색을 전혀 안 쓰므로 몇 초~10여 초 안에 끝나고 비용은 사실상 무시할 수준이다. 이게 지금의 "느리고 비싸다"는 불만을 정면으로 해결한다.
- **선택(느림/비용 발생)**: 결과 화면에 "🔬 심층 조사" 버튼을 추가한다. 누르면 지금까지 써온 Sonnet 5 + web_search/web_fetch 병렬 회사별 조사(30일 범위, effort:low, 재시도 포함)가 돌아서 더 최신이고 근거(citations) 있는 결과로 교체된다. 사용자가 정말 필요할 때만 그 비용/시간을 쓰게 된다.

두 경로 모두 같은 응답 형식(`{analysis, sources}`)을 유지해서, 화면 렌더링 코드(`renderCompareResults()`)는 손대지 않고 그대로 재사용한다.

## 3. 서버 변경 (`server.js`)

### 3-1. 새 함수: `synthesizeFastComparison(apiKey, companies, ctxFor, activeTopics)` ✅

회사별로 이미 만들어져 있는 `ctxFor(co)` 컨텍스트(뉴스/판례/SEC 공시)를 전부 모아서 Haiku 한 번에 통짜로 넣고, 기존 응답 스키마 전체(`summary`, `isCompetitors`, `industryContext`, `focusAreas`, `companies{riskLevel,keyRisks,legalAdvantages,recentDevelopments,regulatoryExposure,citations}`, `industryTrends`, `comparativeVerdict`, `watchlist`)를 한 번에 생성하도록 요청한다. 도구 없음, `max_tokens: 3000` 정도, 온도 0.

`ctxFor()`에 뉴스 기사 링크가 빠져 있어서(현재 제목만 넣음) 이 경로에서 인용 가능한 citations를 만들려면 링크도 컨텍스트에 포함하도록 살짝 고친다 — 이건 심층 조사 경로에도 그대로 이득이라 부작용 없음.

### 3-2. `/api/compare-companies` 요청에 `deep` 플래그 추가 ✅

```js
const { companies = [], urls = [], topics = {}, sectors = {}, deep = false } = req.body;
```

- `deep`이 false(기본)면 3-1의 빠른 Haiku 통합 호출 한 번만 실행.
- `deep`이 true면 지금 코드 그대로(회사별 병렬 Sonnet 5 조사 + Haiku 종합) 실행.

### 3-3. Rate limit 버킷 분리 ✅

- `compare-companies` (빠른 경로): 시간당 30회로 상향 — 비용이 사실상 없으니 지금의 10회 제한은 과함.
- `compare-companies-deep` (심층 경로): 시간당 10회 유지 — 지금 걸려있는 비용 방어선 그대로.

## 4. 클라이언트 변경 (`index.html`)

### 4-1. `compareCompanies(deep = false)`로 함수 시그니처 변경 ✅

기존 호출부(`goToCompare()`, 새로고침 버튼)는 인자 없이 호출하므로 자동으로 빠른 경로(`deep: false`)를 탄다. 요청 바디에 `deep`을 실어 보낸다.

### 4-2. `renderCompareResults()`에 "🔬 심층 조사" 버튼 추가 ✅

빠른 결과에는 "⚡ 저장된 소스 기반 빠른 비교" 라벨 + "🔬 심층 조사" 버튼을 상단에 표시하고, 심층 조사 결과에는 그 대신 "🔬 심층 조사 — 실시간 웹 검색, 오늘 기준 최신 정보" 배지만 표시(버튼 없음 — 이미 심층 상태라 재클릭 유도 안 함). 6개 언어 i18n 키 모두 추가.

### 4-3. 로딩 상태 문구 차등 ✅

`compare-spin`의 텍스트를 `deep` 여부에 따라 동적으로 바꿔서, 빠른 경로는 기존 문구, 심층 조사는 "실시간 웹 검색 중 — 몇 분 걸릴 수 있습니다" 문구로 표시.

## 5. 영향받지 않는 부분

- Daily Briefing의 `companyIntel`(per-company Sonnet 5 조사)은 이번 범위에서 제외 — 사용자가 이번에 명시적으로 "Compare하는 속도"만 지적했다. Daily Briefing 쪽도 같은 불만이 나오면 동일한 패턴(빠른 기본 + 심층 옵션)을 그대로 적용할 수 있다.
- `researchCompareCompanyIntel`/`attemptResearchCompareCompanyIntel`(심층 조사용 함수)은 코드 그대로 유지, 호출 여부만 `deep` 플래그로 게이팅.

## 6. 검증 계획

- 로컬: syntax 체크, API 키 없는 상태에서 여전히 정상적으로 400/500 에러 나는지.
- 라이브: 빠른 경로 한 번 실행해서 몇 초 안에 끝나는지, 두 회사 다 결과가 채워지는지(웹 검색 자체가 없으니 "한 쪽만 실패" 문제가 원천적으로 없어야 함) 확인. 이건 비용이 거의 안 들어서 여러 번 테스트해도 괜찮음. 심층 조사 버튼은 사용자가 원할 때만 실제 비용 들여서 한 번 확인.

## 7. Self-review

- **베스트인가**: 지금까지 시도한 파라미터 튜닝은 전부 "매번 실시간 웹 검색"이라는 전제 자체는 안 건드렸다. 이 plan은 그 전제를 사용자 선택 사항으로 바꿔서, "느리고 비싸다"는 반복된 불만의 근본 원인을 없앤다.
- **빠진 거 없는지**: 응답 스키마를 두 경로 모두 동일하게 맞춰서 클라이언트 렌더링 코드 변경을 최소화했다. rate limit 버킷도 두 경로의 실제 비용 차이에 맞게 분리했다.
- **오버한 거 없는지**: Daily Briefing 쪽 companyIntel은 이번엔 손대지 않는다(사용자가 지적한 범위 밖). 캐싱 레이어는 어제 사용자가 "하루 1~2번"이라고 답해서 필요 없다고 판단한 것 그대로 유지 — 다시 추가하지 않는다.
- **테스트 충분한지**: 빠른 경로는 비용이 거의 없어 여러 번 검증 가능. 심층 경로는 기존 코드를 그대로 재사용하는 것이라 오늘까지 이미 여러 번 검증된 상태 — 새로 테스트할 건 "게이팅이 제대로 되는지"뿐.

확신도: 92%. 빠른 경로의 결과 품질(Haiku가 이미 모은 뉴스/판례/SEC 데이터만으로 얼마나 쓸모있는 비교를 만드는지)은 실제 라이브로 봐야 확실해지는 부분이라 완전히 100%는 아니다.
