# 웹 검색 엔진 통합 가이드

> **브랜치**: `feat/pluggable-web-search`  
> **베이스**: `hikaMaeng/claw-code` (fork of `ultraworkers/claw-code`)

---

## 목차

1. [설계 개요](#1-설계-개요)
2. [아키텍처](#2-아키텍처)
3. [수정된 파일 목록](#3-수정된-파일-목록)
4. [settings.json — 활성 프로바이더 설정](#4-settingsjson--활성-프로바이더-설정)
5. [searchProvider.json — 엔진 구동 명세](#5-searchproviderjson--엔진-구동-명세)
6. [내장 프로바이더 레퍼런스](#6-내장-프로바이더-레퍼런스)
7. [커스텀 프로바이더 추가](#7-커스텀-프로바이더-추가)
8. [시작 배너](#8-시작-배너)
9. [런타임 동작 흐름](#9-런타임-동작-흐름)
10. [설정 파일 우선순위](#10-설정-파일-우선순위)
11. [코드 레퍼런스](#11-코드-레퍼런스)
12. [변경 이력](#12-변경-이력)

---

## 1. 설계 개요

### 원본 코드의 문제

원본 claw-code는 웹 검색 엔진이 DuckDuckGo(DDG) 하나로 고정되어 있었다. 검색 API를 바꾸려면 코드를 수정하고 재빌드해야 했다.

### 이 패치의 목표

두 가지 관심사를 분리한다.

| 관심사 | 담당 파일 | 변경 주체 |
|--------|-----------|-----------|
| **어떤 엔진을 쓸지** (선택 + API 키) | `settings.json` | 사용자 |
| **각 엔진이 어떻게 동작하는지** (HTTP 스펙 + 파싱 규칙) | `searchProvider.json` | 시스템 (이미지에 내장) |

외부 검색 API의 스펙이 바뀌면 `searchProvider.json`만 수정하고 컨테이너를 재시작한다. **재빌드 불필요**.

### 설계 원칙

- `settings.json`은 cascading 지원 — 전역 설정 위에 프로젝트별 설정이 덮어쓴다
- `searchProvider.json`도 동일한 우선순위 체계 — 워크스페이스 로컬 → 유저 홈 → 시스템 기본값
- Rust 코드는 범용 HTTP 실행기로만 동작 — 프로바이더별 로직 없음
- DDG만 HTML 파서가 필요해 특수 처리, 나머지는 모두 JSON API 범용 경로

---

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    claw (Rust 바이너리)                   │
│                                                          │
│  WebSearch 툴 호출                                        │
│       │                                                  │
│       ▼                                                  │
│  execute_web_search()                                    │
│       │                                                  │
│       ├─ ConfigLoader::default_for(cwd).load()           │
│       │    └─ settings.json 읽기 (cascading)              │
│       │         provider: "tavily"                        │
│       │         apiKey:   "tvly-..."                      │
│       │                                                  │
│       ├─ load_search_provider_spec("tavily")              │
│       │    └─ searchProvider.json 읽기 (우선순위 탐색)      │
│       │         endpoint, method, auth, body,             │
│       │         resultPath, titleField, urlField          │
│       │                                                  │
│       ├─ format == "html" ?                              │
│       │    ├─ YES → execute_web_search_html()            │
│       │    │         DDG HTML 파서 (extract_search_hits)  │
│       │    └─ NO  → 범용 JSON API 실행기                  │
│       │              substitute_query($q → 실제 쿼리)     │
│       │              apply_auth(bearer | header:X-...)    │
│       │              HTTP GET/POST                        │
│       │              json_path(resultPath)               │
│       │              SearchHit 배열 추출                   │
│       │                                                  │
│       └─ apply_domain_filters → dedupe → truncate(8)     │
│            └─ WebSearchOutput 반환                        │
└─────────────────────────────────────────────────────────┘

설정 파일 탐색 순서 (먼저 발견된 것 사용):
  1. {cwd}/.claw/searchProvider.json     ← 워크스페이스 로컬 오버라이드
  2. ~/.config/claw/searchProvider.json  ← 유저 홈 오버라이드
  3. /etc/claw/searchProvider.json       ← 시스템 기본값 (이미지에 내장)
```

---

## 3. 수정된 파일 목록

### rust/crates/runtime/src/config.rs

**추가된 내용:**

```rust
/// Web search provider settings from settings.json `websearch` section.
#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub struct WebSearchConfig {
    pub provider: Option<String>,
    pub api_key: Option<String>,
}
```

`RuntimeFeatureConfig` 구조체에 `web_search: WebSearchConfig` 필드 추가.

`RuntimeConfig`에 접근자 추가:
```rust
pub fn web_search(&self) -> &WebSearchConfig {
    &self.feature_config.web_search
}
```

파서 함수 추가:
```rust
fn parse_optional_web_search_config(root: &JsonValue) -> WebSearchConfig {
    let Some(obj) = root.as_object() else { return WebSearchConfig::default(); };
    let Some(ws) = obj.get("websearch").and_then(JsonValue::as_object) else {
        return WebSearchConfig::default();
    };
    WebSearchConfig {
        provider: ws.get("provider").and_then(JsonValue::as_str).map(ToOwned::to_owned),
        api_key:  ws.get("apiKey").and_then(JsonValue::as_str).map(ToOwned::to_owned),
    }
}
```

---

### rust/crates/runtime/src/config_validate.rs

`TOP_LEVEL_FIELDS` 배열에 `websearch` 항목 추가. 이 배열에 없는 키는 settings.json 로드 시 오류로 거부된다.

```rust
FieldSpec { name: "websearch", expected: FieldType::Object },
```

---

### rust/crates/runtime/src/lib.rs

public 재수출 목록에 `WebSearchConfig` 추가:

```rust
pub use config::{
    ...,
    WebSearchConfig,
    ...
};
```

---

### rust/crates/tools/src/lib.rs

가장 많이 변경된 파일. 핵심 변경 사항:

**1) import 추가**
```rust
use runtime::{
    ...,
    WebSearchConfig,
    ConfigLoader,
    ...
};
```

**2) 프로바이더별 하드코딩 함수 제거**  
기존의 `execute_web_search_ddg`, `execute_web_search_tavily`, `execute_web_search_brave`,
`execute_web_search_firecrawl` 함수가 모두 제거되었다.

**3) 범용 실행기로 교체**  
세 가지 핵심 헬퍼 함수가 추가되었다:

- `load_search_provider_spec(provider)` — `searchProvider.json`에서 해당 프로바이더 스펙을 읽어 반환
- `json_path(val, path)` — `"web.results"` 같은 점 구분 경로로 JSON을 탐색
- `substitute_query(val, query)` — body 템플릿의 `"$q"` 문자열을 실제 쿼리로 치환

`apply_auth(req, auth_spec, api_key)` 함수:
- `"bearer"` → `Authorization: Bearer {api_key}`
- `"header:X-Foo"` → `X-Foo: {api_key}`
- 그 외 → 인증 헤더 없음

`execute_web_search()` — 메인 디스패처:
- settings.json에서 `provider`, `apiKey` 읽기
- `load_search_provider_spec(provider)` 호출
- `format == "html"` 이면 HTML 파서 경로, 아니면 JSON API 경로

`execute_web_search_html()` — DDG 전용 HTML 파서 경로:  
기존 `extract_search_hits()` / `extract_search_hits_from_generic_links()` HTML 파싱 로직을 재사용.

---

### rust/crates/rusty-claude-cli/src/main.rs

`startup_banner()` 함수 수정. `workspace` 계산 블록 다음에 `websearch_label` 계산 블록 추가:

```rust
let websearch_label = {
    use runtime::ConfigLoader;
    let cwd_path = std::path::Path::new(&cwd);
    ConfigLoader::default_for(cwd_path)
        .load()
        .ok()
        .map(|cfg| {
            let ws = cfg.web_search();
            let provider = ws.provider.as_deref().unwrap_or("ddg");
            let has_key = ws.api_key.as_deref().map_or(false, |k| !k.is_empty());
            match provider {
                "ddg" => "ddg  \x1b[2m(default)\x1b[0m".to_string(),
                p if !has_key => format!("{p}  \x1b[33m⚠ apiKey 없음\x1b[0m"),
                p => format!("{p}  \x1b[32m✓\x1b[0m"),
            }
        })
        .unwrap_or_else(|| "ddg  \x1b[2m(default)\x1b[0m".to_string())
};
```

format 문자열에 `WebSearch` 줄 추가 (Permissions 다음, Branch 앞):
```rust
"  \x1b[2mWebSearch\x1b[0m        {}\n\",
// ...
websearch_label,
```

---

### searchProvider.json (신규, 레포 루트)

엔진 구동 명세 파일. Dockerfile이 이 파일을 `/etc/claw/searchProvider.json`으로 COPY한다.

---

## 4. settings.json — 활성 프로바이더 설정

`settings.json`의 `websearch` 블록에서 활성 프로바이더와 API 키를 설정한다.

### 위치

claw의 cascading 설정 구조를 따른다:

```
~/.claw/settings.json          ← 전역 (모든 워크스페이스에 적용)
{project}/.claw/settings.json  ← 프로젝트별 (전역을 덮어씀)
```

### 스키마

```json
{
  "websearch": {
    "provider": "tavily",
    "apiKey": "your-api-key-here"
  }
}
```

| 필드 | 타입 | 기본값 | 설명 |
|------|------|--------|------|
| `provider` | string | `"ddg"` | 사용할 검색 엔진 이름. `searchProvider.json`의 키와 일치해야 함 |
| `apiKey` | string | `""` | 해당 엔진의 API 키. DDG는 불필요 |

### 예시 — Tavily

```json
{
  "model": "openai/gpt-4o",
  "permissions": { "defaultMode": "danger-full-access" },
  "websearch": {
    "provider": "tavily",
    "apiKey": "tvly-..."
  }
}
```

### 예시 — DDG (키 없음)

```json
{
  "websearch": {
    "provider": "ddg"
  }
}
```

### 주의

- `websearch` 블록이 없으면 자동으로 `ddg`가 사용된다
- `apiKey`가 필요한 프로바이더인데 비어있으면 검색 시 오류가 반환된다
- `provider` 값이 `searchProvider.json`에 없는 이름이면 오류가 반환된다

---

## 5. searchProvider.json — 엔진 구동 명세

### 역할

각 검색 엔진에 어떻게 요청을 보내고 응답을 파싱할지를 정의한다. Rust 코드는 이 파일을 읽어서 HTTP 요청을 만들고 결과를 추출한다.

**API 스펙이 바뀌면 이 파일만 수정하고 컨테이너를 재시작한다. 재빌드 불필요.**

### 탐색 순서

claw는 아래 경로를 순서대로 탐색해 해당 프로바이더 항목을 처음 발견한 파일을 사용한다:

```
1. {현재 작업 디렉토리}/.claw/searchProvider.json   ← 워크스페이스 로컬 오버라이드
2. ~/.config/claw/searchProvider.json              ← 유저 홈 오버라이드
3. /etc/claw/searchProvider.json                  ← 시스템 기본값
```

워크스페이스 로컬 파일이 있으면 항상 우선한다. 시스템 기본값(`/etc/claw/`)을 건드리지 않고도 특정 워크스페이스에서만 다른 엔진 설정을 테스트할 수 있다.

### 스키마 — JSON API 엔진 (GET)

```json
{
  "엔진이름": {
    "method": "GET",
    "endpoint": "https://api.example.com/search",
    "auth": "header:X-Api-Key",
    "queryParam": "q",
    "resultPath": "web.results",
    "titleField": "title",
    "urlField": "url"
  }
}
```

### 스키마 — JSON API 엔진 (POST)

```json
{
  "엔진이름": {
    "method": "POST",
    "endpoint": "https://api.example.com/search",
    "auth": "bearer",
    "body": {
      "query": "$q",
      "limit": 10,
      "someOtherParam": "value"
    },
    "resultPath": "data.items",
    "titleField": "title",
    "urlField": "url"
  }
}
```

### 스키마 — HTML 파서 엔진 (DDG 전용)

```json
{
  "ddg": {
    "method": "GET",
    "endpoint": "https://html.duckduckgo.com/html/",
    "queryParam": "q",
    "format": "html"
  }
}
```

`"format": "html"`이 지정되면 JSON 파싱 대신 내장 HTML 파서(`extract_search_hits`)가 사용된다.

### 필드 레퍼런스

| 필드 | 필수 | 기본값 | 설명 |
|------|------|--------|------|
| `method` | 권장 | `"GET"` | HTTP 메서드. `"GET"` 또는 `"POST"` |
| `endpoint` | **필수** | — | API 엔드포인트 URL |
| `auth` | 선택 | `"none"` | 인증 방식 (하단 표 참조) |
| `queryParam` | GET 필수 | `"q"` | 쿼리 문자열 파라미터 이름 |
| `body` | POST 필수 | — | 요청 본문 JSON 템플릿. `"$q"` 가 쿼리로 치환됨 |
| `resultPath` | 선택 | `"results"` | 응답 JSON에서 결과 배열의 경로. 점(`.`) 구분 |
| `titleField` | 선택 | `"title"` | 각 결과 객체에서 제목 필드 이름 |
| `urlField` | 선택 | `"url"` | 각 결과 객체에서 URL 필드 이름 |
| `format` | 선택 | `"json"` | `"html"` 로 설정하면 HTML 파서 경로 사용 |

### auth 필드 값

| 값 | 동작 |
|----|------|
| `"none"` 또는 생략 | 인증 헤더 없음 |
| `"bearer"` | `Authorization: Bearer {apiKey}` 헤더 추가 |
| `"header:헤더이름"` | `{헤더이름}: {apiKey}` 헤더 추가 |

예시:
- `"auth": "bearer"` → `Authorization: Bearer sk-abc123`
- `"auth": "header:X-Subscription-Token"` → `X-Subscription-Token: sk-abc123`
- `"auth": "header:Api-Key"` → `Api-Key: sk-abc123`

### $q 치환

`body` 내 어디에든 `"$q"` 문자열이 있으면 런타임에 실제 검색 쿼리로 치환된다. 중첩 객체도 재귀적으로 치환된다.

```json
"body": {
  "query": "$q",
  "metadata": { "source": "$q" }
}
```

쿼리가 `"서울 날씨"`이면:
```json
{
  "query": "서울 날씨",
  "metadata": { "source": "서울 날씨" }
}
```

### resultPath 점 표기법

응답 JSON이 중첩 구조일 때 점(`.`)으로 경로를 지정한다.

```json
// 응답 예시
{
  "web": {
    "results": [
      { "title": "...", "url": "..." }
    ]
  }
}
```

`"resultPath": "web.results"` 로 지정하면 `results` 배열을 정확히 가리킨다.

---

## 6. 내장 프로바이더 레퍼런스

시스템 기본값 (`/etc/claw/searchProvider.json`):

### ddg — DuckDuckGo (무료, 키 불필요)

```json
{
  "method": "GET",
  "endpoint": "https://html.duckduckgo.com/html/",
  "queryParam": "q",
  "format": "html"
}
```

- API 키 불필요
- HTML 응답을 내장 파서로 파싱
- 속도 제한 없음, 무료
- 검색 품질은 유료 API 대비 낮을 수 있음

**settings.json 설정:**
```json
{ "websearch": { "provider": "ddg" } }
```

---

### tavily — Tavily Search API

```json
{
  "method": "POST",
  "endpoint": "https://api.tavily.com/search",
  "auth": "bearer",
  "body": { "query": "$q", "max_results": 15, "include_answer": false },
  "resultPath": "results",
  "titleField": "title",
  "urlField": "url"
}
```

- API 키 필요
- 키 발급: https://app.tavily.com
- AI 검색에 최적화된 결과 제공
- Free tier 월 1,000 크레딧

**settings.json 설정:**
```json
{
  "websearch": {
    "provider": "tavily",
    "apiKey": "tvly-..."
  }
}
```

---

### brave — Brave Search API

```json
{
  "method": "GET",
  "endpoint": "https://api.search.brave.com/res/v1/web/search",
  "auth": "header:X-Subscription-Token",
  "queryParam": "q",
  "resultPath": "web.results",
  "titleField": "title",
  "urlField": "url"
}
```

- API 키 필요
- 키 발급: https://brave.com/search/api/
- 독립적인 인덱스 보유, 개인정보 친화적
- Free tier 월 2,000 쿼리
- **주의**: `Accept-Encoding: gzip` 헤더를 보내면 reqwest가 압축 해제를 못해 오류 발생 (이 패치에서 이미 해당 헤더 제거됨)

**settings.json 설정:**
```json
{
  "websearch": {
    "provider": "brave",
    "apiKey": "BSA..."
  }
}
```

---

### firecrawl — Firecrawl Search

```json
{
  "method": "POST",
  "endpoint": "https://api.firecrawl.dev/v1/search",
  "auth": "bearer",
  "body": { "query": "$q", "limit": 10 },
  "resultPath": "data",
  "titleField": "title",
  "urlField": "url"
}
```

- API 키 필요
- 키 발급: https://firecrawl.dev
- 웹 크롤링 기반 검색, 페이지 본문 추출 능력 우수
- Free tier 월 500 크레딧

**settings.json 설정:**
```json
{
  "websearch": {
    "provider": "firecrawl",
    "apiKey": "fc-..."
  }
}
```

---

## 7. 커스텀 프로바이더 추가

재빌드 없이 새 검색 엔진을 추가할 수 있다.

### 단계 1 — searchProvider.json에 항목 추가

워크스페이스 로컬 오버라이드 파일을 만들거나 시스템 파일에 추가한다.

**워크스페이스 로컬 (재시작만 필요):**
```bash
cat >> /workspace/.claw/searchProvider.json << 'EOF'
# 또는 파일을 새로 만들어 원하는 엔진 항목만 포함
EOF
```

**예시 — Serper.dev 추가 (GET, bearer):**
```json
{
  "serper": {
    "method": "POST",
    "endpoint": "https://google.serper.dev/search",
    "auth": "header:X-API-KEY",
    "body": { "q": "$q", "num": 10 },
    "resultPath": "organic",
    "titleField": "title",
    "urlField": "link"
  }
}
```

**예시 — SearXNG 자체 호스팅 인스턴스 추가 (GET, 키 없음):**
```json
{
  "searxng": {
    "method": "GET",
    "endpoint": "https://my-searxng.example.com/search",
    "auth": "none",
    "queryParam": "q",
    "resultPath": "results",
    "titleField": "title",
    "urlField": "url"
  }
}
```

### 단계 2 — settings.json에서 활성화

```json
{
  "websearch": {
    "provider": "serper",
    "apiKey": "your-serper-key"
  }
}
```

### 단계 3 — 재시작

```bash
docker compose restart
```

재빌드 없이 즉시 적용된다.

### 새 엔진 추가 시 확인 사항

1. **resultPath**: `curl` 로 API를 직접 호출해 응답 JSON 구조를 확인하고, 결과 배열까지의 경로를 점 표기법으로 작성
2. **auth**: API 문서에서 인증 방식 확인 (`Bearer`, 커스텀 헤더 등)
3. **body의 $q 위치**: POST API라면 쿼리가 들어가는 필드 이름이 `query`인지 `q`인지 확인

---

## 8. 시작 배너

claw를 대화형 REPL 모드로 실행하면 시작 배너에 현재 활성 검색 엔진이 표시된다.

```
 ██████╗██╗      █████╗ ██╗    ██╗
██╔════╝██║     ██╔══██╗██║    ██║
...
  Model            openai/gpt-4o
  Permissions      danger-full-access
  WebSearch        tavily  ✓              ← 활성 프로바이더 표시
  Branch           main
  Workspace        clean
```

### 표시 형태

| 상태 | 표시 |
|------|------|
| DDG (기본값, 키 없음) | `ddg  (default)` (회색) |
| 프로바이더 설정 + API 키 있음 | `tavily  ✓` (초록) |
| 프로바이더 설정 + API 키 없음 | `brave  ⚠ apiKey 없음` (노랑) |

배너는 대화형 모드에서만 출력된다. `--output-format json` 같은 비대화형 모드에서는 출력되지 않는다.

---

## 9. 런타임 동작 흐름

claw 내부에서 `WebSearch` 툴이 호출될 때의 처리 순서:

```
1. WebSearch 툴 입력 수신
   { query: "서울 날씨", allowed_domains: null, blocked_domains: null }

2. execute_web_search(input) 호출
   │
   ├─ ConfigLoader::default_for(cwd).load()
   │    cascading settings.json 읽기
   │    provider = "tavily", api_key = "tvly-..."
   │
   ├─ load_search_provider_spec("tavily")
   │    searchProvider.json 탐색 (cwd/.claw/ → ~/.config/claw/ → /etc/claw/)
   │    spec = { method:"POST", endpoint:"...", auth:"bearer", body:{...}, ... }
   │
   ├─ spec["format"] == "html" ? → NO
   │
   ├─ api_key 필요 && api_key 비어있음 ? → NO (키 있음)
   │
   ├─ method == "POST"
   │    body = substitute_query(spec["body"], "서울 날씨")
   │         = { "query": "서울 날씨", "max_results": 15, "include_answer": false }
   │    req  = POST https://api.tavily.com/search
   │           Authorization: Bearer tvly-...
   │           Content-Type: application/json
   │           body (JSON)
   │
   ├─ HTTP 응답 수신 (200 OK)
   │    { "results": [ {title:..., url:...}, ... ] }
   │
   ├─ json_path(data, "results") → 결과 배열
   │
   ├─ SearchHit 배열 구성
   │    [ {title: "...", url: "https://..."}, ... ]
   │
   ├─ apply_domain_filters (allowed/blocked 도메인 필터)
   ├─ dedupe_hits (URL 중복 제거)
   └─ truncate(8) → 최대 8개
        └─ WebSearchOutput 반환
```

---

## 10. 설정 파일 우선순위

### settings.json (활성 프로바이더 선택)

claw의 기존 cascading 체계를 그대로 따른다:

```
우선순위 높음
  {project}/.claw/settings.json        ← 프로젝트별
  ~/.claw/settings.json                ← 전역
우선순위 낮음
```

하위 파일의 `websearch` 블록이 상위를 덮어쓴다.

### searchProvider.json (엔진 구동 명세)

별도의 탐색 순서를 가진다:

```
우선순위 높음
  {cwd}/.claw/searchProvider.json      ← 워크스페이스 로컬 오버라이드
  ~/.config/claw/searchProvider.json   ← 유저 홈 오버라이드
  /etc/claw/searchProvider.json        ← 시스템 기본값 (이미지 내장)
우선순위 낮음
```

**중요**: 파일 전체를 덮어쓰는 것이 아니라, 프로바이더 이름으로 탐색한다. 즉, `tavily` 스펙을 찾을 때 `cwd/.claw/searchProvider.json`에 `tavily` 키가 없으면 다음 파일로 넘어간다. 로컬 파일에 `tavily` 항목만 있어도 다른 프로바이더(`brave`, `firecrawl`)는 시스템 기본값에서 읽힌다.

---

## 11. 코드 레퍼런스

### 관련 함수 위치

| 함수 | 파일 | 역할 |
|------|------|------|
| `execute_web_search` | `rust/crates/tools/src/lib.rs` | 메인 디스패처 |
| `load_search_provider_spec` | `rust/crates/tools/src/lib.rs` | JSON 스펙 로더 |
| `json_path` | `rust/crates/tools/src/lib.rs` | 점 표기법 JSON 탐색 |
| `substitute_query` | `rust/crates/tools/src/lib.rs` | `$q` 치환 |
| `apply_auth` | `rust/crates/tools/src/lib.rs` | 인증 헤더 적용 |
| `execute_web_search_html` | `rust/crates/tools/src/lib.rs` | DDG HTML 파서 경로 |
| `extract_search_hits` | `rust/crates/tools/src/lib.rs` | DDG HTML 파서 (기존) |
| `WebSearchConfig` | `rust/crates/runtime/src/config.rs` | 설정 구조체 |
| `parse_optional_web_search_config` | `rust/crates/runtime/src/config.rs` | settings.json 파서 |
| `startup_banner` | `rust/crates/rusty-claude-cli/src/main.rs` | 배너 출력 |

### 관련 설정 파일

| 파일 | 위치 | 용도 |
|------|------|------|
| `searchProvider.json` | 레포 루트 / `/etc/claw/` | 엔진 구동 명세 (이미지에 내장) |
| `settings.json` | `.claw/settings.json` | 활성 프로바이더 + API 키 |

---

## 12. 변경 이력

| 커밋 | 내용 |
|------|------|
| `bd11289` | settings.json 기반 프로바이더 설정 (환경 변수 방식 제거). DDG·Tavily·Brave·Bing·Custom 프로바이더 추가. `WebSearchConfig` 구조체 및 `websearch` 검증 필드 추가 |
| `fa93cd3` | 시작 배너에 활성 검색 엔진 표시. `startup_banner()`에 `websearch_label` 계산 블록 및 `WebSearch` 줄 추가 |
| `5f2540a` | Firecrawl 프로바이더 추가. Brave `Accept-Encoding: gzip` 헤더 제거 버그 수정 (reqwest에 gzip feature 없어 압축 해제 불가로 decode 오류 발생) |
| `0b57cc7` | Bing·Custom 프로바이더 제거 (미사용, 키 미발급) |
| `7f34d91` | **파서 외재화**: 프로바이더별 하드코딩 함수 제거. 범용 HTTP 실행기로 교체. `searchProvider.json` 도입. 탐색 순서: `{cwd}/.claw/` → `~/.config/claw/` → `/etc/claw/` |
