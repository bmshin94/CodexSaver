# CodexSaver 분석 & 활용 가이드 (한국어)

> 이 문서는 CodexSaver 저장소를 직접 분석하고, 설치/사용법·정체성·수익화·포팅 가능성까지
> 정리한 대화 기록을 문서화한 것입니다.

- **원본 저장소**: https://github.com/fendouai/CodexSaver
- **이 저장소(포크)**: https://github.com/bmshin94/CodexSaver
- **참고 통계**: https://trendshift.io/repositories/29293
- **참고 문서**: https://deepwiki.com/fendouai/CodexSaver
- 작성일: 2026-09-18
- 분석 대상 버전: `0.3.6`

---

## 목차

1. [CodexSaver란 무엇인가](#1-codexsaver란-무엇인가)
2. [쉬운 설명 (비유)](#2-쉬운-설명-비유)
3. [상세 Q&A](#3-상세-qa)
4. [수익화 아이디어](#4-수익화-아이디어)
5. [요약 치트시트](#5-요약-치트시트)

---

## 1. CodexSaver란 무엇인가

### 1.1 정체

**CodexSaver** = OpenAI Codex(코딩 에이전트)에 붙이는 **비용 절감 라우터 MCP 서버**.

슬로건: *"Make Codex cheaper without making it dumber"*
(Codex를 멍청하게 만들지 않으면서 저렴하게 쓰기)

### 1.2 핵심 아이디어

코딩 작업은 두 종류가 섞여 있다.

| 종류 | 예시 | 담당 |
|---|---|---|
| **비싼 판단** | 아키텍처, 인증/결제 로직, DB 마이그레이션, 최종 리뷰 | Codex (고비용 모델) |
| **싼 노가다** | 코드 설명, 파일 검색, 테스트 작성, 문서 갱신, 린트 수정 | DeepSeek / Pi Agent / 로컬 모델 |

현재는 둘 다 Codex가 처리해서 비용이 낭비된다.
CodexSaver가 중간에서 **교통정리**를 해준다.

```text
Use the expensive model for judgment.
Use the cheaper model for volume.
Never confuse the two.
```

### 1.3 저장소 구조 (실측)

```
CodexSaver/
├── codexsaver_mcp.py       # MCP 서버 본체 (stdio JSON-RPC 2.0)
├── cli.py                  # codexsaver CLI (376줄)
├── codexsaver/             # 핵심 엔진 (약 3,600줄)
│   ├── router.py           # 작업 분류 + 위험도 판정 (83줄)
│   ├── agent_registry.py   # Agent Card(.agent-card.json) 탐색
│   ├── agent_router.py     # 워커 가중치 점수 라우팅
│   ├── orchestrator.py     # v3 오케스트레이션 (922줄, 최대 파일)
│   ├── work_packet.py      # v2 샌드박스 패치 실행 (581줄)
│   ├── installer.py        # ~/.codex/config.toml 자동 설치 (499줄)
│   ├── verifier.py         # 결과 검증 (보호 경로 / 명령어)
│   ├── policy.py           # 금지 경로 / 허용 명령 정책
│   ├── cost.py             # 절감률 추정
│   ├── ledger.py           # 실행 기록
│   └── pi_agent.py         # Pi Agent CLI 어댑터
├── tests/                  # 테스트 13개 파일
├── docs/benchmarks/        # 벤치마크 리포트 12세트 (json + md)
├── AGENTS.md               # Codex에게 주는 위임 규칙서
└── .codex/config.toml      # MCP 서버 등록 설정
```

> **주목할 점**: `pyproject.toml`의 `dependencies = []` — **외부 의존성 0개**.
> 파이썬 표준 라이브러리만 사용해서 설치가 가볍고 깨질 위험이 적다.

### 1.4 router.py 실제 로직

```python
PROTECTED_PATH_KEYWORDS = [
    "auth", "oauth", "jwt", "session", "security", "permission", "rbac",
    "payment", "payments", "billing", "invoice", "migration", "migrations",
    "schema", "infra", "terraform", ".github/workflows", ".env", "secret",
    "key", "token",
]

DELEGATABLE = {
    "code_search", "explain", "write_tests", "fix_lint",
    "docs", "boilerplate", "simple_refactor",
}
```

판정 흐름:

1. **작업 분류(classify)** — "explain/summarize" → `explain`, "test" → `write_tests` 등
2. **위험도 판정(risk)** — 경로·지시문에 `auth`, `payment`, `secret` 등이 있으면 `high`
3. **결정(decide)** — `high`면 Codex, `unknown`(모호)이면 Codex, 안전하면 워커로 위임

흥미로운 비대칭:

- `Explain auth code` → **위임 가능** (읽기는 싸도 됨)
- `Refactor auth service` → **Codex 고정** (쓰기는 위험)

### 1.5 버전별 진화

| 버전 | 추가된 것 | 성숙도 |
|---|---|---|
| v1 | 단순 위임 (`delegate_task`) | 기본 |
| v2 | **Work Packet** — 허용 파일/금지 경로/수락 기준 + 샌드박스 패치 검증 | 성숙 |
| v3 | **오케스트레이션** — 전문가(explainer, perf_reviewer) 병렬 실행 | 읽기전용 확립 |
| v3.4 | **액션 단위** 위험 판정 + 부분 핸드오프 | 확립 |
| v3.5 | 패치 lint + **노드 단위 자동 수리(repair)** | 확립 |
| v3.6 | **Agent Card 발견** + 가중치 라우팅, Pi Agent 기본 워커 | 현재 (0.3.6) |

### 1.6 벤치마크 (문서 기록 기준)

- **v2 (5개 작업)**: 5/5 성공, 45% 절감, 이미 만족된 작업은 100% 절감(0.03초)
- **v3.6 Pi Agent 실측 (읽기전용 5개)**: 5/5 성공,
  워커 비용 `$0.00968` vs Codex 추정 `$0.4796` → **약 98% 절감**, 품질 점수 `1.0`
- **v3.6 패치 성공률**: `0.4` → `0.8` (repair 효과로 2배)
- **정직한 부분**: 프로젝트 벤치마크 5개 중 2개만 성공,
  패치가 무거운 3개는 `needs_codex`로 보수적 후퇴

### 1.7 나에게 주는 이점

1. **비용 절감** — 읽기전용 작업 비중이 높을수록 절감 폭이 크다
2. **사고 방지** — auth/결제/마이그레이션은 절대 저가 모델로 넘기지 않음
3. **속도 향상** — 설명 + 성능리뷰 병렬 → `sum` 대신 `max(단일 실행시간)`
4. **에이전트 설계 교과서** — 라우팅/샌드박스/검증/폴백 패턴의 레퍼런스

---

## 2. 쉬운 설명 (비유)

### 2.1 식당 비유

미슐랭 셰프(= Codex)를 시급 10만원에 고용했는데, 그 셰프가 지금 하는 일:

- 감자 깎기
- 설거지
- 메뉴판 오타 고치기
- **진짜 요리** ← 이것만 하면 되는데

**CodexSaver = 똑똑한 매니저**. 주방 입구에서 주문표를 보고 나눠준다.

- "감자 깎기 → 알바생(저가 모델)"
- "소스 배합 비율 결정 → 셰프님"
- "애매하다? → 그냥 셰프님께" (안전 제일)

### 2.2 세 가지 상태

| 상태 | 의미 | 쉽게 |
|---|---|---|
| `preview` | 라우팅 미리보기만 | "이거 알바 줄 건데 괜찮아?" (비용 0) |
| `delegated_execution` | 위임 실행 완료 | "알바가 끝냈어요" |
| `codex_takeover` | Codex가 가져감 | "위험해서 셰프님께 넘겼어요" |

### 2.3 워커를 믿지 않는 구조 (핵심)

```
1. 워커가 결과 제출
2. → 임시 샌드박스에 먼저 적용
3. → 검사: 허용 파일만 건드렸나? 금지 경로는? diff 크기는?
4. → 실패하면 repair(수리 컨텍스트 주고 재시도)
5. → 또 실패하면 needs_codex (상위 모델에게 반환)
6. → 통과해야만 Codex 최종 리뷰 후 반영
```

그래서 README의 이 문장이 핵심이다:

> **"YOLO 자동 편집 봇"이 아니다.**

### 2.4 한 문장 요약

> **비싼 AI는 판단만, 싼 AI는 잡일만. 잡일 결과는 반드시 검사하고, 애매하면 무조건 비싼 AI에게.**

---

## 3. 상세 Q&A

### 3.1 설치 및 사용법

#### 사전 준비물

| 필요한 것 | 용도 | 필수 여부 |
|---|---|---|
| Python 3.10+ | CLI + MCP 서버 | 필수 |
| Node.js + npm | Pi Agent 설치 | v3.6 사용 시 |
| Pi Agent CLI | 기본 워커 | v3.6 사용 시 |
| DeepSeek API 키 | 워커 모델 호출 | 사실상 필수 (로컬 모델이면 불필요) |

#### 전역 설치 (권장)

```bash
git clone https://github.com/bmshin94/CodexSaver
cd CodexSaver

python -m pip install -e .                                 # ① CLI 설치
npm install -g @earendil-works/pi-coding-agent             # ② Pi Agent 설치
codexsaver auth set --provider deepseek --api-key YOUR_KEY # ③ 키 저장 (1회)
codexsaver install                                         # ④ 전역 MCP 등록
codexsaver doctor --workspace .                            # ⑤ 상태 점검
codexsaver agents list --workspace .                       # ⑥ 워커 발견 확인
```

- ③은 키를 두 곳에 저장: `~/.codexsaver/config.json`, `~/.pi/agent/auth.json` (둘 다 `0600`)
- ④는 `~/.codex/config.toml`에 MCP 항목 기록:

```toml
[mcp_servers.codexsaver]
command = "python"
args = ["/Users/you/.codexsaver/codexsaver_mcp.py"]
startup_timeout_sec = 10
tool_timeout_sec = 120
```

- ⑤에서 `CodexSaver is ready`가 뜨면 성공

#### 레포 로컬 설치

```bash
codexsaver install --project   # .codex/config.toml 만 생성
```

#### 사용법 A — Codex에게 말로 지시

```text
Use CodexSaver for safe low-risk tasks.
Add unit tests for user service.
```

#### 사용법 B — CLI 직접 호출

```bash
# 위임 미리보기
codexsaver delegate "Explain the routing logic" --files codexsaver/router.py --dry-run

# v2 바운디드 패치
codexsaver work-packet "Create docs/example.md with one sentence." \
  --files README.md --allowed-file docs/example.md \
  --acceptance "docs/example.md exists in sandbox" --workspace .

# v3 오케스트레이션 (설명 + 성능리뷰 동시)
codexsaver orchestrate "Explain config loader logic and review performance" \
  --files codexsaver/config.py

# 단일 전문가
codexsaver specialist explainer "Explain this module" --files codexsaver/config.py
```

#### 워커 출력 압축

```bash
codexsaver compression set --enabled true --level full
```

레벨: `lite` / `full` / `ultra` / `wenyan`(문언문).
워커 응답만 짧아지고 Codex 최종 답변은 그대로다.

#### 윈도우 함정

`C:\Users\...` 경로에서 TOML이 `\U`를 유니코드 이스케이프로 해석해 실패한다.
최신 버전으로 재설치하거나 `C:/Users/...`(슬래시) 또는 `C:\\Users\\...`로 수정.

---

### 3.2 이건 플러그인? 스킬? MCP?

**정답: MCP 서버다.** (정석적인 stdio JSON-RPC 2.0 MCP 서버)

근거 — `codexsaver_mcp.py`:

```python
JSONRPC = "2.0"
def respond(id_, result=None, error=None):
    msg = {"jsonrpc": JSONRPC, "id": id_}
    print(json.dumps(msg, ensure_ascii=False), flush=True)
```

노출하는 MCP 툴 4개:

| 툴 | 역할 |
|---|---|
| `codexsaver.delegate_task` | v1 단순 위임 |
| `codexsaver.delegate_work_packet` | v2 바운디드 패치 |
| `codexsaver.orchestrate_task` | v3 작업 그래프 |
| `codexsaver.run_specialist` | 단일 전문가 실행 |

다만 실제로는 **하이브리드 구조**다:

| 구성요소 | 분류 | 정체 |
|---|---|---|
| `codexsaver_mcp.py` | **MCP 서버** | Codex가 호출하는 툴 |
| `AGENTS.md` | **스킬/지침** | "이런 건 위임, 저런 건 금지" 규칙서 |
| `codexsaver superpower install` | **플러그인 설치기** | `.codex/hooks.json` + 프롬프트 훅 |
| `cli.py` | **일반 CLI** | MCP 없이 단독 사용 가능 |

`superpower install` 프로필:

- `basic`: AGENTS.md에 관리 블록만 추가 (안전)
- `full`: 훅 스크립트 + `.codex/config.toml` 플래그까지 (침습적)

---

### 3.3 API 토큰을 꼭 써야 하나?

**경우에 따라 다르다. 완전 무료 경로도 있다.**

#### 프로바이더 매트릭스

| Provider | 기본 모델 | 키 필요 |
|---|---|---|
| `deepseek` | deepseek-chat | 필요 |
| `openai` | gpt-4o-mini | 필요 |
| `anthropic` | claude-3-5-haiku-latest | 필요 |
| `opencode-go` | deepseek-v4-flash | 필요 |
| `gemini` | gemini-2.0-flash | 필요 |
| `qwen` | qwen-plus | 필요 |
| **`ollama`** | llama3.1 | **불필요** |
| **`lmstudio`** | local-model | **불필요** |

#### 토큰 없이 쓰는 법

```bash
codexsaver auth set --provider ollama --model llama3.1
# 또는
codexsaver auth set --provider lmstudio --model local-model
```

로컬에 Ollama가 있으면 API 비용 0원으로 워커를 돌릴 수 있다.

#### 보안 체크

- 설정 파일 `~/.codexsaver/config.json`은 `0600`(본인만 읽기) 권한
- `doctor`는 키를 마스킹해서 출력
- 환경변수 임시 사용도 가능:

```bash
export CODEXSAVER_PROVIDER=deepseek
export CODEXSAVER_API_KEY=YOUR_KEY
```

- **주의**: 키를 절대 커밋하지 말 것. `--api-key`는 셸 히스토리에도 남는다.

#### 비용 추정 로직 (`codexsaver/cost.py`)

```python
if chars < 8_000:  return 45   # 45% 절감
if chars < 50_000: return 62   # 62% 절감
else:              return 70   # 70% 절감
```

컨텍스트가 클수록 절감폭이 커진다. 실측 벤치마크는 최대 98%까지 기록.

---

### 3.4 왜 GitHub에서 유명할까?

원본 저장소는 트렌딩에 올랐고 별 400개 이상을 받았다.

1. **타이밍** — 2025~2026년 개발자 최대 고통 = AI 코딩 비용
2. **슬로건** — "품질 포기 없이 저렴하게"라는 포지셔닝이 기존 대안과 차별됨
3. **숫자로 증명** — `docs/benchmarks/`에 JSON 원본 + MD 리포트 12세트 + SVG 그래프
4. **정직함** — README에 `2/5 성공`, `still maturing`, `not solved magic`처럼 약점을 먼저 공개
5. **의존성 0개** — 설치 마찰 최소화
6. **안전 설계** — 샌드박스 + 검증기 + 보호 경로 + Codex 폴백
7. **중국어 README** — DeepSeek 기반이라 중국 커뮤니티와 궁합이 좋음
8. **MCP 생태계 초기 진입** — 쓸만한 MCP 서버 자체가 희소, 카테고리 선점
9. **한 문장 설명 가능** — "비싼 모델은 판단, 싼 모델은 실행" (인용·바이럴 최적화)

---

### 3.5 로컬 에이전트 구축에 도움이 될까?

**매우 도움된다. 코드를 안 쓰고 읽기만 해도 가치가 있다.**

#### 바로 가져다 쓸 수 있는 패턴 6가지

**① 라우팅 계층 분리 (`router.py`, 83줄)**

```python
task_type = self.classify(instruction)      # 무엇을 하는 일인가
risk, hits = self.risk(instruction, files)  # 얼마나 위험한가
```

`classify`(무엇)와 `risk`(위험도)를 분리한 것이 핵심. 대부분 이걸 뭉쳐서 짜다가 스파게티가 된다.

**② 키워드 기반 보호 경로**

LLM에게 묻지 않고 문자열 매칭으로 위험을 감지 → 빠르고, 저렴하고, 결정론적.

**③ Work Packet 패턴 (`work_packet.py`, 581줄)**

```
goal / allowed_files / forbidden_paths /
acceptance_criteria / allowed_commands /
max_iterations / max_diff_lines
```

"알아서 해"가 아니라 **계약서를 쥐여주는** 방식.

**④ 샌드박스 → 검증 → 수리 → 폴백 루프**

```
실행 → 임시폴더 적용 → lint/verify
     → 실패 시 repair(수리 컨텍스트 재시도)
     → 재실패 시 needs_codex (상위 모델/사람에게)
```

**실패를 설계에 포함시킨 구조.** 자작 에이전트가 가장 많이 무너지는 지점이다.

**⑤ Agent Card + 가중치 라우팅 (`agent_router.py`)**

```json
{
  "id": "pi-agent-default",
  "capabilities": ["code_generation", "testing", "docs"],
  "languages": ["python"],
  "cost_weight": 0.1
}
```

| 점수 항목 | 가중치 |
|---|---:|
| 능력 매칭 | 0.40 |
| 과거 성공률 | 0.25 |
| 비용 | 0.20 |
| 현재 부하 | 0.10 |
| 컨텍스트 적합도 | 0.05 |

`if/else` 지옥 대신 점수제. 멀티 에이전트 설계에 그대로 적용 가능.

**⑥ Task Lifecycle (A2A 호환)**

```
submitted -> running -> completed
                     -> failed
                     -> timed_out
```

#### 내 프로젝트에 적용한다면

```
[내 앱]
   ↓
[라우터]  ← router.py 패턴
   ├─ 저위험 → 로컬 Ollama (무료)
   └─ 고위험/모호 → Claude/GPT (유료)
   ↓
[검증기]  ← verifier.py 패턴
   ↓
[샌드박스 적용]  ← work_packet.py 패턴
```

#### 한계

- Codex 전용 설계 (다른 툴에 쓰려면 개조 필요)
- 규칙 기반이라 키워드를 못 잡으면 뚫린다 (예: `auth` 대신 `signin`)
- 패치 오케스트레이션은 아직 미성숙 (본인들도 인정)

---

### 3.6 React나 PHP로 만들 수 있을까?

**가능하다. 오히려 기회가 더 클 수도 있다.**

핵심은 알고리즘이지 언어가 아니다: 문자열 매칭 + JSON-RPC over stdio +
파일 복사 + 서브프로세스. 게다가 의존성 0개라 포팅 난이도가 낮다.

#### A. Node.js / TypeScript 포팅 (가장 추천)

MCP 공식 SDK가 TS 퍼스트이고, `npx`로 설치 마찰이 0이다.

```bash
npx codexsaver-js install
```

```ts
// router.ts
const PROTECTED = ['auth', 'payment', 'migration', '.env', 'secret'] as const;

export function decide(instruction: string, files: string[]): RouteDecision {
  const text = instruction.toLowerCase();
  const paths = files.join('\n').toLowerCase();
  const hits = PROTECTED.filter(k => text.includes(k) || paths.includes(k));
  if (hits.length) return { route: 'codex', risk: 'high', hits };
  return { route: 'worker', risk: 'low', hits: [] };
}
```

```ts
// server.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new McpServer({ name: 'codexsaver', version: '1.0.0' });
server.tool('delegate_task', schema, async (args) => { /* ... */ });
await server.connect(new StdioServerTransport());
```

#### B. React 활용 (가장 큰 빈자리)

React로 MCP 서버를 만들 수는 없다(UI 라이브러리이므로).
그런데 **CodexSaver에 지금 가장 없는 것이 바로 UI**다.
원본 레포에 대시보드가 없고, 벤치마크 결과가 전부 `.md` / `.json`으로만 존재한다.

```
[Python/Node MCP 서버] ──ledger.json / SSE──→ [React 대시보드]
```

만들면 좋은 화면:

| 화면 | 내용 |
|---|---|
| 비용 대시보드 | 오늘/이번주 절감액, 누적 그래프 |
| 라우팅 로그 | 어떤 작업이 어디로 갔는지 실시간 스트림 |
| 워커 성능 | 성공률 / 지연시간 / repair 횟수 비교 |
| 정책 편집기 | 보호 키워드·허용 명령을 GUI로 편집 |
| 시뮬레이터 | 지시문 입력 → 라우팅 결과 미리보기 (토큰 0원) |

추천 스택: `Vite + React + TypeScript + TanStack Query + Recharts + shadcn/ui`
(`codexsaver/ledger.py`가 이미 있어 데이터 소스는 준비되어 있다.)

#### C. PHP 포팅 (틈새 블루오션)

```php
<?php
// codexsaver_mcp.php — stdio JSON-RPC
while (($line = fgets(STDIN)) !== false) {
    $req = json_decode($line, true);
    $res = handle($req);
    echo json_encode($res, JSON_UNESCAPED_UNICODE) . "\n";
    flush();
}
```

- `proc_open()` → 서브프로세스
- `curl` → API 호출
- Composer 패키지로 배포

**진짜 기회는 Laravel / WordPress 특화판이다.**

- Laravel: `migrations/`, `.env`, `config/`, `Auth::` → 자동 보호 경로
- WordPress: `wp-config.php`, 플러그인 업데이트 → 자동 차단
- PHP 커뮤니티에는 AI 코딩 툴이 거의 없다 = 경쟁이 적다

```bash
php artisan codexsaver:delegate "Add tests for UserService"
wp codexsaver route "Explain this plugin hook"
```

#### 언어별 비교

| 언어 | 난이도 | 강점 | 추천도 |
|---|---|---|---|
| TypeScript / Node | 쉬움 | 공식 SDK, npx 배포 | ★★★★★ |
| React (UI 전용) | 쉬움 | 아무도 안 만든 빈자리 | ★★★★★ |
| PHP | 보통 | Laravel / WP 블루오션 | ★★★★ |
| Go | 보통 | 단일 바이너리 배포 | ★★★ |
| Python | - | 이미 존재 | - |

---

## 4. 수익화 아이디어

### Tier 1 — 지금 당장 가능

#### 아이디어 1. CodexSaver Dashboard (React) ★★★★★

원본에 UI가 전혀 없고, "얼마나 아꼈는지" 보여주는 것이 결제 근거 그 자체다.

| 항목 | 내용 |
|---|---|
| 제품 | 라우팅 로그 + 절감액 실시간 대시보드 |
| 가격 | Free(로컬 1인) / Pro $9-19/월(히스토리·리포트) / Team $49/월 |
| 핵심 훅 | "이번 달 $347 아꼈습니다" 알림 |
| 난이도 | 낮음 (`ledger.py` 데이터가 이미 존재) |
| 기간 | 2-4주 MVP |

절감액이 구독료보다 크면 고민 없이 결제한다. ROI가 눈에 보인다.

#### 아이디어 2. Team Cost Control SaaS ★★★★★

개인은 안 써도 회사는 쓴다.

| 항목 | 내용 |
|---|---|
| 제품 | 팀 전체 AI 코딩 비용 통제 플랫폼 |
| 기능 | 조직 단위 정책 배포 / 개발자별 사용량 / 예산 알림 / 감사 로그 / SSO |
| 가격 | $15-30 / 개발자 / 월 |
| 규모 | 개발자 20명 팀 = 월 $400 → 연 $4,800 |

세일즈 한 줄:

> "개발자들 AI 요금 관리 안 되시죠? 정책 한 번 배포하면 auth/결제 코드는 자동 차단되고, 비용은 60% 줄어듭니다."

#### 아이디어 3. PHP / Laravel 특화판 ★★★★

| 항목 | 내용 |
|---|---|
| 제품 | `laravel-codexsaver` Composer 패키지 + WordPress 플러그인 |
| 수익 | Nova/Spark 스타일 $99 평생 라이선스 또는 $199 상용 |
| 이유 | PHP엔 AI 코딩 툴이 거의 없다 = 경쟁 0 |

---

### Tier 2 — 중기 (3-6개월)

#### 아이디어 4. Policy Marketplace ★★★★

"우리 회사 규칙에 맞는 라우팅 정책"을 판매.

- 금융권 팩 (PCI-DSS 경로 보호) — $199
- 헬스케어 팩 (HIPAA) — $199
- 프레임워크 팩 (Django / Rails / Next.js) — $29~
- 커스텀 제작 — $2,000+

규제 산업은 컴플라이언스에 돈을 아끼지 않는다.

#### 아이디어 5. "AI 비용 진단" 컨설팅 ★★★★

| 패키지 | 내용 | 가격 |
|---|---|---|
| 진단 | 현재 지출 분석 + 절감 리포트 | $500-1,500 |
| 구축 | CodexSaver 설치 + 정책 커스텀 | $3,000-8,000 |
| 운영 | 월간 리포트 + 튜닝 | $500-1,500/월 |

초기 자본 0원. 고객 2-3곳만 확보해도 월 수익이 발생한다.

#### 아이디어 6. Managed Worker Pool ★★★

API 키 설정이 번거로운 사용자를 위해 워커를 대신 운영.

- 사용량 기반 (1M 토큰당 $X, 원가 위 마진)
- 또는 월 $29 무제한(공정 사용)
- 셀프호스팅 옵션 병행 (엔터프라이즈)

---

### Tier 3 — 장기 / 큰 그림

#### 아이디어 7. Universal AI Router ★★★★★

Codex 전용을 벗어나 모든 AI 코딩 툴에 연결.

```
Claude Code / Cursor / Copilot / Windsurf / Codex
              ↓
    [Universal Cost Router]
              ↓
   로컬 / 저가 API / 고가 API
```

카테고리 선점 = "AI 비용의 Cloudflare" 포지션. 투자 유치 가능한 규모의 스토리.

#### 아이디어 8. Open Core 모델 ★★★★ (가장 현실적인 구조)

| 구분 | 내용 |
|---|---|
| 오픈소스 | 코어 라우터, CLI, MCP 서버 (= 마케팅) |
| 유료 | 대시보드, 팀 정책, SSO, 감사로그, 우선지원 |

GitLab / Sentry / Supabase가 모두 이 구조다.

#### 아이디어 9. 부수입 채널

- 콘텐츠: "AI 코딩 비용 90% 줄이기" YouTube / 블로그 → 광고·제휴
- 유료 강의: "AI 에이전트 라우팅 설계" ($49-199)
- GitHub Sponsors (별 400개면 충분히 가능)
- 제휴 레퍼럴: DeepSeek / Ollama 호스팅

---

### 추천 실행 루트

```
1단계 (1개월)   → React 대시보드 오픈소스 공개 (별 모으기)
2단계 (2-3개월) → Pro 티어 추가 ($9-19/월, 히스토리 + 리포트)
3단계 (3-6개월) → Team 플랜 ($15-30/개발자) + 컨설팅 병행
4단계 (6개월+)  → Universal Router로 확장
```

### 반드시 확인할 것

| 항목 | 내용 |
|---|---|
| **라이선스** | 원본 저장소 라이선스를 먼저 확인할 것 (현재 LICENSE 파일이 없어 원작자 확인 필요) |
| **기여 vs 포크** | 업스트림 기여 + 크레딧이 장기적으로 유리할 수 있음 |
| **차별화** | "포크"가 아니라 "UI를 붙였다"로 포지셔닝 |
| **리스크** | 모델 가격이 하락하면 절감 가치가 줄어듦 → "통제/거버넌스"로 축 이동 준비 |

### 가장 강력한 세일즈 한 문장

> **"AI 코딩 비용을 60% 줄이면서, 위험한 코드는 한 줄도 저가 모델에 맡기지 않습니다."**

비용과 안전을 동시에 파는 것이 핵심이다. 대부분의 경쟁자는 하나만 판다.

---

## 5. 요약 치트시트

| 질문 | 답 |
|---|---|
| 뭐하는 물건? | Codex 비용 절감 라우터 (MCP 서버) |
| 플러그인/스킬/MCP? | **MCP 서버** (+ AGENTS.md 지침 + 선택적 훅) |
| API 토큰 필요? | 보통 필요. **Ollama/LM Studio면 0원** |
| 왜 유명? | 타이밍 + 슬로건 + 벤치마크 공개 + 정직함 + 의존성 0 |
| 로컬 에이전트에 도움? | **매우 도움됨** (라우팅·샌드박스·검증·폴백 패턴) |
| React/PHP 가능? | 가능. **React 대시보드 / PHP Laravel판이 최대 기회** |
| 수익화? | 대시보드 SaaS > 팀 비용통제 > 정책 마켓 > 컨설팅 |

### 주요 링크

- 원본 저장소: https://github.com/fendouai/CodexSaver
- 이 저장소(포크): https://github.com/bmshin94/CodexSaver
- 트렌드 통계: https://trendshift.io/repositories/29293
- DeepWiki 문서: https://deepwiki.com/fendouai/CodexSaver
- Pi Agent: `@earendil-works/pi-coding-agent` (npm)
- 레포 내부 참고 문서:
  - [SPEC.md](../SPEC.md)
  - [docs/SPEC_v2.md](./SPEC_v2.md)
  - [docs/SPEC_v3.md](./SPEC_v3.md)
  - [docs/benchmarks/](./benchmarks/)

---

*이 문서는 저장소를 직접 읽고 분석한 내용을 바탕으로 작성되었습니다.*
