# Agent Skills 전수조사 분석 정리 (한국어)

> 이 문서는 `agentskills` 저장소를 전수조사하여 **무엇을 하는 프로젝트인지, 언제 쓰는지,
> 어떤 도움이 되는지**를 정리한 분석 노트입니다. Claude Code와의 대화 내용을 정리한 것입니다.
>
> - 작성일: 2026-10-13
> - 분석 대상 커밋: `6ff459a` (branch `main`)

## 관련 GitHub 주소

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/agentskills |
| **원본 저장소 (upstream)** | https://github.com/agentskills/agentskills |
| 공식 문서 사이트 | https://agentskills.io |
| 공식 규격 | https://agentskills.io/specification |
| 클라이언트 쇼케이스 | https://agentskills.io/clients |
| 공식 예제 스킬 모음 | https://github.com/anthropics/skills |
| 참조 구현 (`skills-ref`) | https://github.com/agentskills/agentskills/tree/main/skills-ref |
| 커뮤니티 Discord | https://discord.gg/MKPE9g8aUy |
| 이슈 | https://github.com/agentskills/agentskills/issues |
| 디스커션 (제안/질문) | https://github.com/agentskills/agentskills/discussions |

---

## 1. 이 저장소는 무엇인가

**AI 에이전트에게 새로운 능력과 전문성을 부여하는 표준 파일 포맷(Agent Skills)의
공식 규격서 + 문서 사이트.** 실행되는 애플리케이션이 아니라 **문서 + 규격 + 작은 검증 도구** 저장소다.

- 원래 Anthropic이 개발해 **오픈 표준**으로 공개
- 라이선스: 코드 **Apache-2.0** / 문서 **CC-BY-4.0** (상업적 이용 가능, 출처 표기 조건)
- 현재 **46개 AI 제품**이 채택 (OpenAI, Google, Microsoft, JetBrains, Databricks, Snowflake, Mistral 포함)

### 핵심 개념

**폴더 하나 + `SKILL.md` 파일 하나 = 스킬 완성.**

```
my-skill/
├── SKILL.md          # 필수: 메타데이터(YAML frontmatter) + 지시문(Markdown)
├── scripts/          # 선택: 실행 코드
├── references/       # 선택: 참고 문서
└── assets/           # 선택: 템플릿, 이미지, 데이터
```

```markdown
---
name: roll-dice
description: Roll dice using a random number generator. Use when asked to roll a die (d6, d20, etc.).
---

주사위를 굴릴 때는 아래 명령으로 1부터 면수까지의 난수를 생성하세요:

    echo $((RANDOM % <sides> + 1))

`<sides>`를 주사위 면수로 바꾸세요 (일반 주사위 6, d20은 20).
```

### 프론트매터 필드 (전체 6개)

| 필드 | 필수 | 제약 |
|---|---|---|
| `name` | O | 64자 이하, 소문자+숫자+하이픈만, 하이픈 시작/끝 금지, `--` 금지, **폴더명과 일치 필수** |
| `description` | O | 1024자 이하. "무엇을 하는지 + 언제 쓰는지" 둘 다 |
| `license` | X | 라이선스명 또는 번들된 라이선스 파일명 |
| `compatibility` | X | 500자 이하. 환경 요구사항 (대상 제품, 필요 패키지, 네트워크 등) |
| `metadata` | X | 문자열 키-값 맵 (클라이언트 자유 사용) |
| `allowed-tools` | X | 공백 구분 사전 승인 툴 목록 (**실험적 — 변경 가능**) |

### 작동 원리: 점진적 공개 (Progressive Disclosure)

이 포맷의 존재 이유.

| 단계 | 로드되는 것 | 시점 | 토큰 비용 |
|---|---|---|---|
| 1. 발견(Discovery) | `name` + `description`만 | 세션 시작 | 스킬당 **50~100 토큰** |
| 2. 활성화(Activation) | `SKILL.md` 본문 전체 | 작업이 매칭될 때 | 5,000 토큰 이하 권장 |
| 3. 실행(Execution) | `scripts/`, `references/` 파일 | 지시문이 참조할 때 | 가변 |

→ 스킬 50개를 깔아놔도 평소 비용은 5,000 토큰 정도. 실제 쓸 때만 전체를 읽는다.
"프롬프트를 전부 시스템 프롬프트에 박아넣기"와의 결정적 차이.

### 저장소 구조 (실측)

```
agentskills/
├── README.md                  # 프로젝트 소개
├── AGENTS.md                  # AI 에이전트용 기여 지침
├── CLAUDE.md                  # (!) 심볼릭 링크 - 깨져있음, 아래 '발견된 이슈' 참고
├── CONTRIBUTING.md            # 기여 규칙 (AI 사용 시 고지 의무)
├── LICENSE                    # Apache 2.0
├── package.json               # npm run dev -> Mintlify 문서 서버
├── .claude/
│   ├── settings.json          # SessionStart 훅 설정
│   └── hooks/session-start.sh # Mintlify CLI 설치 여부 확인
├── docs/                      # 핵심: Mintlify 문서 사이트 (총 2,356줄)
│   ├── docs.json              # 네비게이션/테마/리다이렉트
│   ├── home.mdx               # 개요 (74줄)
│   ├── specification.mdx      # 공식 규격 - 유일한 권위 문서 (247줄)
│   ├── clients.mdx            # 지원 제품 쇼케이스 (9줄)
│   ├── skill-creation/        # 스킬 제작자용 가이드 5편
│   │   ├── quickstart.mdx            # 주사위 스킬 실습 (80줄)
│   │   ├── best-practices.mdx        # 좋은 스킬 작성법 (275줄, 가장 알참)
│   │   ├── optimizing-descriptions.mdx # description 튜닝 + 트리거 평가 (193줄)
│   │   ├── evaluating-skills.mdx     # 품질 평가/채점 자동화 (298줄)
│   │   └── using-scripts.mdx         # scripts/ 설계법 (298줄)
│   ├── client-implementation/
│   │   └── adding-skills-support.mdx # 에이전트에 Skills 지원 붙이기 (335줄)
│   ├── snippets/
│   │   ├── clients.jsx               # 지원 클라이언트 46개 데이터 (416줄)
│   │   ├── LogoCarousel.jsx          # 로고 캐러셀 컴포넌트
│   │   └── ClientShowcase.jsx        # 쇼케이스 그리드 컴포넌트
│   └── images/logos/                 # 46개 제품 로고 (light/dark 쌍)
└── skills-ref/                # 참조 구현 (Python, 데모 전용 — 프로덕션 금지 명시)
    ├── src/skills_ref/
    │   ├── parser.py          # SKILL.md YAML frontmatter 파싱 (strictyaml)
    │   ├── validator.py       # 규격 위반 검사 (이름/길이/디렉터리 일치)
    │   ├── models.py          # SkillProperties 데이터클래스
    │   ├── prompt.py          # <available_skills> XML 생성
    │   ├── cli.py             # validate / read-properties / to-prompt
    │   └── errors.py          # SkillError, ParseError, ValidationError
    └── tests/                 # pytest 549줄, 40개 테스트 (i18n: 중국어/러시아어 포함)
```

### 지원 제품 46개

Claude Code, Claude, **ChatGPT & Codex**, **Gemini CLI**, **GitHub Copilot**, **VS Code**,
**Cursor**, JetBrains Junie, Kiro, TRAE, Roo Code, OpenCode, OpenHands, Goose, Amp, Factory,
Letta, Databricks Genie Code, Snowflake Cortex Code, Mistral AI Vibe, Spring AI,
**Laravel Boost (PHP)**, Tabnine, Qodo, Pulumi Neo, Firebender, Mux, pi, Agentman, Ona,
VT Code, Workshop, Emdash, nanobot, fast-agent, bub, Vita, Superconductor, Deep Code,
Hermes Agent, OpenClaw, ZeroClaw, Autohand Code CLI, Piebald, Command Code,
Google AI Edge Gallery

→ 벤더 중립 표준으로 자리를 잡았다는 의미. **"Claude 전용"이 아니다.**

---

## 2. 쉬운 비유로 다시 보기

### "회사 신입사원 + 업무 매뉴얼 캐비닛"

AI 에이전트는 **머리는 좋지만 우리 회사 사정은 전혀 모르는 신입사원**이다.
Python도 알고 엑셀도 쓰지만, 이런 건 모른다:

- "우리 회사는 세금계산서를 이 양식으로 발행한다"
- "`users` 테이블은 soft delete라서 `WHERE deleted_at IS NULL`을 꼭 붙여야 한다"
- "배포 전에 꼭 `scripts/migrate.py --verify --backup`을 먼저 돌려야 한다"

**방법 A (나쁜 방법)**: 매번 채팅창에 복붙 → 귀찮고, 까먹고, 사람마다 다르게 설명
**방법 B (Agent Skills)**: 매뉴얼을 폴더에 정리해 캐비닛에 꽂아둔다

```
.agents/skills/
├── 세금계산서-발행/SKILL.md
├── db-쿼리-규칙/SKILL.md
└── 배포-절차/SKILL.md
```

신입은 출근하면 캐비닛의 **라벨만** 읽는다(= `name` + `description`).
"세금계산서 발행해줘" 요청이 오면 → 라벨 보고 → 해당 매뉴얼만 꺼내 읽고 → 실행.

**왜 라벨만 먼저 읽나?** AI가 한 번에 기억하는 양(컨텍스트 윈도우)은 노트 한 권처럼 정해져 있다.
매뉴얼 50권을 전부 베껴 쓰면 정작 사용자 질문을 적을 자리가 없다.
라벨 50개만 적으면 노트의 5%만 쓰고, 필요할 때 1권만 추가로 펼친다.

### 다른 방식과의 비교

| | 프롬프트 복붙 | CLAUDE.md / .cursorrules | **Agent Skills** |
|---|---|---|---|
| 자동 선택 | X 사람이 고름 | X 항상 전체 로드 | O AI가 상황 보고 고름 |
| 토큰 비용 | 매번 전체 | **항상** 전체 | 평소엔 라벨만 |
| 코드 실행 | X | X | O `scripts/` 번들 |
| 다른 도구 호환 | X | X 도구마다 다른 파일 | O **46개 제품 공통** |
| 버전 관리 | X | 부분적 | O git 폴더 |

마지막 줄이 핵심. 스킬 하나 만들면 Claude Code·Cursor·Copilot·Codex에서 그대로 작동한다. USB 규격처럼.

### 구체적으로 어떤 도움이 되나

1. **반복 설명을 영구히 없앤다** — 프로젝트 컨벤션을 `SKILL.md`에 한 번만 쓰면 끝
2. **AI의 실수를 영구 수정한다** — Gotchas 섹션에 한 줄 추가하면 다음부터 안 틀린다
3. **노하우를 자산/상품으로 만든다** — 폴더로 패키징 → 팀 배포, 공개, 판매 가능
4. **진입 장벽이 사실상 0** — SDK/API키/서버/빌드 전부 불필요. 마크다운 1개. 10분이면 작동

---

## 3. 주요 질문 답변

### Q. 설치 및 사용법?

**케이스 A — 스킬을 만들어 쓰고 싶다 (99%의 경우: 설치할 게 없다)**

```bash
cd ~/my-project
mkdir -p .agents/skills/roll-dice
# .agents/skills/roll-dice/SKILL.md 작성 (위 예시 참고)
```
Claude Code / Cursor / VS Code Copilot(Agent 모드)에서 "d20 굴려줘" → 자동 활성화.

**스킬 탐색 경로**

| 범위 | 경로 | 용도 |
|---|---|---|
| 프로젝트 | `<project>/.agents/skills/` | **크로스 클라이언트 표준 (권장)** |
| 프로젝트 | `<project>/.claude/skills/` | Claude Code 네이티브 |
| 사용자 | `~/.agents/skills/` | 모든 프로젝트 공통 |
| 사용자 | `~/.claude/skills/` | Claude Code 전역 |

충돌 시: **프로젝트 레벨이 사용자 레벨을 덮어쓴다.**

**케이스 B — 문서 사이트 로컬 실행**

```bash
git clone https://github.com/bmshin94/agentskills
cd agentskills
npm i -g mint
npm run dev          # 또는: cd docs && mint dev  → http://localhost:3000
```

**케이스 C — 검증 CLI**

```bash
cd agentskills/skills-ref
uv sync && source .venv/bin/activate
# 또는: python -m venv .venv && source .venv/bin/activate && pip install -e .

skills-ref validate ./my-skill          # 규격 위반 검사
skills-ref read-properties ./my-skill   # 메타데이터 JSON 출력
skills-ref to-prompt ./a ./b            # <available_skills> XML 생성
```
Windows는 `.venv\Scripts\Activate.ps1`.

**작성 핵심 규칙**

1. `name`은 폴더명과 반드시 같아야 한다 (다르면 검증 실패)
2. `description`에 "무엇을" + "언제"를 둘 다 쓴다 — 발동 여부를 100% 결정
   - 나쁨: `description: PDF 작업을 도와줍니다`
   - 좋음: `description: Extracts text and tables from PDFs, fills forms, merges files. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction.`
3. `SKILL.md`는 500줄 / 5,000토큰 이하. 넘으면 `references/`로 분리하고 **"언제 읽어야 하는지" 조건을 명시**
4. AI가 이미 아는 건 쓰지 않는다 ("PDF란 무엇인가" 금지, "우리는 pdfplumber를 쓴다"만)
5. 선택지 나열 금지, 기본값을 정해준다

### Q. 플러그인? 스킬? MCP?

**셋 다 아니다. 이 저장소는 "스킬의 규격 문서"다.** 그리고 셋은 경쟁 관계가 아니라 층(layer)이 다르다.

| | **Agent Skills** | **MCP** | **플러그인** |
|---|---|---|---|
| 정체 | 지시문 폴더 | 툴 연결 프로토콜 | 배포 번들 |
| 핵심 질문 | **"어떻게 해야 하나?"** | **"무엇을 할 수 있나?"** | "어떻게 묶어 배포하나?" |
| 구성물 | `SKILL.md` (마크다운) | 실행 중인 서버 (JSON-RPC) | 스킬+명령+훅+MCP 묶음 |
| 서버 필요 | X | O 프로세스 상주 | 부분적 |
| API 키 | X | 보통 필요 | 부분적 |
| 토큰 비용 | 라벨 50~100토큰/개 | 툴 스키마 전체 상주 | - |
| 난이도 | 마크다운 | 서버 코딩 | 중간 |
| 표준 소속 | agentskills.io | modelcontextprotocol.io | 제품별 |

**가장 쉬운 구분**
- **MCP = 손 (capability)** — Slack에 메시지 보낼 능력, DB 조회할 능력
- **Skill = 뇌/매뉴얼 (procedure)** — "장애 시 (1) DB 조회 → (2) 원인 분석 → (3) Slack #incident에 이 양식으로 보고"
- **플러그인 = 가방** — 위 둘을 묶어 팀에 배포하는 포장

함께 쓰는 게 정석. `allowed-tools` 필드가 "이 스킬이 쓸 MCP 툴"을 사전 승인하는 용도.
Claude Code의 "플러그인"은 스킬·슬래시명령·훅·MCP를 묶은 상위 개념이므로,
**스킬은 플러그인의 구성 요소 중 하나**다.

### Q. API 토큰을 사용해야 돼?

**이 포맷 자체는 토큰이 전혀 필요 없다.** 스킬은 텍스트 파일이고 아무것도 네트워크를 호출하지 않는다.

| 상황 | 토큰 필요? |
|---|---|
| `SKILL.md` 작성 | X |
| 스킬이 AI에 로드/발동 | X |
| `skills-ref validate` 실행 | X |
| 문서 사이트 로컬 실행 | X |
| Claude Code / Cursor로 그 스킬 쓰기 | O — 단 그건 AI 도구 구독료/API 키이고 스킬과 무관 |
| 스킬 안의 스크립트가 외부 API 호출 | O — 그 API의 키, 당신이 설계한 것 |

**중요: 토큰을 `SKILL.md`에 절대 넣지 말 것.** 스킬 폴더는 보통 git에 커밋되고 팀에 공유된다.
비밀값은 환경변수로 빼고 `SKILL.md`에는 "어디서 읽어라"만 쓴다.

```markdown
## 인증
`STRIPE_API_KEY` 환경변수를 사용한다. 값을 직접 출력하거나 로그에 남기지 말 것.
설정이 안 되어 있으면 사용자에게 설정을 요청하고 중단한다.
```

**보안 주의 (문서의 Trust considerations):** 프로젝트 레벨 스킬은 클론한 저장소에서 따라온다.
남의 오픈소스를 클론하면 그 안의 `.agents/skills/`가 AI 컨텍스트에 조용히 지시문을 주입할 수 있다.
모르는 저장소를 클론할 때는 `.agents/skills/`와 `.claude/skills/`를 먼저 확인할 것.

### Q. AI 에이전트 구축에 도움이 될까?

**매우 큰 도움. 두 방향으로.**

**방향 1 — 에이전트를 "쓰는" 쪽 (즉시 실용적)**

`best-practices.mdx`의 검증된 5가지 패턴은 Agent Skills를 안 쓰더라도
프롬프트 엔지니어링 그 자체로 가치가 있다:

1. **Gotchas 섹션** — 상식을 배반하는 환경 사실 목록 (가장 가치 높은 콘텐츠)
2. **Templates** — 출력 포맷은 산문 설명보다 템플릿이 훨씬 안정적
3. **Checklists** — 다단계 작업의 단계 누락 방지
4. **Validation loops** — 작업 → 검증 스크립트 → 실패 시 수정 → 재검증 → 통과 시에만 진행
5. **Plan-validate-execute** — 파괴적 작업은 (1) 계획을 JSON으로 생성 → (2) 스크립트로 정답과 대조 → (3) 그 다음에만 실행

5번이 특히 강력하다. 에러 메시지를
`"Field 'signature_date' not found — available fields: customer_name, order_total, signature_date_signed"`
처럼 설계하면 AI가 스스로 자기 실수를 고친다. 믿을 수 있는 에이전트의 핵심 기법.

**방향 2 — 에이전트를 "만드는" 쪽 (설계 교과서)**

`client-implementation/adding-skills-support.mdx` 335줄은 사실상 에이전트 하네스 설계 가이드:

| Step | 해결하는 문제 | 핵심 지침 |
|---|---|---|
| 1. Discover | 능력을 어디서 찾나 | 프로젝트/유저 2개 스코프, `.git`·`node_modules` 제외, **최대 깊이 4~6 / 최대 2000 디렉터리** 상한, 충돌은 프로젝트 우선 |
| 2. Parse | 깨진 입력 처리 | 콜론 포함 비인용 YAML 폴백, **관대한 검증** (name 불일치는 경고만, description 없으면 스킵) |
| 3. Disclose | 모델에 뭘 알리나 | `<available_skills>` 카탈로그, 비활성 스킬은 **아예 숨김**, 스킬 0개면 블록 전체 생략 |
| 4. Activate | 어떻게 로드하나 | 파일읽기 vs 전용툴, 툴 파라미터를 **enum으로 제한해 환각 방지**, `<skill_content>` 래핑, **스킬 디렉터리 권한 사전 허용** |
| 5. Manage | 어떻게 유지하나 | **컨텍스트 압축에서 스킬 제외**(안 하면 조용히 성능 저하), 중복 활성화 제거, 서브에이전트 위임 |

**Step 5의 "압축에서 스킬 제외"** 는 직접 에이전트 만들다 꼭 당하는 함정이다.
대화가 길어지면 오래된 메시지를 요약/삭제하는데, 스킬 지시문이 같이 날아가면
**에러 하나 없이 에이전트가 점점 멍청해진다.**

`skills-ref/`의 Python 코드는 이 5단계 중 1~3단계의 레퍼런스 구현(약 300줄)이다.
어떤 언어로 포팅하든 교과서가 된다.

**추가: 평가(eval) 체계**

`evaluating-skills.mdx` + `optimizing-descriptions.mdx`는 "에이전트 품질 측정"에 답한다.
테스트 케이스 설계 → 워크스페이스 구조 → 병렬 실행 → assertion → LLM 채점 → 집계 → 패턴 분석 → 반복.
`should_trigger: true/false` 라벨 **20개 쿼리**(긍정 8~10, 부정 8~10) 구성법,
**near-miss 네거티브 케이스**가 가장 가치 있다는 점,
**train/validation 분할로 오버피팅 방지**까지 다룬다. 에이전트 개발의 CI/CD 격.

### Q. React나 PHP로 만들 수 있어?

**세 가지 해석 모두 Yes.**

**해석 A — React/PHP 프로젝트용 스킬 만들기: 가장 흔한 용도**

스킬은 마크다운이라 언어 무관. **Laravel Boost**가 이미 공식 지원 46개에 포함(PHP/Laravel).

```markdown
---
name: react-component
description: Create React components following this project's conventions. Use when adding a new component, page, or hook to the frontend.
---

## 컴포넌트 생성 절차
1. `src/components/<PascalCase>/` 디렉터리 생성
2. `index.tsx` — named export만 사용. default export 금지.
3. `index.test.tsx` — React Testing Library. 최소 렌더 테스트 1개 필수.
4. 스타일은 Tailwind 유틸리티. CSS 모듈 신규 생성 금지.
5. `npm run typecheck && npm run lint` 통과 확인

## Gotchas
- `useAuth()`는 `<AuthProvider>` 안에서만 동작. Storybook에서는 `MockAuthProvider`를 써라.
- `src/api/client.ts`의 axios 인스턴스를 써라. `fetch` 직접 호출 금지 (인터셉터가 토큰 갱신 처리).
- 서버 상태는 TanStack Query. `useState`로 API 응답을 들고 있지 말 것.
```

**해석 B — 스킬을 다루는 도구를 React/PHP로 만들기: 가능, 기회 있음**

`skills-ref`는 Python 300줄. 로직은 (1) YAML 프론트매터 파싱 (2) 규칙 검증 (3) XML 생성뿐.

React로: 스킬 에디터(`gray-matter` + `monaco-editor`), 라이브러리 브라우저,
트리거 테스트 대시보드, 스킬 린터 VS Code 웹 확장
PHP로: 팀 내부 스킬 레지스트리(Laravel + `symfony/yaml`), Composer 패키지
`php artisan skill:make`, WordPress 플러그인, 사내 승인 워크플로우

검증 로직 핵심 (JS/TS 포팅):

```js
const errors = [];
const n = name.normalize('NFKC').trim();
if (!n) errors.push("name은 비어있을 수 없음");
if (n.length > 64) errors.push("name 64자 초과");
if (n !== n.toLowerCase()) errors.push("name은 소문자여야 함");
if (n.startsWith('-') || n.endsWith('-')) errors.push("하이픈으로 시작/끝 금지");
if (n.includes('--')) errors.push("연속 하이픈 금지");
if (!/^[\p{Ll}\p{N}-]+$/u.test(n)) errors.push("허용되지 않는 문자");
if (dirName !== n) errors.push("디렉터리명과 불일치");
if (description.length > 1024) errors.push("description 1024자 초과");
if (compatibility && compatibility.length > 500) errors.push("compatibility 500자 초과");
```

**주의 2가지**
1. **이 저장소의 `skills-ref/`에 PR을 보내지 말 것.** `CONTRIBUTING.md`가 "현재 코드 기여를 받지 않음"을 명시하고 `AGENTS.md`도 "일반 기여 대상이 아니다"라고 못 박음. **별도 저장소로 만들 것.**
2. Mintlify 문서 사이트는 React 기반 (`docs/snippets/*.jsx`가 실제 React 컴포넌트)

**해석 C — Skills를 지원하는 에이전트 만들기: 가능**

`adding-skills-support.mdx` 5단계를 구현하면 된다.
이미 Spring AI(Java), Laravel Boost(PHP), ZeroClaw(Rust)가 해냈다. 언어 제약 없음.

### Q. 유튜브 강의 영상 제작 가능할까?

**매우 적합한 소재.**

| 조건 | 평가 |
|---|---|
| 진입장벽 | 최상 — 마크다운 1개. **"설치 없음"이 썸네일 후킹** |
| 시연 가능성 | 최상 — 주사위 스킬 10분 내 작동 확인 |
| 시의성 | 최상 — OpenAI·Google·MS 전부 채택 = "표준 전쟁 끝났다" 서사 |
| 한국어 콘텐츠 | 최상 — **거의 없음.** MCP는 포화, Skills는 블루오션 |
| 라이선스 | 최상 — 문서 CC-BY-4.0, 인용·번역·재구성 합법 (출처 표기만) |
| 콘텐츠 깊이 | 상 — 2,356줄 문서 = 10편 이상 시리즈 분량 |

**추천 시리즈 10편**

| # | 제목 | 길이 |
|---|---|---|
| 0 | MCP는 잊어라? 업계 표준 'Agent Skills' 5분 정리 (쇼츠) | 5분 |
| 1 | 마크다운 1개로 AI에게 새 능력 주기 — 10분 실습 | 10분 |
| 2 | 스킬 vs MCP vs 플러그인 — 3분이면 완벽 이해 | 8분 |
| 3 | 내 프로젝트 전용 스킬 만들기 (React 실전) | 20분 |
| 4 | AI가 같은 실수를 반복한다면 — Gotchas 패턴 | 12분 |
| 5 | 스킬이 발동 안 될 때 — description 튜닝 과학 | 18분 |
| 6 | scripts/로 코드 번들링 — AI에게 도구 쥐어주기 | 15분 |
| 7 | Plan-Validate-Execute — 안전한 파괴적 작업 | 15분 |
| 8 | 팀에 스킬 배포하기 + 보안 주의사항 | 15분 |
| 9 | 내 에이전트에 Skills 지원 직접 붙이기 (개발자편) | 30분 |

**제작 시 지킬 것**
1. 출처 표기 필수 (CC-BY-4.0) — 설명란에 `https://agentskills.io` + Anthropic 원저작 명시
2. 로고 사용 주의 — `docs/images/logos/`의 46개는 각 회사 상표. 저장소 라이선스로 커버되지 않음
3. **"Claude 전용"이라고 말하지 말 것** — 가장 흔한 오해이고 핵심 세일즈 포인트를 버리는 것
4. 실제로 돌아가는 화면 필수 — 문서 낭독 영상은 안 봄
5. `allowed-tools`는 **실험적** 필드. 변경될 수 있다고 언급

---

## 4. 수익화 아이디어

### 전제: "스킬 파일 자체"는 팔기 어렵다

| 문제 | 설명 |
|---|---|
| 복제 비용 0 | 마크다운 파일. 한 명이 사면 Ctrl+C로 무한 배포 |
| DRM 불가 | AI가 읽어야 하므로 평문 필수. 난독화 불가 |
| 공짜 경쟁자 | `github.com/anthropics/skills`에 공식 예제 무료 공개 중 |
| 모방 용이 | 잘 팔리는 스킬은 설명만 보고 하루면 복제 |

**결론: 스킬은 "미끼"로 쓰고, 돈은 (1) 지식·교육 (2) 지속적 유지보수 (3) 인프라·운영 (4) 인증·신뢰 에서 받는다.**

### 트랙 1. 교육·콘텐츠 — 추천도 최상 (지금 당장 가능)

근거: 한국어 Agent Skills 콘텐츠가 사실상 없다. MCP는 이미 포화.

```
유튜브 무료 시리즈 (유입)
   → 유료 강의 (인프런/클래스101/유데미) 10~15만원
   → 유료 뉴스레터/멤버십 월 1~2만원
   → 기업 사내 교육 회당 100~500만원   ← 실제 돈은 여기
```

- 0~2개월: 유튜브 10편 + 블로그. **번역이 아니라 한국 개발 현실에 맞춘 재구성** (SI 환경, 금융권 규제 등)
- 2~4개월: 인프런 강의 "AI 에이전트를 내 회사에 맞게 길들이기" 4시간, 11만원
- 4개월~: 기업 교육 영업 — 개발팀 20명 반나절 워크숍 = 300만원대

리스크: 포맷 변경 시 노후화 → `specification.mdx`를 git watch, 영상에 기준일 명시.

### 트랙 2. 수직 도메인 스킬팩 — 추천도 상

**핵심: 범용 스킬을 팔지 말고 "AI가 절대 모르는 좁은 도메인"을 판다.**
AI가 이미 아는 것(React 기초, Git)은 가치 0. **가치는 "AI가 틀리는 영역"에 있다.**

| 도메인 | 왜 AI가 틀리나 | 상품 형태 | 가격대 |
|---|---|---|---|
| 전자세금계산서/부가세 신고 | 국세청 양식·규정 매년 변경, 영어 자료 없음 | 세무법인용 + 월 업데이트 | 월 10~30만원 |
| 금융권 보안 규제 (전금법, 망분리) | 한국 특수 규제, 컴플라이언스 체크리스트 | 금융 SI 코드리뷰 스킬 | 연 500만원+ |
| 공공 조달/나라장터 제안서 | 양식 까다롭고 매년 변경 | 제안서 작성 스킬 + 템플릿 | 건당/연간 |
| 의료 EMR/심평원 청구 | 수가 코드, 삭감 사유 로직 | 의료기관용 | 월 과금 |
| 노무/근로계약 (주52시간) | 판례·고용부 지침 수시 변경 | 노무법인용 | 월 과금 |
| Laravel/PHP 레거시 현대화 | 한국 SI에 PHP 레거시 대량 | 마이그레이션 스킬팩 | 프로젝트 단위 |

복제 저항의 근거: **도메인 전문성이 진입장벽**이고, **규정이 바뀐다.**
파일을 복사해가도 다음 개정 때 쓸모가 없어진다. 즉 "파일"을 파는 게 아니라
**"최신 상태로 유지해주는 것"** 을 판다.

```
스킬팩 기본 = 무료 공개 (마케팅)
  + 업데이트 구독 = 월 과금 (법령 개정 반영)
  + 기업 커스터마이징 = 건당
```

### 트랙 3. 스킬 개발·운영 툴링 — 추천도 상 (React/PHP 역량 직결)

근거: 공식 `skills-ref`가 "데모 전용, 프로덕션 금지"를 명시하고 코드 기여도 받지 않는다.
→ **프로덕션급 툴링 공백이 열려 있다.**

1. **스킬 에디터/스튜디오 (React)** — 프론트매터 폼 + 실시간 검증 + 토큰 카운터(5,000 경고선) + description 품질 점수. Freemium, 팀 월 $15~30
2. **트리거 테스트 SaaS (가장 차별화)** — 쿼리 20개 → 여러 모델 N회 실행 → 발동률 히트맵, train/validation 오버피팅 자동 경고. 월 $29~99. **수동으로 하는 사람이 거의 없다**
3. **스킬 CI/CD GitHub Action** — PR마다 validate + 토큰 예산 + 트리거 회귀 테스트. OSS 무료 / 프라이빗 유료
4. **팀 스킬 레지스트리 (PHP/Laravel)** — 제출→리뷰→승인→배포 + **감사 로그**(금융·공공 필수). 온프레미스 연 라이선스
5. **스킬 보안 스캐너 (수요 임박)** — 프롬프트 인젝션 패턴, 하드코딩 시크릿, 위험한 `allowed-tools`(`Bash(*)`) 탐지. **"스킬 공급망 보안"은 아직 아무도 안 하고 있다**

### 트랙 4. 컨설팅·구축 (SI) — 추천도 최상 (수익 즉시성 1위)

근거: 한국 기업은 "AI 도입" 압박은 받지만 방법을 모른다. 그리고 소프트웨어 구독보다 용역비 지출에 익숙하다.

| 상품 | 내용 | 기간 | 가격대 |
|---|---|---|---|
| AI 워크플로우 진단 | 팀 작업 관찰 → 스킬화 대상 10개 식별 → 리포트 | 1~2주 | 300~800만원 |
| 스킬팩 구축 | 스킬 10~20개 제작 + 평가 세트 | 4~8주 | 1,500~5,000만원 |
| 사내 에이전트 구축 | `adding-skills-support.mdx` 5단계 구현 | 2~4개월 | 5,000만원~ |
| 운영 리테이너 | 월 단위 유지보수·개선·교육 | 상시 | 월 200~500만원 |

**결정적 무기: 측정 가능한 ROI.** 대부분의 AI 컨설팅은 성과를 증명하지 못한다.
`evaluating-skills.mdx`의 평가 프레임워크로 **"적용 전 성공률 40% → 적용 후 85%"** 를
숫자로 보여줄 수 있다. 이게 재계약과 단가의 근거.

리드 확보: 트랙 1(유튜브/강의)이 그대로 영업 파이프라인.

### 트랙 5. 템플릿·마켓플레이스 — 추천도 하

Gumroad 등에 스킬팩 판매($19~99) 또는 마켓플레이스 운영.
**솔직한 평가: 복제 비용 0 문제를 정면으로 맞고 단가가 낮아 생계가 안 된다.**
트랙 1·4의 보조 수익(리드 확보용 유료 샘플)로만 쓸 것.
마켓플레이스는 닭-달걀 문제가 심하고 공식 레지스트리가 나오면 즉사.

### 트랙 6. 인증·커뮤니티 — 추천도 중 (트랙 1 성숙 후)

"Agent Skills 인증 개발자" 과정+시험(20~30만원), 유료 커뮤니티(월 2~5만원),
기업 "AI 활용 역량 진단" 상품. 권위 확보가 선행 조건. 6~12개월 후 과제.

### 종합 실행 순서

```
[0~1개월]  유튜브 0·1·2편 + 블로그  → 비용 0, 시장 반응 측정
[1~3개월]  트랙 2 도메인 1개 선정 (본인이 가장 잘 아는 것)
           스킬팩 무료 공개 + 유튜브 홍보
[2~4개월]  인프런 강의 출시 / 트랙 3-(2) 트리거 테스트 도구 MVP (React)
[3~6개월]  트랙 4 컨설팅 수주  ← 실제 매출
[6개월~]   리테이너 전환 + 트랙 3 SaaS 유료화 + 트랙 6
```

**가장 현실적인 단일 추천: "도메인 1개 + 콘텐츠 + 컨설팅" 삼각 구조**
- 도메인 스킬팩 = 신뢰의 증거 (무료 공개)
- 유튜브/강의 = 유입 채널 (저비용)
- 컨설팅/교육 = 실제 매출 (고단가)

SaaS(트랙 3)는 매력적이지만 자본·시간이 필요하고 공식 툴링이 나오면 경쟁이 생긴다.
먼저 1·4로 현금흐름을 만들고, 그 과정에서 반복되는 불편을 3으로 제품화하는 순서를 권함.

### 피해야 할 함정

1. 범용 스킬팩 판매 ("최고의 코딩 스킬 100개") — 공식 무료 예제와 경쟁, 즉시 복제
2. 이 저장소에 PR 보내 영향력 확보 시도 — `skills-ref` 기여 안 받음
3. "Claude 전용"으로 포지셔닝 — 46개 제품 호환이 최대 무기인데 스스로 버리는 것
4. **AI 기여 미고지** — `CONTRIBUTING.md`가 AI 사용 시 PR/이슈에 반드시 고지하라고 요구하며 미고지 PR은 닫는다고 명시. 평판이 자산인 비즈니스에서 치명적
5. 평가 없이 스킬 납품 — ROI 증명 못 하면 재계약 없음

---

## 발견된 이슈: `CLAUDE.md`가 깨진 심볼릭 링크

전수조사 중 발견. 커밋 `7b1128d` ("docs: appended CLAUDE.md persona guide", PR #1 `feat/claude-guide`)에서
`CLAUDE.md`가 **일반 파일이 아니라 심볼릭 링크로 커밋됐다.**

```
$ ls -la CLAUDE.md
lrwxrwxrwx CLAUDE.md -> # Agent Instructions↵↵## Project perspective↵...  (3,407자 전체)

$ cat CLAUDE.md
cat: CLAUDE.md: File name too long
```

파일 **내용**이 되어야 할 3,407자가 심볼릭 링크의 **대상 경로**로 들어갔다. 결과:

- `cat CLAUDE.md` 실패 → **Claude Code가 이 파일을 읽을 수 없다**
- 추가한 페르소나 지침이 **전혀 적용되지 않는다**

수정 방법:

```bash
git rm --cached CLAUDE.md
rm CLAUDE.md
# 의도한 내용을 일반 파일로 다시 작성
cat > CLAUDE.md <<'EOF'
(원하는 내용)
EOF
git add CLAUDE.md && git commit -m "fix: restore CLAUDE.md as a regular file"
```

---

## 참고: 핵심 문서 빠른 링크

| 알고 싶은 것 | 읽을 문서 |
|---|---|
| 포맷 규격 전체 | `docs/specification.mdx` |
| 첫 스킬 10분 실습 | `docs/skill-creation/quickstart.mdx` |
| 좋은 스킬 작성법 (가장 알참) | `docs/skill-creation/best-practices.mdx` |
| 스킬이 발동 안 될 때 | `docs/skill-creation/optimizing-descriptions.mdx` |
| 스킬 품질 측정 | `docs/skill-creation/evaluating-skills.mdx` |
| scripts/ 설계 | `docs/skill-creation/using-scripts.mdx` |
| 내 에이전트에 Skills 붙이기 | `docs/client-implementation/adding-skills-support.mdx` |
| 기여 규칙 (AI 고지 의무) | `CONTRIBUTING.md` |
| 참조 구현 코드 | `skills-ref/src/skills_ref/` |
