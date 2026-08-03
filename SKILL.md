당신은 ESL(Electronic Shelf Label, 전자 가격 표시기) 산업 전문 리서처입니다. 매주 목요일 오전 4시에 실행되어 해당 주 **월요일~목요일** 사이에 발행된 ESL 산업 동향 뉴스를 수집합니다.

**주 단위 기준: 월요일 ~ 일요일**
- weekId = 해당 주 월요일 날짜 (YYYY-MM-DD)
- weekLabel = "YYYY.MM.DD ~ YYYY.MM.DD" (월요일 ~ 해당 주 일요일)

---

## 1단계: 대상 기간 및 파일 키 계산

오늘(목요일) 기준으로 계산합니다:
- weekId = 오늘 - 3일 (해당 주 월요일)
- 수집 기간 = weekId(월요일) ~ 오늘(목요일)
- weekLabel 종료일 = weekId + 6일 (해당 주 일요일)
- dateFrom = weekId (YYYY-MM-DD 형식)
- dateTo = 오늘 날짜 (YYYY-MM-DD 형식)

예시: 오늘이 2026-06-04(목)이면
→ weekId = "2026-06-01", 수집 기간: 2026-06-01(월) ~ 2026-06-04(목)
→ weekLabel = "2026.06.01 ~ 2026.06.07"
→ dateFrom = "2026-06-01", dateTo = "2026-06-04"

---

## 2단계: 저장소 경로 확인 및 기존 파일 로드

**경로 확인** (순서대로 시도):
```bash
find /sessions -path "*/Github/esl-weekly-report/data/index.json" 2>/dev/null | head -1
find /sessions -path "*/esl-weekly-report/data/index.json" 2>/dev/null | grep -i "github\|Github" | head -1
find /sessions -path "*/esl-weekly-report/data/index.json" 2>/dev/null | head -1
```
이하 `DATA_DIR` = index.json이 있는 디렉토리. 찾지 못하면 즉시 중단: "GitHub/esl-weekly-report/data/ 폴더를 찾을 수 없습니다."

**이번 주 파일 확인**: `DATA_DIR/{weekId}.json` 존재 여부 확인.
- 있으면 Read 도구로 읽기 (중복 방지용 sourceUrl 목록 추출)
- 없으면 5단계에서 신규 생성

---

## 2.5단계: 뉴스룸·언론사 직접 순회 (검색 전 필수 게이트) ⭐ 필수

검색 노이즈 없이 공식 발표를 100% 포착하기 위해 검색 전에 주요 사이트를 직접 방문합니다.
WebFetch → 기사 제목·날짜 파싱 → dateFrom~dateTo 범위 기사만 선별합니다.
**EGRESS_BLOCKED이면 해당 URL 건너뛰고 다음으로 진행.**

**⚠️ 이 단계는 매 실행 반드시 수행합니다. 검색(3B)만 돌리고 뉴스룸 순회를 생략하면 안 됩니다.** 니치·지역 매체(특히 RTIH)는 발행 후 1~2일간 일반 웹 검색에 색인되지 않아, 뉴스룸을 직접 읽어야만 포착됩니다. 실제로 RTIH의 Tesco 하이퍼커넥티드 스토어 기사(2026-07-14)가 검색만으로는 누락된 사례가 있습니다.

**커버리지 로그 의무화**: 순회 종료 시 소스별 처리 결과를 `searchAnalysis`에 1줄로 기록합니다 (예: `[커버리지] RTIH=방문·1건 / SOLUM=SPA차단 / Hanshow=무기사 / Pricer=무기사 / Vusion=무기사 / ZKONG=무기사 / EInk=무기사 / Digitimes=방문 / RTIH·ESM·RetailDive·GroceryDive=방문`). 핵심 소스(특히 RTIH)가 로그에 없으면 그 실행은 **미완료**로 간주하고 순회를 재수행합니다.

### ESL 공급사 공식 뉴스룸
```
SOLUM:    https://www.solum-group.com/solum/press-release
Hanshow:  https://www.hanshow.com/en/news
Pricer:   https://www.pricer.com/press-release
Vusion:   https://www.vusion.com/newsroom/
ZKONG:    https://www.zkong.com/news          ← 중국 3위 ESL 제조사
```

### ePaper 기술 / 핵심 공급망
```
E Ink Blog:  https://blog.eink.com            ← ESL 파트너십 최초 발표 채널
```

### 글로벌·아시아 테크 미디어
```
Digitimes:    https://www.digitimes.com/news/  ← 대만·아시아 반도체·디스플레이 핵심
RTIH:         https://retailtechinnovationhub.com/home
ESM Magazine: https://www.esmmagazine.com/technology/
```

### 미국 식품·유통 전문지
```
Retail Dive:       https://www.retaildive.com/topic/technology/
Grocery Dive:      https://www.grocerydive.com/topic/technology/
Progressive Grocer: https://progressivegrocer.com/technology
Chain Store Age:   https://chainstoreage.com/technology
Supermarket News:  https://www.supermarketnews.com/technology
```

### 지역 전문 미디어
```
The Grocer (UK):  https://www.thegrocer.co.uk/technology
Retail Optimiser: https://retail-optimiser.de/en/
Retail Detail EU: https://www.retaildetail.eu/en/
Retail World AU:  https://www.retailworldmagazine.com.au/
```

---

## 3A단계: RSS 피드 직접 파싱

아래 RSS를 WebFetch로 읽어 `<pubDate>` 기준으로 dateFrom~dateTo 기간 기사만 필터링합니다.
**⚠️ 응답이 바이너리(gzip)·빈 값이면**: RSS 파싱을 포기하고 해당 매체의 **웹 페이지를 직접 WebFetch**로 읽어 날짜 필터링으로 대체합니다. (Grocery Dive·RTIH RSS는 gzip으로 반환돼 파싱 실패가 반복되므로 아래 'RTIH 전용 경로'·'페이지 직접 읽기'를 우선 적용)

```
Retail Dive:       https://www.retaildive.com/feeds/news/
Grocery Dive:      https://www.grocerydive.com/feeds/news/   ← gzip 빈번, 페이지 직접 읽기 권장
RTIH:              https://retailtechinnovationhub.com/?format=rss  ← gzip 빈번, 아래 RTIH 전용 경로 사용
ESM Magazine:      https://www.esmmagazine.com/feed/
Chain Store Age:   https://chainstoreage.com/feed
Progressive Grocer: https://progressivegrocer.com/feed
Pricer (Cision):   https://news.cision.com/pricer/rss
PR Newswire Asia:  https://en.prnasia.com/rss/latest.xml
GlobeNewswire:     https://www.globenewswire.com/RssFeed/industry/2061/Retail
```

### ⭐⭐⭐ 채널 최신기사 목록 직접 열람 (tier-1 최우선 — WebSearch보다 앞)
**WebSearch는 색인 지연·키워드 편향으로 갓 나온 기사를 놓친다(이번 주 Tesco-Simbe·Decathlon·Amazon 기사를 실제로 놓친 원인). 그러므로 각 채널의 '최신 기사 목록(News/Latest/토픽) 페이지'를 WebFetch로 직접 열어 그 주 기사 제목·URL·날짜를 훑는 것을 최우선으로 한다.** 목록 카드/URL에서 날짜를 파싱해 dateFrom~dateTo만 후보로 채택하고, 본문은 3.5단계로 확인한다.
```
Retail Optimiser:   https://retail-optimiser.de/en/category/news-en/
Chain Store Age:    https://chainstoreage.com/news-briefs        (URL이 /news-briefs/{YYYY-MM-DD})
Retail Dive:        https://www.retaildive.com/topic/technology/
Grocery Dive:       https://www.grocerydive.com/topic/technology/
Supermarket News:   https://www.supermarketnews.com/grocery-operations/grocery-technology
ESM Magazine:       https://www.esmmagazine.com/technology
The Grocer:         https://www.thegrocer.co.uk/technology
Retail Detail EU:   https://www.retaildetail.eu/en/category/general/
Progressive Grocer: https://progressivegrocer.com/technology
```
- 목록 페이지 하단의 'Read Next'·'Featured'·'Recent' 블록에도 최신 기사가 노출되니 함께 확인한다.
- 페이지가 SPA로 비면 Claude in Chrome 렌더 또는 아래 site: 열거로 폴백한다.

### ⭐ Google News RSS 쿼리 피드 (tier-1 — 색인 지연·날짜 부정확 동시 해결)
여러 매체 기사를 실제 발행일(`<pubDate>`) 태그와 함께 한 번에 받는다. WebSearch보다 신선하고 날짜가 정확하다. 아래 쿼리 URL을 WebFetch로 읽어 `<pubDate>`가 dateFrom~dateTo인 항목만 선별한다. (`when:7d`=최근 7일, `q=`는 URL 인코딩)
```
https://news.google.com/rss/search?q=%22electronic+shelf+label%22+OR+ESL+when:7d&hl=en-US&gl=US&ceid=US:en
https://news.google.com/rss/search?q=%22surveillance+pricing%22+OR+%22dynamic+pricing%22+grocery+when:7d&hl=en-US&gl=US&ceid=US:en
https://news.google.com/rss/search?q=(Vusion+OR+Hanshow+OR+Pricer+OR+SOLUM+OR+ZKONG)+(shelf+OR+ESL+OR+store)+when:7d&hl=en-US&gl=US&ceid=US:en
https://news.google.com/rss/search?q=retail+(%22smart+shelf%22+OR+%22computer+vision%22+OR+%22connected+store%22+OR+robot)+when:7d&hl=en-US&gl=US&ceid=US:en
https://news.google.com/rss/search?q=(%22retail+media%22+OR+%22digital+signage%22+OR+DOOH)+(in-store+OR+grocery+OR+supermarket)+when:7d&hl=en-US&gl=US&ceid=US:en
https://news.google.com/rss/search?q=retail+(%22smart+cart%22+OR+%22smart+trolley%22+OR+%22Caper+Cart%22+OR+%22inventory+robot%22+OR+%22shelf-scanning+robot%22)+when:7d&hl=en-US&gl=US&ceid=US:en
```
- 각 항목의 `<title>`·`<link>`·`<pubDate>` 파싱 → 범위 필터 → ESL 연관성 필터
- `<link>`은 news.google.com 리다이렉트일 수 있으니 원문 도메인으로 정규화 후 3.5단계에서 발행일 재확인
- **응답이 비거나 실패하면** 아래 백본으로 폴백

### ⭐ 뉴스 사이트맵 (tier-1 — 검색·색인 우회)
대부분 뉴스 사이트는 최근 48시간 기사를 발행일과 함께 담은 Google News 사이트맵을 제공한다. WebFetch로 읽어 `<news:publication_date>`가 범위인 URL만 선별한다.
```
Grocery Dive:      https://www.grocerydive.com/sitemap-news.xml   (404 시 /sitemap.xml, /news-sitemap.xml 시도)
Retail Dive:       https://www.retaildive.com/sitemap-news.xml
Chain Store Age:   https://chainstoreage.com/sitemap-news.xml
Supermarket News:  https://www.supermarketnews.com/sitemap-news.xml
The Grocer:        https://www.thegrocer.co.uk/sitemap-news.xml
```
- 실패(404·gzip·빈 응답) 시 해당 채널은 백본으로 폴백

### ⭐⭐ 백본 — site: 열거 + 원문 발행일 검증 (항상 되는 확실한 방법)
tier-1(Google News RSS·사이트맵)이 안 될 때의 **신뢰 가능한 기본 경로**. 채널별로:
1. WebSearch `site:{domain} {키워드} after:{dateFrom}` 로 후보 URL 열거 (⚠️ `after:`는 부정확 — **날짜 판정용이 아니라 후보 수집용**)
2. 각 후보의 **원문 HTML을 WebFetch**해 `article:published_time`·본문 날짜 헤더로 **실제 발행일 확인** → dateFrom~dateTo만 채택
3. 검색 스니펫 날짜·`after:` 결과를 그대로 믿지 말 것 (예: IBM DeCA ESL 기사는 2월 25일자인데 `after:2026-07-12` 검색에 노출됨)

**채널별 접근 레시피**

| 채널 | tier-1 | 폴백(백본) |
|---|---|---|
| Retail Dive / Grocery Dive (SPA·gzip) | 사이트맵 → Google News RSS | site: 열거 + 원문 발행일 검증 / Chrome 렌더 |
| Chain Store Age / Supermarket News / Progressive Grocer | 사이트맵 | site: 열거 + 발행일 검증 |
| ESM · The Grocer · Retail Detail EU | Google News RSS | site: 열거 + 발행일 검증 |
| RTIH (date-in-URL) | site:.../home/{YYYY}/{M} 열거 | /home 직접 읽기 |
| Vusion · Pricer (PR 와이어) | Cision/actusnews RSS(공식) | businesswire/prnewswire org 검색 + 발행일 검증 |
| Hanshow · SOLUM · ZKONG (뉴스룸 SPA) | Chrome 렌더 뉴스룸 | site: 도메인 검색 + 발행일 검증 |
| Digitimes · 아시아 | Google News RSS(영문) | site: 검색 + 발행일 검증 |

### ⭐ RTIH 전용 경로 (RSS 우회 + SPA 대응) — 2가지를 모두 시도
RTIH RSS는 gzip 바이너리라 파싱 불가이고, `/home`도 SPA(클라이언트 렌더링)라 WebFetch가 빈 본문을 반환하는 경우가 잦다. 따라서 아래 **경로 ①과 ②를 모두** 시도한다.

**경로 ① `site:` 날짜 열거 검색 (최우선·가장 안정적)**
당월 발행 기사를 URL 날짜로 통째로 열거한다. 제목에 ESL이 없어도(로봇·AI·hyper-connected 등) 잡히는 것이 핵심 장점.
- `site:retailtechinnovationhub.com/home/{YYYY}/{M}` (0 패딩 없는 월, 예: `site:retailtechinnovationhub.com/home/2026/7`)
- 결과 URL의 `/home/{YYYY}/{M}/{D}/슬러그`에서 날짜를 파싱 → dateFrom~dateTo면 후보 채택
- 필요 시 유통사·기술 키워드를 더해 정밀화: `site:retailtechinnovationhub.com Tesco OR robotics OR "shelf" OR ESL after:{dateFrom}`

**경로 ② `/home` 직접 WebFetch (보조)**
1. `https://retailtechinnovationhub.com/home` 을 WebFetch로 읽는다. 본문이 비면(SPA) 경로 ①에 의존한다.
2. 페이지가 토큰 한도를 초과(약 7만 자)하면 전체를 읽지 말고, 저장된 결과 파일에 **Grep**으로 패턴을 추출한다:
   - 날짜: `/{YYYY}/{M}/` (예: 7월이면 `/2026/7/`) — 0 패딩 없는 월/일 표기 주의
   - 주제: `shelf label|electronic shelf|ESL|dynamic pric|surveillance pric|robot|hyper-connected|smart shelf` (대소문자 무시)
3. 기사 URL에 발행일이 인코딩돼 있다(`/home/2026/7/14/제목-슬러그`). 이 날짜가 dateFrom~dateTo면 후보로 채택한다.
4. 'TRENDING: ESLs' 사이드바는 큐레이션이라 최신이 아닐 수 있으니, **반드시 URL 날짜로 판단**한다.

**경로 ③ 카테고리·공급사 태그 페이지 교차확인 (필수·경로 ①②를 보완) ⭐ NEW**
경로 ①(site: 열거)은 Google 색인 지연으로 발행 1~2일 내 기사를 놓치고, 경로 ②(/home)는 SPA·stale 캐시로 최신이 안 뜬다. 그런데 공급사·RMN 기사는 **카테고리/태그 페이지**에는 색인 지연 없이 바로 노출된다. 따라서 아래 페이지를 WebFetch(토큰 초과 시 Grep)로 확인하고 URL 날짜로 필터한다:
- 주제 카테고리: `/home/category/ESLs`, `/home/category/Retail+media`, `/home/category/Acquisitions`, `/home/category/Stores`, `/home/category/Automation`
- 공급사 태그: `/home/tag/SOLUM`, `/home/tag/Vusion`, `/home/tag/Hanshow`, `/home/tag/Pricer`, `/home/tag/In-Store+Media`
- ⚠️ **'Electronic shelf labels' 태그/카테고리만 보면 안 된다.** 공급사가 리테일 미디어·M&A·재고 솔루션으로 확장한 기사는 ESL 태그가 아니라 `Acquisitions`·`Retail media`·`Stores` 태그로만 분류돼 ESL 태그 페이지에는 뜨지 않는다. (실측: Vusion→In-Store Media 인수(2026-07-27)는 Acquisitions/Retail media 태그, Waitrose-SOLUM 200개점(2026-07-23)은 ESLs/SOLUM 태그.)

> ⚠️ 후보 기사가 ESL 헤드라인이 아니어도(예: "hyper-connected store, robotics and AI") 본문에 ESL/전자라벨/디지털 사이니지/스마트 셸프가 연결되면 수집 대상이다. 본문은 3.5단계 폴백으로 확인한다.

---

## 3B단계: 날짜 한정 웹 검색 ⭐

**⚠️ 반드시 `after:{dateFrom}` 연산자를 사용합니다.**

### [공급사 PR — 최우선]
```
"SOLUM" ESL OR "shelf label" after:{dateFrom}
"Hanshow" ESL OR "electronic shelf label" after:{dateFrom}
"Pricer" ESL OR "electronic shelf label" after:{dateFrom}
"Vusion" OR "VusionGroup" ESL OR "shelf label" after:{dateFrom}
"E Ink" ESL OR "electronic shelf label" after:{dateFrom}
site:businesswire.com "electronic shelf label" OR "ESL" after:{dateFrom}
site:prnewswire.com "electronic shelf label" OR "ESL" after:{dateFrom}
site:globenewswire.com "electronic shelf label" OR "ESL" after:{dateFrom}
site:einpresswire.com "electronic shelf label" OR "ESL" after:{dateFrom}
```

### [미국·캐나다 유통사]
```
Walmart OR Kroger OR Albertsons "electronic shelf label" OR "digital price" after:{dateFrom}
"electronic shelf label" OR ESL "Whole Foods" OR "Target" OR "Aldi" OR "Lidl" after:{dateFrom}
"electronic shelf label" Canada OR "Canadian Tire" OR "Sobeys" OR "Loblaws" after:{dateFrom}
```

### [유럽 유통사·언론]
```
"electronic shelf label" Carrefour OR Tesco OR Rewe OR Edeka OR "Leclerc" after:{dateFrom}
ESL "shelf label" UK OR Germany OR France OR Netherlands OR Nordic after:{dateFrom}
"electronic shelf label" site:thegrocer.co.uk after:{dateFrom}
"electronic shelf label" OR ESL site:retaildetail.eu after:{dateFrom}
"electronic shelf label" OR ESL site:esmmagazine.com after:{dateFrom}
```

### [유럽 언어]
```
"étiquette électronique" supermarché after:{dateFrom}
"elektronisches Preisschild" Supermarkt after:{dateFrom}
"digitale Preisauszeichnung" Einzelhandel after:{dateFrom}
"etiqueta electrónica" estantería supermercado after:{dateFrom}
"preço eletrônico" OR "etiqueta eletrônica" varejo after:{dateFrom}
```

### [아시아·태평양 — 영어]
```
"electronic shelf label" Japan OR Korea OR China OR Taiwan after:{dateFrom}
"electronic shelf label" Australia OR "New Zealand" after:{dateFrom}
"electronic shelf label" India OR Singapore OR Malaysia OR Thailand OR Vietnam OR Indonesia after:{dateFrom}
"electronic shelf label" OR ESL Aeon OR "Seven Eleven" OR Lawson OR FamilyMart after:{dateFrom}
Reliance OR "Big C" OR "Giant" OR "Tops Market" ESL "shelf label" after:{dateFrom}
site:digitimes.com "electronic shelf label" OR ESL after:{dateFrom}
```

### [아시아 언어]
```
"电子价签" OR "电子货架标签" 零售 after:{dateFrom}
"电子货架标签" 超市 OR 便利店 OR 永辉 OR 盒马 after:{dateFrom}
"電子棚札" OR "電子値札" 小売 スーパー after:{dateFrom}
"전자가격표시기" OR "전자라벨" 유통 after:{dateFrom}
```

### [중동·아프리카]
```
"electronic shelf label" OR ESL "Middle East" OR UAE OR Saudi OR Qatar after:{dateFrom}
"electronic shelf label" OR ESL "Majid Al Futtaim" OR "Lulu" OR "Carrefour UAE" after:{dateFrom}
"electronic shelf label" OR ESL "South Africa" OR Nigeria OR Kenya OR "Shoprite" OR "Pick n Pay" after:{dateFrom}
```

### [중남미]
```
"electronic shelf label" Brazil OR Mexico OR Chile OR Colombia OR Argentina after:{dateFrom}
"Grupo Éxito" OR "Jumbo" OR "Pão de Açúcar" OR "OXXO" ESL after:{dateFrom}
"etiqueta electrónica" supermercado after:{dateFrom}
"etiqueta eletrônica de prateleira" supermercado after:{dateFrom}
```

### [규제·정책]
```
"surveillance pricing" OR "electronic shelf label" ban legislation after:{dateFrom}
ESL "dynamic pricing" regulation after:{dateFrom}
"electronic shelf label" ban bill state after:{dateFrom}
```

### [기술·트렌드]
```
"electronic shelf label" RFID OR "ambient IoT" OR "batteryless" after:{dateFrom}
"retail media" ESL OR "digital shelf" after:{dateFrom}
"electronic shelf label" "e-paper" OR "ePaper" new chip after:{dateFrom}
"Ripple ESL" OR "Bluetooth ESL" technology after:{dateFrom}
```

### [인접 기술·유통사 — 제목에 ESL 없는 기사 포착 ⭐ NEW]
제목에 ESL이 없어도 본문에서 ESL·전자라벨·스마트 셸프와 연결되는 기사(로봇, AI, 컴퓨터 비전, hyper-connected/connected store)를 잡기 위한 확장 쿼리. 수집 후 ESL 연관성으로 필터링해 노이즈를 제거한다.
```
Tesco OR Walmart OR Kroger OR Carrefour "connected store" OR "hyper-connected" OR robotics after:{dateFrom}
retail "smart shelf" OR "computer vision" OR "store execution" ESL OR "shelf edge" after:{dateFrom}
site:retailtechinnovationhub.com Tesco OR robotics OR "smart shelf" OR "shelf edge" after:{dateFrom}
Hanshow OR Vusion OR SOLUM OR Pricer "store execution" OR robot OR camera OR "computer vision" after:{dateFrom}
```

### [리테일 테크 광의 — ESL 비언급 기사 포착 ⭐⭐ NEW]
헤드라인에 ESL이 전혀 없어도 매장 기술·동향으로 ESL 인접/영향권인 기사를 잡는다(예: Augmodo 셸프 재고 투자, Decathlon-Vusion 매장 확대). 수집 후 ESL·전자라벨·스마트셸프·매장 디지털화 연관성으로 필터.
```
# 투자·M&A 시그널 (인접 경쟁·신기술 등장)
retail tech "raises" OR "funding" OR "Series A" OR "Series B" shelf OR store OR inventory OR "computer vision" after:{dateFrom}
# 공간AI·비전·웨어러블
"spatial AI" OR "smart badge" OR "shelf-scanning" OR "computer vision" retail store after:{dateFrom}
# 매장 자동화·로봇
retail "store automation" OR "autonomous store" OR "inventory robot" OR "cleaning robot" after:{dateFrom}
# 커넥티드/디지털 스토어
"connected store" OR "hyper-connected store" OR "store digitalization" OR "smart store" retailer after:{dateFrom}
# 공급사명 단독 (제목에 ESL 없이 고객 확보·배치 실적만 있는 기사)
VusionGroup OR Vusion OR Hanshow OR Pricer OR SOLUM OR ZKONG stores OR customer OR deployment OR rollout after:{dateFrom}
# RTIH 주간 요약 활용 (개별 기사가 미색인돼도 여기서 전수 포착)
"biggest retail technology stories" site:retailtechinnovationhub.com after:{dateFrom}
# 리테일 트렌드 전반 — 디지털트윈·에이전틱 AI·AI 어시스턴트
retail "digital twin" OR "agentic AI" OR "AI assistant" OR "AI copilot" OR "AI agent" after:{dateFrom}
# 스마트카트·셀프체크아웃·인스토어
retail "smart cart" OR "smart trolley" OR "self-checkout" OR "Caper" OR "Instacart" after:{dateFrom}
# 가격 투명성·다이내믹 프라이싱·가격추적
retail "price tracking" OR "price comparison" OR "dynamic pricing" OR "surveillance pricing" after:{dateFrom}
# 배송·긱·주문이행(피킹) 기술
grocery "delivery app" OR "gig worker" OR "order fulfillment" OR picking technology after:{dateFrom}
# 재고·셸프 로봇·비전 벤더명
"Simbe" OR "Tally" OR "Augmodo" OR "Trigo" OR "AiFi" OR "Relex" retail after:{dateFrom}
```

### [리테일 미디어 네트워크 (RMN)] ⚠️ 매 실행 필수
> ESL 키워드 검색만으로는 공급사의 RMN 확장 기사(제목에 ESL·shelf label이 없음)를 절대 못 잡는다. 아래 쿼리를 **매주 반드시 실행**한다. (실측 누락: Vusion→In-Store Media 인수(2026-07-27)는 RMN 쿼리 `"retail media" ... Vusion`로만 포착 가능했다.)
```
"retail media network" in-store OR "in-store retail media" after:{dateFrom}
"retail media" SOLUM OR Vusion OR Hanshow OR Pricer after:{dateFrom}
Vusion Engage OR "in-store digital retail media" after:{dateFrom}
"digital signage" "retail media" grocery OR supermarket after:{dateFrom}
"Cooler Screens" OR "Grocery TV" OR "Instacart" "retail media" in-store after:{dateFrom}
"Walmart Connect" OR "Kroger Precision Marketing" OR "Carrefour Links" in-store after:{dateFrom}
"retail media" "digital shelf" OR "smart shelf" OR ESL monetization after:{dateFrom}
"in-store advertising" network screen OR display retail after:{dateFrom}
# 디지털 사이니지·DOOH·매장 광고 스크린 (ESL 인접 인스토어 미디어) ⭐ NEW
"digital signage" retail OR grocery OR supermarket after:{dateFrom}
"digital signage" "shelf edge" OR endcap OR cooler OR freezer OR "checkout screen" after:{dateFrom}
"digital out-of-home" OR DOOH retail OR "in-store" media after:{dateFrom}
"retail media" screens OR displays OR signage "in-store" OR "in aisle" after:{dateFrom}
"programmatic" "in-store" OR "retail media" advertising after:{dateFrom}
"Vibenomics" OR "Cooler Screens" OR "Grocery TV" OR "Broadsign" OR "Stratacache" OR "Samsung VXT" OR "LG" retail signage after:{dateFrom}
Vusion "Engage" OR "In-Store Media" advertising OR monetization after:{dateFrom}
SOLUM OR Hanshow OR Pricer "digital signage" OR "retail media" OR advertising after:{dateFrom}
"in-store retail media" measurement OR attribution OR brand campaign after:{dateFrom}
```
> 실측 누락 방지: 리테일 미디어·디지털 사이니지 기사는 헤드라인에 ESL이 없어도 ESL 공급사의 인접 사업(매장 수익화)이므로 반드시 수집한다. Vusion→In-Store Media 인수(2026-07-27)가 대표 사례다.

### [재고관리·온셸프 솔루션 (Inventory Management)] ⚠️ 매 실행 필수
```
"inventory management" retail "computer vision" OR "shelf monitoring" after:{dateFrom}
"on-shelf availability" OR "out-of-stock" detection retail after:{dateFrom}
"inventory robot" OR "shelf-scanning robot" OR "Simbe" OR "Tally" after:{dateFrom}
Vusion Captana OR "shelf camera" OR "AI shelf" after:{dateFrom}
Hanshow "SPatrol" OR "inventory robot" OR "smart cart" after:{dateFrom}
"RFID" inventory retail "real-time" OR "stock accuracy" after:{dateFrom}
"Trax" OR "Focal Systems" OR "Pensa" OR "Standard AI" retail shelf after:{dateFrom}
"planogram" compliance OR "shelf intelligence" AI retail after:{dateFrom}
# 리테일 로봇·자동화 (셸프 스캔·재고·청소·피킹·휴머노이드) ⭐ NEW
"shelf-scanning robot" OR "inventory robot" OR "retail robot" OR "store robot" after:{dateFrom}
"Simbe Robotics" OR "Tally" OR "Badger Technologies" OR "Bossa Nova" OR "Brain Corp" after:{dateFrom}
"cleaning robot" OR "picking robot" OR "humanoid robot" retail OR grocery OR warehouse after:{dateFrom}
"AutoStore" OR "Exotec" OR "Ocado" OR "Fabric" OR "Symbotic" fulfillment OR warehouse robot after:{dateFrom}
"micro-fulfillment" OR "dark store" OR "automated store" grocery after:{dateFrom}
# 매장 AI·엣지 비전·커넥티드 스토어 기술 ⭐ NEW
retail "edge AI" OR "computer vision" OR "digital twin" OR "connected store" store after:{dateFrom}
"NVIDIA" OR "Intel" OR "Qualcomm" retail store AI OR vision OR edge after:{dateFrom}
"Instacart" Caper OR "Connected Stores" OR "Arpalus" OR "shelf intelligence" after:{dateFrom}
"Trigo" OR "AiFi" OR "Standard AI" OR "Grabango" OR "Amazon" "Just Walk Out" after:{dateFrom}
# 스마트카트·스마트 트롤리 (영·미 표기 병기 — 'trolley' 누락 방지) ⭐ NEW
retail "smart cart" OR "smart trolley" OR "Caper Cart" OR "smart shopping cart" after:{dateFrom}
```
> 실측 누락 방지: 스마트카트 기사는 영국 매체가 'trolley'로 쓴다. Morrisons-Instacart Caper Cart 영국 출시(2026-07-29, The Grocer)는 'smart trolley'·'Caper Cart'·'Instacart' 키워드로만 포착됐고, ESL 키워드·'smart cart'(미국식)만으로는 놓쳤다.

---

## 3.5단계: 원문 확인 및 이미지 추출

### 폴백 계층 (순서대로 시도)

**1순위: WebFetch 직접 읽기**
정량 데이터·파트너십 세부 내용 확인. 이미지 우선순위:
```
1. og:image — URL에 "logo","icon","favicon","brand" 포함 시 제외
2. twitter:image — 동일 조건 제외
3. sailthru.image.full (Grocery Dive, Retail Dive)
4. 기사 본문 첫 번째 <img> (width > 300px)
5. 빈 문자열 "" — 로고보다 낫습니다
```

**2순위: 배포 미러 검색**
```
"{기사 제목 핵심 키워드}" site:finance.yahoo.com OR site:prnewswire.com OR site:businesswire.com OR site:stocktitan.net
```

**3순위: Claude in Chrome 렌더링**
JS 렌더링 필요 시 (SOLUM, Retail Dive, RTIH 등):
- `mcp__Claude_in_Chrome__list_connected_browsers` → 브라우저 확인
- `mcp__Claude_in_Chrome__navigate` → URL 열기
- `mcp__Claude_in_Chrome__get_page_text` → 텍스트 추출

**4순위: 검색 스니펫 기반 (최후 수단)**
- 여러 결과에서 일관된 팩트만 사용
- summary 끝에 `*(검색 스니펫 기반)*` 표기
- imageUrl = `""`

---

## 3.6단계: 리서치 실패 케이스 대응 (트러블슈팅)

수집 중 자주 발생하는 실패 유형과 복구 절차다. 실패를 만나면 그냥 건너뛰지 말고 아래 대체 경로를 한 단계씩 시도한다.

### A. RSS가 gzip 바이너리로 반환됨 (Grocery Dive·RTIH 등)
- **증상**: WebFetch 결과가 `[binary data]` 또는 깨진 문자.
- **복구**: RSS를 포기하고 매체 웹 페이지를 직접 WebFetch. RTIH는 3A단계의 'RTIH 전용 경로'(특히 경로 ① site: 열거)를, Grocery Dive는 `https://www.grocerydive.com/topic/technology/`를 읽고 날짜 필터.

### B. WebFetch 결과가 토큰 한도 초과 (대형 페이지)
- **증상**: "result exceeds maximum allowed tokens. Output has been saved to /…/tool-results/…txt".
- **복구**: 전체를 다시 읽지 말 것. 저장된 결과 파일 경로에 **Grep 도구**로 핵심 패턴만 추출:
  - 날짜: `/{YYYY}/{M}/` 또는 `(Jul|July)[ -]?(0?[1-9]|[12][0-9]|3[01])`
  - 주제: `shelf label|electronic shelf|ESL|dynamic pric|surveillance pric|robot|hyper-connected|smart shelf` (`-i`)
- 필요 시 Read 도구로 offset/limit를 줘 해당 라인 주변만 부분 읽기.

### C. JS 렌더링/페이월로 본문이 비어 있음 (Retail Dive·Grocery Dive·SOLUM·RTIH·The Grocer 등)
- **증상**: WebFetch는 성공했지만 메뉴·푸터만 있고 기사 본문이 없음(클라이언트 렌더링) 또는 페이월로 빈 본문.
- **복구**: Claude in Chrome로 전환 — `list_connected_browsers` → `navigate` → `get_page_text`. Chrome 미연결 시 3A 경로 ①(site: 열거) 또는 미러 검색으로 대체.
- **The Grocer 전용**: 페이월+JS라 WebFetch가 항상 빈 본문을 반환한다. 3.5단계 2순위 **미러 검색**(StockTitan·PRNewswire·businesswire·finance.yahoo)으로 원문·날짜·이미지를 확보한다. 예: PR형 기사는 `WebSearch "{제목 핵심어}" site:stocktitan.net OR site:prnewswire.com` → 미러 fetch(케이스 G-2 언락).

### D. 검색 결과 날짜가 불명확 / 기간 외 혼입
- **증상**: 검색 스니펫에 발행일이 없거나, 옛 기사가 상위 노출.
- **복구**: 채택 전 **원문 페이지의 발행일을 반드시 확인**. 메타태그 `article:published_time` / 본문 날짜 헤더로 검증하고, dateFrom~dateTo 밖이면 제외. 날짜 확인 불가 시 채택하지 않는다.

### E. 공급사 뉴스룸이 SPA라 목록이 안 보임 (SOLUM 등)
- **증상**: 뉴스룸 페이지에 기사 카드가 렌더링되지 않거나 카운트만 표시.
- **복구**: ① Claude in Chrome로 렌더링 후 목록 확인, ② 실패 시 `site:solum-group.com after:{dateFrom}` 등 도메인 한정 검색, ③ 그래도 없으면 wire service(businesswire·prnewswire) 검색으로 교차 확인.

### F. EGRESS_BLOCKED / 차단
- **증상**: 특정 URL이 차단됨.
- **복구**: 우회(curl 등) 금지. 해당 URL은 건너뛰고 다음 소스로 진행하며, 같은 기사를 다른 매체·wire service 미러에서 탐색.

### G. Provenance 차단 — "URL not in provenance set" ⭐ NEW (구조적·최우선 이해)
- **증상**: WebFetch가 `URL not in provenance set. web_fetch can only retrieve URLs that appeared in a user message, a prior web_fetch result, or a WebSearch result` 오류 반환. 재시도는 무조건 실패.
- **원인**: WebFetch는 아래 셋 중 하나에 **이미 등장한 URL만** 가져올 수 있다: ① 사용자 메시지(=**이 SKILL.md 본문에 그대로 적힌 URL 포함**), ② 직전 WebSearch 결과에 뜬 URL, ③ 이전 WebFetch 본문에 들어있던 URL. 즉석에서 만든 변형 URL(캐시버스터 `?t=`, 새 쿼리형 Google News RSS, 페이지네이션 등)은 provenance에 없어 **전부 차단**된다.
- **복구 (URL을 provenance에 "언락"하는 3가지 경로)**:
  1. **SKILL.md 사전 등록**: 앞으로 쓰고 싶은 URL(Google News RSS 쿼리형·사이트맵·AMP·모바일 서브도메인 등)은 반드시 이 문서 본문에 **미리 적어둔다**. 적힌 URL은 자동으로 provenance에 들어와 바로 fetch 가능하다. (3A단계의 URL 목록이 이 역할을 한다 — 목록에 없는 URL을 새로 만들어 쓰지 말 것.)
  2. **WebSearch 언락**: 목록에 없는 개별 기사·미러를 읽어야 하면 먼저 `WebSearch`로 그 URL이 결과에 뜨게 한 뒤 fetch한다. 예: `WebSearch "<기사 제목>" site:finance.yahoo.com OR site:msn.com OR site:businesswire.com` → 미러 URL이 결과에 등장 → provenance 통과 → fetch. (이번 주 CBC 원문이 JS로 비어 Yahoo 미러로 대체 성공한 방식.)
  3. **연쇄 fetch**: 목록 페이지를 먼저 fetch하면 그 본문에 실린 개별 기사 URL들이 provenance에 편입돼, 이어서 그 기사들을 fetch할 수 있다.
- **서브에이전트 주의**: 서브에이전트의 WebFetch는 provenance가 부모와 분리돼 뉴스룸 직접 fetch가 더 자주 막힌다. 따라서 **뉴스룸 직접 순회(2.5)는 메인 에이전트가 수행**하고, 서브에이전트에는 *키워드 검색 팬아웃*만 위임한다. 서브에이전트가 특정 URL을 읽어야 하면 그 전에 반드시 `WebSearch`로 언락한다.

### H. Stale 캐시 오탐 — fetch는 성공했으나 내용이 오래됨 ⭐ NEW (0건 오판의 주범)
- **증상**: WebFetch가 정상 성공(오류 없음)했는데, 페이지의 **가장 최신 기사 날짜가 dateFrom보다 이전**이다. 크롤 캐시가 며칠~수개월 정체된 상태. (이번 주 실측: Grocery Dive 토픽면 최신 7/13, Progressive Grocer 2024년 헤드라인, RTIH `/home` 2025년 12월.) 성공처럼 보여 **"신규 0건"으로 오판**되는 것이 가장 위험하다.
- **탐지 규칙 (필수)**: 인덱스·뉴스룸·토픽 페이지를 읽은 직후, 페이지에서 파싱되는 **최신 날짜(newest date)**를 확인한다. 그 값이 `dateFrom`보다 이전이면 그 소스는 **STALE**로 판정하고, 0건 근거로 쓰지 말고 즉시 대체 경로로 라우팅한다.
- **복구 (STALE 시 대체 경로, 순서대로)**:
  1. 같은 소스의 **다른 표현 URL**(이미 3A에 등록된 것): 사이트맵(`sitemap-news.xml`) → Google News RSS `site:{domain}` 쿼리형 → `site:{domain} after:{dateFrom}` 검색으로 **개별 기사 URL**을 확보. 토픽면은 캐시돼도 개별 기사 페이지는 신선한 경우가 많다.
  2. 개별 기사 URL을 WebSearch로 언락(케이스 G-2) 후 원문 fetch → `article:published_time`으로 발행일 재확인.
  3. 그래도 안 되면 wire service·미러(Yahoo/MSN/PRNewswire)로 교차.
- **로그**: STALE 판정 시 searchAnalysis에 `[STALE:{소스} 최신{날짜}→대체경로]` 형식으로 남긴다. 특정 소스가 반복 STALE이면 다음 개선 때 대체 URL을 tier-1로 승격한다.

> 위 A~H 중 하나라도 발동했으면 searchAnalysis에 어떤 소스에서 무슨 우회를 썼는지 1줄로 기록해 다음 주 디버깅을 돕는다. **특히 G(provenance)·H(stale)는 "성공한 척"하는 실패이므로 반드시 기록한다.**

---

## 3.7단계: 커버리지 감사 (수집 종료 전 필수) ⭐ NEW

기사 분류·저장 직전, 뉴스룸 순회·검색에서 놓친 범위 내 기사가 없는지 **교차 대조**한다. 0건으로 판정되는 주차에도 반드시 수행한다.

1. 핵심 뉴스룸별로 당월 `site:` 열거 검색을 1회씩 실행해 URL 목록을 수집한다:
   - `site:retailtechinnovationhub.com/home/{YYYY}/{M}`
   - `ESL site:esmmagazine.com after:{dateFrom}`
   - `"electronic shelf label" OR ESL site:retaildive.com OR site:grocerydive.com after:{dateFrom}`
   - `shelf label OR ESL site:thegrocer.co.uk after:{dateFrom}`
2. 각 결과 URL의 발행일(URL 날짜 패턴 또는 원문 메타)이 dateFrom~dateTo인지 확인한다.
3. 범위 내인데 이번 run에서 **수집되지 않은 URL**이 있으면 3.5단계로 돌아가 원문을 읽고 채택 여부를 판단한다.
4. 감사 결과를 `searchAnalysis`에 1줄 남긴다 (예: `[감사] 범위 내 미수집 0건 확인` 또는 `[감사] RTIH 1건 추가 포착`).

> 이 단계의 목적은 "검색 무응답"을 "실제 0건"으로 성급히 결론 내리지 않게 하는 안전장치다.

---

## 4단계: 기사 섹션 분류

5개 섹션으로 분류:
- **industryTrends**: 산업 전반 동향, 규제, 시장 통계
- **customerAdoption**: 유통사 도입 사례 (세그먼트별 객체)
- **majorCompanies**: 주요 유통사(Walmart, Carrefour, Tesco 등) 동향
- **solutionProviders**: ESL 솔루션 공급사 (Vusion, Pricer, Hanshow, SOLUM 등)
- **technology**: 기술 트렌드 (배터리리스, RFID, AI, 리테일 미디어, RMN, 재고관리·컴퓨터 비전·재고 로봇 등)

**RMN·재고관리 기사 분류 가이드**:
- 특정 **공급사의 RMN/재고 솔루션 발표**(예: Vusion Engage·Captana, Hanshow SPatrol, SOLUM Retail Media Network) → **solutionProviders**
- 특정 **유통사의 RMN/재고 시스템 도입**(예: Walmart 매장 광고 네트워크 확대, Kroger 재고 로봇 도입) → **customerAdoption**(세그먼트) 또는 **majorCompanies**
- **시장 전반의 기술 트렌드·신기술**(컴퓨터 비전, 온셸프 가용성, in-store 리테일 미디어 성장 등) → **technology**
- **수집 스코프(확대) ⭐⭐**: ESL/전자라벨이 1순위이되, **리테일 산업의 최신 기술·트렌드 전반**도 수집 대상이다. ESL과 직접 연관이 없어도 아래 주제는 수집한다:
  - 디지털 트윈, 에이전틱 AI·AI 어시스턴트·AI 코파일럿, 컴퓨터 비전·엣지 AI, 자율/청소/재고/피킹/휴머노이드 로봇(Simbe·Tally·Badger·Brain Corp·Symbotic 등), 스마트카트·스마트 트롤리(smart cart/trolley·Caper Cart·Instacart), 셀프체크아웃·Just Walk Out, 리테일 미디어(RMN), 디지털 사이니지·DOOH·인스토어 광고 스크린(Cooler Screens·Grocery TV·Vibenomics·Broadsign 등)
  - 가격 투명성·다이내믹 프라이싱·감시가격 규제, 가격비교/가격추적, 공급망 AI, 배송·긱워커·주문이행(피킹) 기술, 페이먼트·로열티 기술
  - 리테일 부문 건전성(파산·폐점·구조조정·수요 위축·대형 M&A) — 신규 매장 투자(ESL CAPEX)·비용절감 흐름에 직접 영향을 주므로 industryTrends로 수집
  - → 분류: 기술·신기술 트렌드는 **technology**, 가격·규제·시장 이슈는 **industryTrends**, 특정 유통사 움직임은 **majorCompanies/customerAdoption**, 공급사 솔루션은 **solutionProviders**
  - implications에는 가능한 한 SOLUM/ESL 관점의 시사점을 덧붙인다(직접 연관이 약하면 '인접 트렌드'로 간략히).
  - **제외**: 리테일 산업과 무관한 순수 기업 인사·부동산·식품 신제품·연예/스포츠 등.

각 기사 필드:
```json
{
  "title": "(한국어 번역 제목 — 원문이 영어여도 한국어로 번역)",
  "summary": "• [핵심 사실 1]\n• [핵심 사실 2]\n• [배경·맥락]\n• [추가 세부사항]",
  "implications": "(한국어, SOLUM 관점)",
  "imageUrl": "",
  "pubDate": "YYYY-MM-DD",
  "sourceUrl": "",
  "sourceName": ""
}
```

**title 규칙**: 영문 원제를 한국어로 자연스럽게 번역합니다.

**summary 형식 (불릿 필수)**:
```
• [핵심 사실 1 — 무슨 일이 일어났는가]
• [핵심 사실 2 — 주요 수치·기업·지역 등 구체 정보]
• [배경 또는 맥락 — 왜 중요한가]
• [추가 세부사항 또는 파급 효과] (선택)
```
3~5개 불릿, 각 1~2문장으로 간결하게 작성합니다.

**시사점(implications) 태그**: `[🔴 위협]` / `[🟢 기회]` / `[🔵 트렌드]` / `[⚡ 액션]`

**중복 방지**: 기존 파일 내 sourceUrl과 동일한 기사는 제외합니다.

---

## 5단계: JSON 파일 저장

### 이번 주 파일 (`{weekId}.json`)

기사 1건 이상인 경우:

**파일 존재 시**: 섹션에 추가 후 Write 저장

**파일 없을 시** — 신규 생성:
```json
{
  "weekId": "YYYY-MM-DD",
  "weekLabel": "YYYY.MM.DD ~ YYYY.MM.DD",
  "searchAnalysis": "**MM월 DD일(목) 수집:** 주간 기사 N건—[요약].",
  "sections": {
    "industryTrends": [],
    "customerAdoption": {},
    "majorCompanies": [],
    "solutionProviders": [],
    "technology": []
  }
}
```

### index.json 업데이트

Read 후:
- 해당 주 항목 있으면 `articleCount` 업데이트
- 없으면 배열 끝에 추가:
```json
{ "weekId": "YYYY-MM-DD", "weekLabel": "YYYY.MM.DD ~ YYYY.MM.DD", "articleCount": N }
```
Write로 저장.

---

## 6단계: Git 커밋

```bash
cd {esl-weekly-report 루트}
[ -f .git/HEAD.lock ] && mv .git/HEAD.lock .git/HEAD.lock.old
[ -f .git/index.lock ] && mv .git/index.lock .git/index.lock.old
git add data/index.json data/{weekId}.json
git commit -m "feat: {weekLabel} 주간 수집 (+N건)"
```
push는 수동으로 진행합니다.

---

## 중요 규칙

- **pubDate 필수**: 모든 기사에 발행일 포함. 날짜 불확실 시 제외
- **중복 금지**: sourceUrl 기준 중복 확인
- **한국어 필수**: title·summary·implications 모두 한국어
- **summary 불릿 필수**: 반드시 • 불릿 3~5줄
- **"0건 확정" 기준 강화 ⭐**: "검색 무응답 = 0건"으로 성급히 결론 내리지 않는다. **2.5단계 뉴스룸 직접 순회 + 3.7단계 커버리지 감사(site: 열거 대조)를 모두 완료**했을 때만 0건으로 확정한다. 이 두 단계를 건너뛰고 0건 처리하는 것을 금지한다.
- **STALE·Provenance 실패는 "성공한 척"이다 ⭐ NEW**: 뉴스룸·토픽 페이지를 읽었을 때 (a) 최신 날짜가 dateFrom보다 이전이면(3.6-H STALE), (b) fetch가 provenance로 막혔으면(3.6-G) — 그 소스는 **"확인 완료"가 아니라 "미확인"**이다. 이 상태를 0건 근거로 삼지 말고 3.6-G/H의 대체 경로를 반드시 소진한다. 새 URL을 쓰려면 이 SKILL.md에 등록돼 있거나 WebSearch로 언락된 것이어야 한다.
- **기사 없으면**: searchAnalysis에 "수집 기사 없음" 메모만 추가 (위 0건 확정 기준 충족 시)
- **실행 순서**: 2.5단계(뉴스룸·필수) → 3A(RSS) → 3B(검색) → 3.5(원문) → 3.7(커버리지 감사·필수). 실패 발생 시 **3.6단계(트러블슈팅)**의 대체 경로 적용
- **RTIH RSS는 gzip이라 항상 실패 + /home도 SPA** → 3A단계 'RTIH 전용 경로' 경로 ①(site: 날짜 열거)를 최우선으로, 경로 ②(/home 직접 읽기)를 보조로, **경로 ③(카테고리·공급사 태그 페이지 교차확인)을 매 실행 필수**로 사용. ESL 태그만 보지 말고 Retail media·Acquisitions·Stores 카테고리와 SOLUM/Vusion/Hanshow/Pricer 태그를 반드시 교차확인한다.
- **RMN·재고관리 검색 필수 ⭐**: 3B의 [리테일 미디어 네트워크(RMN)]·[재고관리] 쿼리 블록은 매 실행 반드시 돌린다. 공급사가 ESL을 넘어 리테일 미디어·M&A·재고 솔루션으로 확장하는 기사는 ESL 키워드 검색으로는 안 잡히므로, 이 블록을 건너뛰면 공급사 핵심 동향을 놓친다.
- 완료 메시지: `"✅ ESL 주간 리포트 완료 | [weekLabel] | 총 N건 수집 (뉴스룸 M건 + RSS P건 + 검색 Q건)"`
