---
title: 3. Agent · MCP · Harness
parent: Bio → AI 트랙
nav_order: 3
---

# Agent · MCP · Harness 아키텍처
{: .no_toc }

한 번에 한 명령이 아니라, **분석 전체를 위임** 하는 자율 에이전트 시대로.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 용어 정리

세 가지가 자주 혼동됩니다.

| 용어 | 의미 | 예시 |
|---|---|---|
| **LLM** | 언어 모델 자체 | Claude, GPT, Gemini, Llama |
| **Agent** | LLM + 도구 사용 + 멀티 스텝 루프 | Claude Code, Cursor Composer, Codex |
| **Harness** | Agent 위에 얹는 운영·확장 레이어 | oh-my-claudecode, Goose, OpenHands |
| **MCP** | Agent에게 도구를 노출하는 표준 프로토콜 | Model Context Protocol |

---

## Agent 의 작동 원리

기본 루프:

```
1. 사용자 의도 받기
2. 계획 (plan) 만들기
3. 도구 호출 (tool call) — 셸 명령, 파일 편집, 웹 검색 등
4. 결과 관찰 (observe)
5. 목표 달성? → 종료 / 아니면 → 2번으로
```

이걸 **ReAct 패턴 (Reasoning + Acting)** 이라고 부릅니다. 핵심은 **도구 결과를
다시 컨텍스트에 넣어** 다음 행동을 결정한다는 것.

생물학자에게 의미: 사용자가 모든 명령을 외울 필요가 없어집니다.
"GEO ID 받아 DEG 분석" 한 줄이 50줄짜리 Bash 파이프라인으로 변환됩니다.

---

## MCP — Model Context Protocol

Anthropic이 2024년 11월 공개한 **에이전트와 외부 도구 간 표준 프로토콜**.
"USB-C for AI agents" 라고 비유됩니다.

### 왜 중요한가

MCP 이전에는 각 에이전트마다 도구 통합 방식이 다 달랐습니다.
"이 LangChain 툴을 Cursor에 붙이려면..." 같은 문제가 만연.

MCP 이후:
- **한 번 만든 MCP 서버는 Claude Code · Cursor · Cline · Goose 등에서 동일하게 사용**
- 서버 = "이 도구는 이런 함수가 있다"를 노출
- 클라이언트 (에이전트) = 표준 방식으로 호출

### 생물학자에게 유용한 MCP 서버

| MCP 서버 | 제공 도구 | 출처 |
|---|---|---|
| **filesystem** | 로컬 파일 안전한 접근 | Anthropic 공식 |
| **github** | 이슈/PR/코드 검색 | Anthropic 공식 |
| **postgres** / **sqlite** | DB 직접 쿼리 | 공식 |
| **fetch** | 웹 페이지 가져오기 | 공식 |
| **NCBI MCP** (커뮤니티) | Entrez · BLAST · PubMed | 검색해 보세요 — 빠르게 생기는 중 |
| **ChEMBL / UniProt MCP** | 약물·단백질 DB | 커뮤니티 개발 |
| **Bioconductor MCP** | R 패키지 호출 | 개발중 |

{: .tip }
> **자기만의 MCP 서버 만들기** — 연구실의 사내 DB나 도구를 MCP로 감싸면,
> Claude Code 안에서 자연어로 호출 가능. TypeScript / Python 모두 SDK 제공.
> [modelcontextprotocol.io](https://modelcontextprotocol.io)

### MCP 설치 예 (Claude Code)

```bash
# Claude Code 안에서
/mcp

# 또는 ~/.claude/mcp_servers.json 직접 편집
```

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/data/projects"]
    },
    "ncbi": {
      "command": "uvx",
      "args": ["ncbi-mcp-server"]
    }
  }
}
```

---

## Harness — Agent 위의 운영 레이어

Claude Code 자체도 강력하지만, **반복적인 작업·다중 에이전트·자동 검증** 같은
시나리오에는 부족합니다. 그 위에 얹는 것이 harness.

### oh-my-claudecode (OMC)

오픈소스 Claude Code harness. 별도 가이드 페이지를 만들었습니다 →
[**omc-guide**](../omc-guide/index.html) (별도 HTML 가이드).

핵심 기능:
- **autopilot** — 아이디어 한 줄 → 완성 코드까지 자율
- **team N:role** — N명 에이전트가 plan → exec → verify → fix 파이프라인
- **ralph** — 검증 통과할 때까지 끈질기게 반복
- **ulw (ultrawork)** — 5+ 에이전트 병렬

생물학 분석에 특히 유용한 패턴:
- 동일 RNA-seq 데이터를 STAR / Salmon 두 방법으로 병렬 처리 후 비교
- 여러 organism의 같은 분석을 워크트리별로 분리해 동시 실행

### 다른 harness들

| 이름 | 특징 |
|---|---|
| **OpenHands (구 OpenDevin)** | 오픈소스, Docker 격리, 멀티 LLM |
| **Goose** (Block) | 로컬 우선, MCP 깊이 통합 |
| **Aider** | Git 중심, 작은 규모 |
| **SWE-agent** | 학술 벤치마크 출신, 자율 SWE 작업 |

---

## 멀티 에이전트 (Multi-Agent)

한 명의 LLM이 **모든 역할을 다 하는** 대신, 역할 분담:

```
Planner       ← 사용자 의도를 PRD로 변환
   ↓
Architect     ← 시스템 설계
   ↓
Executors     ← 코드 작성 (병렬)
   ↓
Verifier      ← 결과 검증
   ↓
Fixer         ← 실패 케이스 수정
```

### 왜 도움되나

- **에러 복구** — 한 에이전트가 막히면 verifier가 잡아내고 fixer에게 패스
- **편향 감소** — 같은 모델이 코드와 리뷰를 모두 하면 자기 실수 못 봄
- **다른 모델 혼합** — 코딩은 Claude, 리뷰는 Codex, 사실 확인은 Gemini

### 생물학에서의 예

```
"이 scRNA-seq 데이터에서 희귀 세포 타입을 찾아"

Planner   → "QC → 클러스터링 → 마커 유전자 → 문헌 조회 → 검증"
Executor1 → scanpy 파이프라인 코드
Executor2 → 마커 유전자 PubMed 검색
Verifier  → UMAP 시각화 + 클러스터 안정성 체크
Fixer     → batch effect 보이면 Harmony 추가
```

이걸 사람이 직접 조율하는 게 multi-agent harness 의 역할.

---

## 보안과 격리

에이전트가 **임의 셸 명령 실행 권한** 을 갖는다는 사실은 진지하게 받아들여야 합니다.

### 격리 계층

```
1. 신뢰 못하는 데이터/코드면 Docker / Apptainer 컨테이너 안에서 실행
2. 그 위에서 Claude Code 실행, 호스트 파일시스템 격리
3. API 키는 환경변수, 코드에 박지 않기
4. 권한 명시적 허용 — 모든 Bash 자동 허용 (--yolo) 은 격리 환경에서만
```

### 데이터 민감도

**환자 데이터 / 미공개 시퀀스** 는 외부 API로 송신될 수 있습니다.

대응:
- **Bedrock / Vertex AI / Azure** 위의 Claude/Gemini 사용 — 데이터 거버넌스 통제
- **자가 호스팅 모델** (Llama 3.3, Qwen 2.5 + aider) — 인터넷 자체를 끊은 환경
- **민감 토큰 마스킹** — `presidio` 같은 PII 마스킹 후 전송

병원·게놈 데이터 다룬다면 IRB / IT 부서와 사전 협의 필수.

---

## 실전 셋업 권장

생물학자에게 **2026년 5월 기준** 가장 견고한 조합:

```
[로컬]
  └── Claude Code (CLI)
       ├── oh-my-claudecode harness (선택)
       ├── filesystem MCP (분석 폴더 격리)
       ├── github MCP (코드 협업)
       └── (선택) NCBI / UniProt MCP

[IDE]
  └── Cursor (편집·노트북)

[격리 실행]
  └── Codex Cloud 또는 GitHub Codespaces
       — 큰 변경·실험적 코드는 여기서
```

---

## 다음

→ **[4. 실전 워크플로우](workflows)** — 위 도구들을 실제 분석 시나리오에 어떻게
끼우는지 코드 단위로.
