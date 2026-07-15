# Plan — Daily Briefing 콘텐츠 선정 기준을 소스 언어가 아닌 전략적 중요도로

상태: 원인 확정(find-cause), 인터뷰 불필요, 구현 대기.

## 1. 원인 (find-cause에서 확정)

운영 사이트에 직접 `/api/search-news`를 호출해 확인했다.

- `law360.com` — 기사 0개(시도한 모든 RSS 경로에서 404, 공개 RSS 자체가
  없어 보임).
- `courthousenews.com` — 기사 0개(모든 경로에서 403, 봇 차단).
- `legaltimes.co.kr`(오늘 사용자가 추가한 한국어 소스) — 기사 8개.

즉 사용자가 등록한 세 소스를 합쳐 `generateBriefing()`이 실제로 받는
기사 재료가 처음부터 100% 한국어였다. 이건 오늘 한국어 소스를 추가하며
생긴 문제가 아니라, 기본 소스 두 개가 원래부터 기사를 못 가져오던
상태였던 것이 이번에 처음 드러난 것이다 — 기본 소스가 죽어 있을 땐
`hasArticles`가 false라 AI가 실제 기사 없이 "그럴듯한 브리핑"을 생성하는
분기(`index.html:951`의 `:` 뒤)를 타고 있었고, 사용자는 그 결과물이
어느 소스 기반인지 체감할 수 없었다. 한국어 소스가 실제로 기사를
반환하기 시작하면서 처음으로 `hasArticles`가 true가 됐고, AI가 프롬프트
지시("Using ONLY the articles listed above") 그대로 있는 재료를 전부
썼을 뿐인데 그 재료가 우연히 100% 한국어였다.

## 2. 이번 plan의 범위

사용자 요청의 핵심은 "소스 언어와 무관하게, 선택된 Legal Topics·Business
Sectors 기준으로 가장 전략적으로 중요한 내용을 고르도록" 프롬프트를
고치는 것이다. `law360.com`/`courthousenews.com`이 애초에 기사를 못
가져오는 문제는 **별도 이슈로 분리**한다 — courthousenews는 명시적 봇
차단(403)이라 우회 자체가 이 앱의 단순 RSS 스크래핑 방식으로는 어렵고,
law360은 유료 구독형 매체라 공개 RSS가 아예 없을 가능성이 높다. 둘 다
"프롬프트를 고쳐서 해결되는 문제"가 아니라 소스 자체의 접근성 문제라,
지금 요청 범위 밖이라고 판단했다(5번 리스크에 남겨둔다).

## 3. 구현 — 프롬프트에 선정 기준 명시

`generateBriefing()`(`index.html:906`)의 시스템 프롬프트
(`index.html:948-954`)에서, 지금은 "Using ONLY the articles listed
above, write 2-3 bullet summaries per section"라고만 돼 있어 기사가
어느 소스에서 왔는지·몇 개씩 왔는지에 따라 그대로 편향된다. 여기에
선정 기준을 명시하는 문장을 추가한다.

`index.html:951`을 아래로 교체.

```js
${hasArticles ? articleCtx + `\n\nUsing ONLY the articles listed above, select and summarize the ones most strategically significant for the selected topics${activeSectors.length ? ' and business sectors' : ''} — prioritize by likely business/legal impact (deal risk, regulatory exposure, competitive positioning, revenue impact), never by which source domain or language an article happens to come from. Write 2-3 bullet summaries per section from that selection. Each bullet MUST reference one article by its number using "ref": <integer>. Do NOT copy URLs into the JSON.${foreignInstr}` : 'No live articles available — generate a plausible briefing based on current legal trends. Omit "ref" from bullets.'}
```

바뀌는 부분은 "Using ONLY the articles listed above," 바로 뒤에
"select and summarize the ones most strategically significant...
never by which source domain or language" 구절을 끼워 넣은 것뿐이다 —
나머지(참조 번호 규칙, JSON 스키마, foreignInstr)는 그대로 둔다.

이 지시는 3-1(foreignInstr)과 자연스럽게 공존한다 — foreignInstr는
"어느 도메인이 비영어인지 판별해서 그 언어로 별도 요약 섹션을 추가하라"는
지시이고, 이번에 추가하는 지시는 "본문 섹션(sections)에 어떤 기사를
고를지는 언어가 아니라 전략적 중요도로 판단하라"는 지시다 — 서로 다른
목적의 두 지시가 같은 프롬프트 안에서 충돌 없이 공존한다.

## 4. 테스트 plan

- 지금과 같은 상태(한국어 소스만 실제로 기사를 반환하는 상태)에서
  Renew — 본문 섹션들이 여전히 한국어 소스의 기사를 인용하는 건 당연하다
  (재료가 그것뿐이므로). 다만 각 섹션의 불릿이 "가장 전략적으로 중요해
  보이는" 기사 위주로 골라졌는지, 8개 기사를 무작위 순서로 나열하는 게
  아니라 topic/sector 관련성이 높은 것부터 우선했는지 육안으로 확인한다
  (완전 자동화된 검증은 불가능한 영역 — 6번 참고).
- 영어 소스가 하나라도 실제로 기사를 반환하는 상태를 임시로 만들어(예:
  RSS가 살아있는 다른 영어 법률 뉴스 도메인을 하나 추가) 한국어·영어
  기사가 섞인 상태에서 Renew — 특정 언어 쪽으로 쏠리지 않고 topic 관련성
  기준으로 양쪽 다 인용되는지 확인.
- 기존 회귀 확인 — `foreignSummaries` 섹션(오늘 오전에 추가한 기능)이
  이번 프롬프트 수정 이후에도 여전히 정상적으로 붙는지, 인용 번호(`ref`)
  체계가 깨지지 않는지 확인.

## 5. 리스크 / 엣지케이스

`law360.com`/`courthousenews.com`이 기사를 못 가져오는 문제는 이번
plan에서 고치지 않는다 — 별도로 원하시면 (a) 실제 RSS가 살아있는 다른
영어 법률 뉴스 도메인으로 기본 소스를 교체하거나, (b) courthousenews의
봇 차단을 우회하는 방법(예: 다른 User-Agent, 헤더 조정)을 시도해볼 수
있지만 안정적으로 계속 작동한다는 보장이 없어서 후속 작업으로 분리해
둔다.

프롬프트 지시만으로 "전략적 중요도"를 판단하게 하는 건 AI의 정성적
판단에 의존한다 — 이 앱 전체가 이미 topic/bullet 생성을 AI 판단에
맡기는 구조라 일관된 접근이지만, 결과가 매번 100% 예측 가능하진 않다는
점은 감안해야 한다.

## 6. Self-review

**베스트 plan인지** — 프롬프트 문장 하나만 고치는 최소 변경이다. 기사
fetch 로직이나 데이터 구조는 그대로 두고, "이미 모은 재료 중에서 무엇을
쓸지" 판단 기준만 AI에게 명시적으로 알려주는 방식이라 이 앱의 기존
"AI에게 판단을 맡긴다" 철학과 일관된다.

**빠진 게 있는지** — 근본 원인(기본 소스 두 개가 애초에 죽어있음)을
찾아서 사용자에게 투명하게 알렸고, 이번 plan에서 고치는 부분과 안 고치는
부분을 명확히 나눴다.

**오버한 게 있는지** — 죽어있는 기본 소스를 고치는 것, RSS 우회 로직을
새로 만드는 것 모두 이번 요청 범위(콘텐츠 선정 기준)를 넘어서는 별도
작업이라 포함하지 않았다.

**테스트 충분한지** — 현재 상태(한국어만 실제 기사)와 다국어 혼합 상태
둘 다 커버했고, 오늘 추가한 foreignSummaries 기능과의 회귀도 포함했다.
다만 "전략적 중요도"라는 정성적 기준 자체는 자동화 테스트로 검증 불가능한
영역이라 육안 검수가 필요하다는 한계를 그대로 남겨둔다.

## 7. 사용자 결정 필요 항목

없음.
