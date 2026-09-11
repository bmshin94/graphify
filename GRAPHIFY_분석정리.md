# Graphify 분석 정리

> 이 문서는 graphify 프로젝트를 직접 뜯어보고 정리한 분석 노트입니다.
> 설치/사용법부터 구조, 라이선스, 활용 방안, 수익화 아이디어까지 한 번에 정리했습니다.

## 관련 링크

| 구분 | 주소 |
|---|---|
| **원본 프로젝트 (upstream)** | https://github.com/Graphify-Labs/graphify |
| **이 저장소 (fork)** | https://github.com/bmshin94/graphify |
| PyPI 패키지 | https://pypi.org/project/graphifyy/ |
| 공식 사이트 | https://graphify.com |
| Early Access 플랫폼 | https://app.graphify.com/login |
| Discord | https://discord.gg/2DDrEgvZb4 |

---

## 목차

1. [한 줄 요약](#1-한-줄-요약)
2. [왜 필요한가](#2-왜-필요한가)
3. [설치 및 사용법](#3-설치-및-사용법)
4. [플러그인? 스킬? MCP?](#4-플러그인-스킬-mcp)
5. [API 토큰이 필요한가](#5-api-토큰이-필요한가)
6. [내부 구조 분석](#6-내부-구조-분석)
7. [왜 유명한가](#7-왜-유명한가)
8. [로컬 에이전트 구축에 도움이 되는가](#8-로컬-에이전트-구축에-도움이-되는가)
9. [React / PHP로 만들 수 있는가](#9-react--php로-만들-수-있는가)
10. [라이선스 체크](#10-라이선스-체크)
11. [수익화 아이디어](#11-수익화-아이디어)
12. [실행 로드맵](#12-실행-로드맵)

---

## 1. 한 줄 요약

> **내 프로젝트 전체를 "지도(지식 그래프)"로 만들어서, 파일을 일일이 읽지 않고 질문으로 찾아내는 도구**

- 패키지명: `graphifyy` (y 두 개) / CLI 명령어: `graphify` (y 하나)
- 버전: `0.9.58` (v1 미만)
- 언어: Python (약 67,700줄, 테스트 273개 파일)
- 제작사: Graphify Labs (Y Combinator S26)

---

## 2. 왜 필요한가

AI에게 "로그인 기능이 DB랑 어떻게 연결돼?"라고 물으면 현재는 이렇게 동작합니다.

```
grep "auth" → 파일 수백 개 → 하나씩 Read → 토큰 폭발 → 부정확한 답변
```

graphify를 쓰면:

```
graphify query "auth가 DB랑 어떻게 연결돼?" → 관련 서브그래프만 반환 → 정확한 답변
```

**즉, AI에게 "커닝페이퍼"를 만들어주는 도구입니다.**

---

## 3. 설치 및 사용법

### 설치

```bash
# 1. CLI 설치
uv tool install graphifyy        # 또는: pipx install graphifyy

# 2. AI 어시스턴트에 스킬 등록
graphify install
```

> **주의:** 패키지명은 `graphifyy`(y 두 개), 명령어는 `graphify`(y 하나).
> `command not found`가 뜨면 `uv tool update-shell` 후 터미널 재시작.

### 기본 사용 3단계

```bash
# ① 그래프 생성
graphify extract . --code-only    # 완전 오프라인, API 키 불필요 (추천 시작점)
graphify extract .                # 문서/PDF/이미지까지 포함 (API 키 필요)

# ② 결과 확인
#   graphify-out/graph.html       ← 브라우저로 열면 인터랙티브 그래프
#   graphify-out/GRAPH_REPORT.md  ← 요약 리포트
#   graphify-out/graph.json       ← AI가 읽는 그래프 데이터

# ③ 질문
graphify query "인증이 DB랑 어떻게 연결돼?"
graphify path "UserService" "DatabasePool"
graphify explain "RateLimiter"
```

### 그래프 최신 상태 유지

```bash
graphify update .          # 변경된 파일만 재스캔 (빠름, 무료)
graphify hook install      # git commit/checkout 시 자동 갱신 (권장)
graphify watch ./src       # 파일 저장 시 실시간 갱신
```

`hooks.py` 확인 결과, 커밋 후 rebuild가 **백그라운드로 분리 실행**되어 커밋 속도에 영향을 주지 않도록 설계되어 있습니다.

---

## 4. 플러그인? 스킬? MCP?

**전부 다입니다.** 4겹 구조로 되어 있습니다.

```
┌─────────────────────────────────────────┐
│  ① CLI 도구 (본체, Python)               │  ← 심장
├─────────────────────────────────────────┤
│  ② Skill (SKILL.md 마크다운)             │  ← /graphify 명령어
├─────────────────────────────────────────┤
│  ③ MCP 서버 (stdio / HTTP)               │  ← AI가 도구로 호출
├─────────────────────────────────────────┤
│  ④ Hook (PreToolUse / git hook)          │  ← 자동 개입
└─────────────────────────────────────────┘
```

### ① CLI 도구 — 진짜 본체
`pyproject.toml:105` → `graphify = "graphify.__main__:main"`. AI 없이 단독 실행 가능.

### ② Skill — `/graphify` 명령어
`graphify install` 실행 시 `~/.claude/skills/graphify/SKILL.md` 설치.
코드가 아니라 **AI에게 주는 마크다운 사용설명서**입니다.

| 플랫폼 | 설치 위치 |
|---|---|
| Claude Code | `.claude/skills/graphify/SKILL.md` |
| Codex | `.codex/skills/...` + `AGENTS.md` |
| Cursor | `.cursor/rules/graphify.mdc` |
| Copilot | `.copilot/skills/...` |
| 그 외 25개 플랫폼 | 각 플랫폼 규격대로 |

### ③ MCP 서버
```bash
graphify serve            # 또는 /graphify . --mcp
```

`serve.py`에서 노출하는 MCP 도구 **10개**:

```
query_graph · get_node · get_neighbors · get_community
god_nodes · graph_stats · shortest_path
list_prs · get_pr_impact · triage_prs
```

MCP Resource 6개도 제공: `graphify://report`, `graphify://stats`, `graphify://god-nodes`, `graphify://surprises`, `graphify://audit`, `graphify://questions`

### ④ Hook — 자동 개입
```bash
graphify claude install
```
`PreToolUse` 훅이 설치되어, AI가 `grep`을 시도하면 훅이 먼저 개입해 "graphify query를 쓰라"고 유도합니다.

---

## 5. API 토큰이 필요한가

### API 키가 **불필요한** 작업 (대부분)

| 작업 | API 키 | 이유 |
|---|---|---|
| 코드 파싱 | 불필요 | tree-sitter AST — 순수 로컬 문법 분석 |
| `query` / `path` / `explain` | 불필요 | 기존 graph.json 탐색만 수행 |
| 커뮤니티 분할 (Leiden) | 불필요 | 순수 수학 알고리즘 |
| god nodes / 통계 | 불필요 | 그래프 계산 |
| 영상·오디오 → 텍스트 | 불필요 | faster-whisper 로컬 실행 |

```bash
graphify extract . --code-only    # 완전 오프라인, 100% 무료
```

실제 리포트 출력 예시:
```
Token cost: 0 input · 0 output
```

### API 키가 **필요한** 작업

| 작업 | 이유 |
|---|---|
| 문서(.md), PDF, 이미지 분석 | 의미 파악은 LLM 필요 |
| 커뮤니티 이름 자동 생성 | "Community 0" → "인증 모듈" 변환 |
| `--dedup-llm`, `prs --triage` | LLM 판단 필요 |

### 우회 방법 3가지

`llm.py` 분석 결과, 백엔드 9종 지원:
`Gemini · Kimi · Claude · OpenAI · DeepSeek · Azure · Bedrock · Ollama · claude-cli`

```bash
# 방법 1: Ollama — 완전 로컬, 키 불필요
graphify extract ./docs --backend ollama

# 방법 2: claude-cli — 구독료로 커버, 별도 키 불필요
graphify extract ./docs --backend claude-cli

# 방법 3: IDE 안에서 /graphify 실행 — 세션 모델이 대신 처리
```

> **결론: 코드만 분석한다면 API 키 0개로 영구 사용 가능합니다.**

---

## 6. 내부 구조 분석

### 파이프라인 (ARCHITECTURE.md 기준)

```
detect() → extract() → build() → cluster() → analyze → report → export
 파일스캔    파싱        그래프     커뮤니티     분석      리포트    내보내기
           노드/엣지     생성       분할
```

### 주요 모듈

| 모듈 | 줄 수 | 역할 |
|---|---|---|
| `extract.py` | 7,838 | 파서 디스패처 |
| `extractors/engine.py` | 6,509 | 파싱 엔진 |
| `cli.py` | 4,742 | CLI 진입점 |
| `llm.py` | 3,544 | LLM 백엔드 추상화 |
| `extractors/resolution.py` | 3,521 | 심볼 해석 (가장 어려운 부분) |
| `detect.py` | 2,566 | 파일 스캔 |
| `serve.py` | 2,508 | MCP 서버 |
| `install.py` | 2,366 | 플랫폼별 스킬 설치 |
| **전체** | **67,726** | Python 소스 총합 |

`extractors/` 아래에 언어별 파서 30개 파일 (rust, go, csharp, sql, terraform, fortran, commonlisp 등)

### 신뢰도 태그 (핵심 차별점)

모든 엣지에 태그가 붙습니다.

| 태그 | 의미 |
|---|---|
| `EXTRACTED` | 소스에 명시적으로 존재 (import문, 직접 호출) — 확실 |
| `INFERRED` | graphify가 추론 (호출 그래프 2차 패스) — 추정 |
| `AMBIGUOUS` | 불확실, 사람 검토 필요 |

### 실제 출력 예시 (worked/httpx)

```
## Summary
- 144 nodes · 330 edges · 6 communities detected
- Extraction: 53% EXTRACTED · 47% INFERRED · 0% AMBIGUOUS
- Token cost: 0 input · 0 output

## God Nodes (most connected)
1. Client       - 26 edges
2. AsyncClient  - 25 edges
3. Response     - 24 edges
```

---

## 7. 왜 유명한가

> 참고: 이 세션에서는 별(star) 개수를 직접 확인할 수 없었습니다. 아래는 저장소 내부 근거에 기반한 추론입니다.

| 이유 | 근거 |
|---|---|
| **타이밍** | "AI 컨텍스트 한계" = 2025~2026 최대 이슈. RAG 아닌 그래프 접근 |
| **토큰 0원** | 경쟁 도구는 임베딩 비용 발생. "무료 + 로컬"은 개발자 최애 문구 |
| **벤치마크 도발** | `BENCHMARKS.md`에서 mem0, supermemory와 직접 비교 (LOCOMO recall@10: 0.497 vs 0.048 / 0.149) |
| **강력한 마케팅** | YC S26 배지, Trendshift 배지, README 35개 언어 번역, Discord/YouTube/LinkedIn 운영 |
| **낮은 진입장벽** | 설치 2줄 → 30초 만에 시각적 결과물 → SNS 공유 유도 |

### 냉정한 평가

- 별 개수 ≠ 실사용률. "신기하다" 하고 별만 누르는 케이스 다수
- 버전 0.9.58 (v1 미만) — 안정화 진행 중
- 오픈소스는 유료 플랫폼(`app.graphify.com`)의 유입 깔때기 역할
- 단, 67,700줄 + 테스트 273개는 진지한 개발의 증거
- QA 정확도 지표는 supermemory에 근소하게 뒤짐 (유리한 지표를 강조한 것은 사실)

---

## 8. 로컬 에이전트 구축에 도움이 되는가

**매우 도움이 됩니다.** graphify는 사실상 로컬 에이전트용 부품입니다.

### 핵심 가치: 컨텍스트 압축

```
기존:     코드 전체 50,000 토큰 → 로컬 모델(8K~32K) 초과
graphify: 질문 → 관련 서브그래프 1,500 토큰 → 로컬 모델도 처리 가능
```

```bash
graphify query "..." --budget 1500     # 토큰 예산 직접 지정
```

### 바로 사용 가능한 부품

**① MCP 서버 — 코드 작성 없이 연결**
```bash
graphify serve    # stdio MCP 서버
```
에이전트가 MCP 클라이언트라면 `query_graph`, `shortest_path` 등을 즉시 사용 가능.

**② HTTP 서버 모드**
`serve_http(graph_path, host, port)` — 팀/다중 에이전트가 하나의 그래프 공유.

**③ Python 라이브러리로 직접 import**
```python
from graphify.extract import extract
from graphify.build import build

result = extract(paths, root=Path(".").resolve())
G = build([result])     # NetworkX 그래프 반환
```

**④ Ollama 백엔드 — 완전 오프라인 에이전트**
```bash
graphify extract ./docs --backend ollama
```
인터넷 없이 전체 파이프라인 동작. 에어갭(폐쇄망) 환경에서도 사용 가능.

### 보너스: 에이전트 메모리 기능 내장

```bash
graphify save-result --question "Q" --answer "A" --outcome useful
                     # outcome ∈ useful | dead_end | corrected
graphify reflect     # 기록을 집계해 LESSONS.md 생성
```

이후 `explain` / `query` 실행 시 **"Lesson:"** 힌트가 함께 표시되며,
소스가 변경되면 `"code changed — re-verify"` 태그까지 붙습니다.

> **RAG 대체용 "그래프 기반 검색 레이어" 완제품**으로 사용 가능합니다.

---

## 9. React / PHP로 만들 수 있는가

| | React | PHP | Node/TS |
|---|---|---|---|
| 뷰어 (그래프 화면) | 최고 | 가능 | 좋음 |
| 파서 (코드 분석) | 불가 | 사실상 불가 | 가능 |
| 종합 추천 | 뷰어만 | 비추천 | 유일한 현실적 대안 |

### React

- **파서로는 불가.** 브라우저 UI 라이브러리이므로 파일시스템 스캔 불가.
- **뷰어로는 최적.** 현재 `graph.html`은 정적 HTML 1개(`exporters/html.py`).
  `graph.json`이 이미 완성된 데이터이므로, React + `react-force-graph` / Cytoscape.js로
  훨씬 나은 뷰어(검색·필터·북마크·비교뷰·팀 공유)를 만들 수 있습니다.

> **가장 현실적인 진입점: 파서는 graphify 그대로 쓰고, React로 뷰어만 제작.**

### PHP

막히는 지점 3가지:
1. **tree-sitter 바인딩 빈약** — 37개 언어 문법 로드 사실상 불가
2. **그래프 라이브러리 부재** — NetworkX 대체품 없음, Leiden 직접 구현 필요
3. **장시간 실행 부적합** — 요청-응답 모델이라 대형 저장소 스캔에 부적합

단, 이 조합은 훌륭합니다:
```
파싱   = Python graphify (그대로)
         ↓ graph.json
웹 UI  = Laravel + Inertia/React
```
> **"PHP로 파서를 만들지 말고, PHP로 서비스를 만들 것."**

### Node / TypeScript (직접 만든다면 이것)

`web-tree-sitter`(WASM)가 있어 브라우저/Node에서 tree-sitter 실제 동작 가능.

```
web-tree-sitter                  → 파싱
graphology                       → 그래프 (NetworkX 대체)
graphology-communities-louvain   → 클러스터링
react-force-graph                → 시각화
```

단, **1~2개 언어만**으로 시작해야 합니다. 37개 언어는 비현실적입니다.

---

## 10. 라이선스 체크

```
라이선스: Apache License 2.0
저작권: Copyright 2026 Safi Shamsi and the Graphify contributors
(일부 기여분은 MIT로 남아있으며 LICENSE-MIT에 보존)
```

### Apache-2.0에서 가능한 것

| 가능 | 설명 |
|---|---|
| 상업적 이용 | 유료 판매 가능 |
| 수정 | 자유롭게 수정 가능 |
| 재배포 | 자사 제품에 포함 가능 |
| **소스 비공개** | 수정본 공개 의무 없음 (GPL과의 결정적 차이) |
| SaaS 서비스 | 웹서비스로 구독료 수취 가능 |
| 특허 라이선스 | 기여자 특허 사용권 포함 |

### 반드시 지켜야 할 것

1. `LICENSE` 파일 포함 (Apache-2.0 원문)
2. `NOTICE` 파일 보존 (저작권 표기 유지)
3. 변경사항 명시 ("modified by ...")
4. MIT 부분도 고지 (`LICENSE-MIT` 포함)

권장 구성:
```
내제품/
├── LICENSE
└── THIRD_PARTY_NOTICES.md
    └─ "Graphify (Apache-2.0), Copyright 2026 Safi Shamsi
        and the Graphify contributors. Modified: 익스트랙터 추가."
        + Apache-2.0 전문 + MIT 전문
```

### 금지 사항

**"Graphify" 상표 사용 금지** — Apache-2.0 제6조는 상표권을 명시적으로 제외합니다.

| 금지 | 안전 |
|---|---|
| "Graphify Korea" | "○○○ (Powered by Graphify)" |
| "GraphifyPro" | "Graphify 기반으로 제작" |
| `graphify-kr.com` | 완전히 다른 브랜드명 |

또한 PyPI에 유사 패키지명 등록 금지 (README에 이미 경고 존재).

---

## 11. 수익화 아이디어

### 경쟁 지형: 본가가 노리는 시장

`README.md:835` "graphify Enterprise" 섹션:
> *"the **always-on layer** ... applies the same graph approach to your **entire working context: meetings, files, docs, and code**, updating continuously in the background."*

```
본가 타겟 = 코드 + 회의록 + 문서 + 파일 전부
          = 항상 켜진 백그라운드 자동 갱신
          = 글로벌 B2B SaaS
```

**피해야 할 전장**

| 회피 | 이유 |
|---|---|
| 범용 코드 그래프 SaaS | 본가 본진. YC 자본 + 풀타임 팀 |
| 회의록+문서 통합 메모리 | 본가 Enterprise가 정확히 이것 |
| 그래프 시각화 툴 자체 | 무료 대안 다수 (Gephi, yEd) |

**노려야 할 전장**

```
본가가 안 할 것 = 기회
├─ 한국 시장 특화 (언어, 문화, SI 구조)
├─ 온프레미스 / 폐쇄망 (금융·공공·방산)
├─ 특정 프레임워크 심층 지원
├─ 사람이 해석해주는 서비스 (컨설팅)
└─ GitHub 외 플랫폼 (GitLab, Bitbucket)
```

### 코드에서 발견한 빈틈

| # | 빈틈 | 근거 |
|---|---|---|
| ① | **GitHub 외 미지원** | `prs.py:141` `_gh()` — `gh` CLI 하드코딩. GitLab/Bitbucket/Gitea 없음 |
| ② | **웹 UI 부재** | `exporters/html.py`는 정적 HTML 1개. 로그인/권한/공유/히스토리 전무 |
| ③ | **프레임워크 미인식** | `extractors/` 30개가 전부 언어 단위. 라우트→컨트롤러→모델 체인 모름 |
| ④ | **시계열 추적 없음** | `graph_diff()`는 있으나 저장·추적·시각화 없음 |
| ⑤ | **CI 통합 패키지 없음** | GitHub Action / GitLab CI 템플릿 미제공 |

### 아이디어 6개

> 아래 가격은 추정치이며, 실제 시장 검증이 필요합니다.

#### 1. 레거시 코드 진단 컨설팅 ★★★★★

**파는 것:** graphify로 그래프 뽑고 **사람이 해석한 진단 보고서 + 발표**

**타겟:** "10년 된 프로젝트 / 문서 0개 / 제작자 전원 퇴사 / 리뉴얼 필요" 상태의 국내 SI·SM 기업

**보고서 구성 (기존 기능만으로 가능):**

| 섹션 | 사용 기능 | 경영진 메시지 |
|---|---|---|
| 순환 의존성 지도 | `find_import_cycles()` | "여기가 얽혀서 수정이 느립니다" |
| 위험한 God Node | `god_nodes()` | "이 5개가 전체의 40%를 붙잡고 있습니다" |
| 고립된 죽은 코드 | 그래프 고립 노드 | "30% 삭제 가능합니다" |
| 모듈 재편 제안 | `cluster()` | "4개 팀으로 나누세요" |
| 리팩토링 우선순위 | 영향도 계산 | "1순위, 2순위, 3순위" |

| 패키지 | 범위 | 추정가 |
|---|---|---|
| 라이트 | 저장소 1개, PDF 보고서 | 150~300만원 |
| 스탠다드 | 3~5개 + 발표 + Q&A | 500~1,000만원 |
| 풀 | 위 + 3개월 리팩토링 동행 | 2,000만원~ |

- 장점: 초기 투자 ≈ 0, 본가와 미충돌, 빠른 현금 흐름, 실고객 문제 학습
- 단점: 확장성 낮음(시간=매출 한계), 영업 필요, 고객별 커스텀

**첫 30일:** 1주 오픈소스 3개 샘플 보고서 → 2주 템플릿 정형화 → 3주 지인 회사 무료 진단 → 4주 사례 공개

---

#### 2. 프레임워크 특화 익스트랙터 ★★★★★

**파는 것:** graphify 위에 얹는 프레임워크 전용 플러그인 (빈틈 ③ 공략)

```
routes/web.php → Route::get('/users', UserController@index)
                    ↓ 그래프 엣지로 변환
UserController::index() --uses--> User (Eloquent Model)
                        --renders--> users.blade.php
User --maps_to--> users 테이블 (migration에서 추출)
```

| 프레임워크 | 시장 | 난이도 | 평가 |
|---|---|---|---|
| **Spring Boot** | 국내 대기업/금융 압도적 | 상 | ★★★★★ 돈이 여기 있음 |
| **Laravel** | 국내 중소 SI | 중 | ★★★★ 규약이 강해 파싱 용이 |
| **NestJS** | 중견 | 하 | ★★★★ 데코레이터라 파싱 쉬움 |
| **Next.js App Router** | 스타트업 | 중 | ★★★ 경쟁 많음 |

기존 `blade.py`, `razor.py` 패턴 참고 가능.

**수익 모델:** (A) 오픈소스 공개 → 컨설팅 연결 / (B) 유료 플러그인(연 30~100만원/팀) / (C) 기업 커스텀 개발(건당 500만원~)

- 장점: 기술적으로 명확, 본가가 안 할 영역, 재사용 가능
- 단점: 프레임워크당 1~2개월, 버전 변경 시 유지보수

---

#### 3. GitLab 전용 팀 대시보드 ★★★★

**파는 것:** 빈틈 ① + ② 동시 공략

```
graphify (파싱 엔진 그대로) → graph.json → 자체 웹 대시보드
   ├─ GitLab MR 자동 영향도 코멘트
   ├─ 팀별 모듈 소유권 지도
   ├─ 복잡도 추이 그래프
   └─ 신입 온보딩 위키 자동생성
```

**왜 GitLab:** `prs.py`가 GitHub 전용 → 빈 시장 / 국내 기업 자체호스팅 GitLab 비율 높음 / 온프레미스 판매 가능 = 단가 높음

| 티어 | 대상 | 추정가 |
|---|---|---|
| 무료 | 개인/오픈소스 | 0원 (유입용) |
| 팀 | 10~50명 | 월 20~50만원 |
| 온프레미스 | 금융/공공 | 연 1,000~3,000만원 |

폐쇄망 고객은 클라우드 SaaS 사용 불가 → graphify의 Ollama 완전 오프라인 동작이 강점.

- 장점: 빈 시장, 온프레미스 고단가·저이탈, React로 구현 가능
- 단점: 개발 3~6개월, 기업 영업 사이클 6~12개월, 본가의 GitLab 지원 시 리스크

---

#### 4. PR/MR 영향도 봇 ★★★★

**파는 것:** PR 등록 시 자동 코멘트

```markdown
그래프 영향도 분석

이 PR은 auth.py, session.py를 수정했습니다.
영향받는 노드: 47개 (전체의 8%)
건드리는 커뮤니티: 인증(2), 세션관리(5)

주의: SessionManager는 god node입니다 (연결 23개)
충돌 위험: PR #89도 같은 커뮤니티를 수정 중
추천 리뷰어: @철수 (인증 모듈 최다 커밋)
```

**핵심 로직이 이미 존재합니다:**
```python
compute_pr_impact(files, G)    # prs.py:260
affected_nodes(...)            # affected.py:190
attach_graph_impact(...)       # prs.py:360
```
→ 만들 것은 "포장"뿐. GitHub App / GitLab 웹훅으로 감싸기만 하면 됩니다.

**가격:** 무료(공개 저장소) / $10~20월(저장소 5개) / $50~100월(팀 무제한)

- 장점: 개발 범위 작음(1~2개월), 핵심 로직 존재, PR 코멘트가 곧 광고, MRR 축적
- 단점: 단가 낮음, 서버 비용, 유사 봇 존재

---

#### 5. 교육 콘텐츠 / 인포프로덕트 ★★★

```
"오픈소스 해부" 시리즈
├─ 유튜브: "React 소스코드를 그래프로 뜯어봤다" (유입)
├─ 뉴스레터: 주 1회 유명 프로젝트 분석
├─ 유료 강의: "레거시 코드 파악하는 법"
└─ 기업 사내교육: 회당 100~300만원 ← 가장 알짜
```

`worked/` 폴더가 이미 이 포맷(httpx, karpathy-repos 분석 예시).

- 장점: 개발 불필요, 개인 브랜딩 → 다른 아이디어의 영업 채널
- 단점: 누적 게임이라 시간 소요

---

#### 6. 문서 자동화 (위키 동기화) ★★★

```
코드 커밋 → git hook → graphify update → wiki.py → Confluence/Notion 자동 푸시
```

`wiki.py`의 기존 기능: `to_wiki()`, `_community_article()`, `_god_node_article()`, `_cross_community_links()`

- 장점: 부품 존재, 기업 구매 명분 명확("문서화 의무")
- 단점: 자동 생성 문서 품질 회의론, API 연동 잡일 다수
- → 단독보다 아이디어 3의 기능 하나로 포함하는 것이 유리

---

### 종합 비교

| 아이디어 | 초기투자 | 첫수익까지 | 확장성 | 본가충돌 | 종합 |
|---|---|---|---|---|---|
| 1. 진단 컨설팅 | 거의 0 | 1~2개월 | 낮음 | 없음 | ★★★★★ |
| 2. 프레임워크 익스트랙터 | 1~2개월 | 3개월 | 중간 | 없음 | ★★★★★ |
| 3. GitLab 대시보드 | 3~6개월 | 6~12개월 | 높음 | 중간 | ★★★★ |
| 4. PR 영향도 봇 | 1~2개월 | 3개월 | 높음 | 중간 | ★★★★ |
| 5. 교육 콘텐츠 | 0 | 6개월+ | 중간 | 없음 | ★★★ |
| 6. 위키 자동화 | 2~4주 | 3개월 | 중간 | 중간 | ★★★ |

### 하지 말아야 할 것

| 금지 | 이유 |
|---|---|
| 본체 포크 후 이름만 바꿔 판매 | 본가가 거의 매일 커밋 중. 추격 불가 |
| "Graphify" 이름/로고 사용 | 상표 분쟁 |
| NOTICE 파일 삭제 | 라이선스 위반 |
| 범용 SaaS 정면승부 | 자본력 격차 |
| 37개 언어 전부 지원 시도 | 1개 프레임워크 깊게가 승리 |
| "AI가 다 해줍니다" 마케팅 | graphify는 AI를 안 쓰는 것이 강점 |

### 포지셔닝

```
X  "AI가 코드를 분석해드립니다"
O  "AI 없이, 비용 0원으로, 코드가 외부로 나가지 않게 분석합니다"

X  "코드 그래프 만들어드립니다"
O  "퇴사자가 남긴 레거시, 3일 만에 지도로 만들어드립니다"
```

> **기능이 아니라 "고통"을 판매해야 합니다.**

---

## 12. 실행 로드맵

### 추천 전략: "컨설팅 → 제품화"

```
Phase 1 (0~3개월) — 현금 + 학습
  아이디어 1(컨설팅) + 5(콘텐츠)
  → 수익을 내면서 실제 고객 문제를 학습
  → "다들 Spring 레거시로 고생한다" 같은 발견

Phase 2 (3~9개월) — 반복 자동화
  컨설팅 중 반복 작업을 도구화
  → 아이디어 2(Spring 익스트랙터) 자연 발생
  → 이미 고객이 있으므로 만들면 바로 판매

Phase 3 (9~18개월) — 제품화
  도구가 쌓이면 웹으로 포장
  → 아이디어 3/4 (대시보드 + 봇)
  → 기존 고객이 첫 구독자
```

**이 순서인 이유:** 제품을 6개월 만들고 아무도 사지 않는 실패를 피하고,
컨설팅이 시장조사 비용을 고객이 대신 부담하는 구조를 만듭니다.

### 이번 주 실행 항목

```bash
# ① 내 프로젝트에 직접 실행 (30분)
uv tool install graphifyy
graphify extract . --code-only
open graphify-out/graph.html

# ② 유명 오픈소스 분석해 감 잡기 (1시간)
graphify clone https://github.com/laravel/framework
# → GRAPH_REPORT.md 읽고 보고서로 쓸 수 있을지 판단

# ③ 핵심 로직 읽기 (2시간)
#   graphify/prs.py:260       compute_pr_impact()
#   graphify/affected.py:190  affected_nodes()
#   → 아이디어 4(봇) 구현 가능성 판단
```

---

## 부록: 자주 쓰는 명령어

```bash
# 빌드
graphify extract . --code-only          # 코드만, 오프라인, 무료
graphify extract . --mode deep          # 더 풍부한 관계 추출
graphify extract . --backend ollama     # 로컬 LLM 사용

# 조회
graphify query "질문" --budget 1500
graphify path "A" "B"
graphify explain "노드명"

# 유지
graphify update .
graphify hook install
graphify watch ./src

# 내보내기
graphify export callflow-html           # Mermaid 아키텍처 다이어그램
/graphify . --wiki                      # 마크다운 위키
/graphify . --obsidian                  # Obsidian 볼트
/graphify . --graphml                   # Gephi / yEd
/graphify . --neo4j                     # Neo4j Cypher

# 통합
graphify serve                          # MCP stdio 서버
graphify claude install                 # 항상 그래프 우선 사용하도록 설정
graphify global add graphify-out/graph.json --as myrepo   # 크로스 프로젝트 그래프

# 제거
graphify uninstall                      # 전 플랫폼 일괄 제거
graphify uninstall --purge              # graphify-out/ 까지 삭제
```

---

*정리 기준일: 2026-09-11 / 분석 대상 버전: graphify 0.9.58*
