---
title: 1. Coding Agents 입문
parent: Bio → AI 트랙
nav_order: 1
---

# Coding Agents 입문
{: .no_toc }

`bwa mem`, `samtools view`, `STAR --runThreadN ...` — 이 명령들을 외우던 시대는
끝났습니다. 자연어로 시키고, 막히면 에이전트가 직접 디버깅합니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Coding Agent 가 뭔가요

"코딩 에이전트(coding agent)"는 ChatGPT의 채팅 인터페이스를 넘어, **터미널·파일
시스템·툴 호출 권한** 을 갖고 사용자의 컴퓨터에서 직접 작업하는 AI를 말합니다.

| 구분 | 채팅 LLM (ChatGPT 등) | Coding Agent (Claude Code 등) |
|---|---|---|
| 코드 실행 | ❌ (또는 sandbox) | ✅ 사용자 셸에서 직접 |
| 파일 접근 | ❌ | ✅ read / write / edit |
| Git | ❌ | ✅ commit / branch / PR |
| 외부 도구 | 제한적 | ✅ MCP · CLI · API |
| 멀티 스텝 | 사용자가 일일이 지시 | ✅ 스스로 계획 → 실행 → 검증 |

### 생물학자에게 왜 중요한가

생물정보학 작업의 진짜 어려움은 **분석 자체** 가 아니라 **도구 체인** 입니다.
하나의 RNA-seq 분석에 fastqc · trim_galore · STAR · samtools · featureCounts · DESeq2
가 들어가고, 각각 옵션 수십 개. 외워서는 못 합니다.

코딩 에이전트는 그 외움을 외주화합니다.

---

## 도구 비교 — 어떤 걸 써야 하나

세 가지가 시장을 양분합니다. 결론부터: **Claude Code (CLI) + Cursor (IDE)** 조합이
지금 가장 강력하고, 단일 도구만 쓴다면 **Claude Code** 권장.

### Claude Code
- **형태**: 터미널 CLI (`claude` 명령)
- **모델**: Claude Opus 4.x / Sonnet 4.x / Haiku 4.x — 자동 라우팅
- **요금**: Max/Pro 구독 ($20~$200/월) 또는 API 종량제
- **강점**: 셸 통합 · long context · 자율 멀티 스텝 작업
- **약점**: IDE 통합은 VS Code/JetBrains 확장으로

### Cursor
- **형태**: VS Code fork — 풀 IDE
- **모델**: Claude · GPT · Gemini 등 멀티 백엔드
- **요금**: 무료 / Pro $20/월 / Business $40/월
- **강점**: 익숙한 IDE UX, 코드 편집 중 빠른 보조
- **약점**: CLI/장기 자율 작업은 약함

### Codex (OpenAI)
- **형태**: CLI + 클라우드 컨테이너 에이전트
- **모델**: GPT-5 / o3
- **요금**: ChatGPT Plus·Pro 포함
- **강점**: 클라우드 격리 환경, 병렬 PR
- **약점**: 로컬 파일 접근은 CLI 변형이 따로

{: .tip }
> **추천 조합 (2026년 5월 기준)**
> - 일상 분석·노트북·Bash 자동화: **Claude Code (CLI)**
> - 인터랙티브 코드 작성 / 큰 코드베이스 탐색: **Cursor**
> - 한 PR이 너무 크고 부담될 때: **Codex Cloud** 로 격리 실행
>
> 셋 다 깔아두고 작업에 맞춰 골라 쓰는 것도 좋은 전략.

---

## 설치 — Claude Code 부터

가장 빠른 길.

### macOS / Linux / WSL

```bash
# 권장: 공식 설치 스크립트
curl -fsSL https://claude.ai/install.sh | bash

# 또는 npm
npm install -g @anthropic-ai/claude-code
```

### Windows

WSL2 (Ubuntu) 위에서 위 명령 그대로. 네이티브 Windows는 PowerShell:

```powershell
# Anthropic 공식 PowerShell 설치 스크립트 (확인 후 실행)
irm https://claude.ai/install.ps1 | iex
```

### 로그인

```bash
claude
# /login → 브라우저로 인증
```

API 키를 쓰려면:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

### 첫 실행

```bash
cd ~/my-bio-project
claude
```

여기서 자연어로 시작:

```
> 이 폴더에 있는 FASTQ 파일 두 개의 read 개수랑 평균 길이를 알려줘
```

에이전트가 `seqkit stats` 같은 도구를 알아서 골라 실행합니다.

---

## Cursor 설치

[cursor.com](https://cursor.com) 에서 다운로드. VS Code 확장이 거의 그대로 호환되므로
기존 환경 그대로 이전 가능.

핵심 단축키:
- `Cmd/Ctrl + K` — 인라인 코드 편집
- `Cmd/Ctrl + L` — 사이드바 채팅
- `Cmd/Ctrl + I` — Composer (멀티파일 편집)

생물학자에게 특히 유용:
- `.ipynb` (Jupyter) 파일 안에서 셀별로 `Cmd+K` 가능
- 데이터 파일 미리보기 (`.csv`, `.tsv` 클릭하면 표로)

---

## 미니 실전 — 5분 안에 끝내기

목표: GEO에서 FASTQ 하나 받아 QC 리포트 만들기.

### 전통 방식

```bash
# 1. 도구 설치 (10분 검색 + 트러블슈팅)
conda install -c bioconda sra-tools fastqc multiqc

# 2. 데이터 받기 (옵션 외우기)
prefetch SRR12345678
fasterq-dump --split-files SRR12345678

# 3. QC
fastqc *.fastq
multiqc .
```

매 단계마다 옵션·에러·경로 문제로 막힘.

### 에이전트 방식

```bash
cd ~/scratch && mkdir geo-test && cd geo-test
claude
```

```
> GEO 시리즈 GSE183947에서 첫 번째 sample의 FASTQ를 받고 fastqc / multiqc로
> QC 리포트까지 만들어 줘. 도구가 없으면 conda로 설치해도 돼.
```

에이전트가:
1. SRA accession 조회 (`pysradb` 또는 직접 ESummary)
2. 도구 설치 확인 → 필요시 설치
3. 다운로드 → QC 실행
4. 결과 요약 + HTML 리포트 위치 알려줌

당신은 **결과의 타당성** 만 검증하면 됩니다.

{: .warning }
> 에이전트가 "성공" 이라고 말해도, **숫자가 말이 되는지** 는 반드시 확인하세요.
> 자주 발생하는 실수: paired-end인데 single로 처리 / wrong reference genome /
> adapter trimming 빼먹기. 이건 [실전 워크플로우](workflows#실패-모드-체크리스트)
> 에서 다룹니다.

---

## 권한 / 안전 가이드

에이전트는 강력하지만 **당신 컴퓨터에서 임의 명령을 실행** 합니다.

기본 원칙:
1. **분석 폴더 단위로 격리** — `cd ~/scratch/proj-X && claude` 식으로
2. **`rm -rf`, `git push --force`, 클라우드 비용 발생 명령은 항상 확인 후 승인**
3. **API 키 등 secret 은 환경변수로** — 코드에 박지 말 것
4. **컨테이너 격리** — Docker/Apptainer 안에서 돌리면 안전성 ↑

Claude Code의 권한 시스템:
```
/permissions          # 현재 허용 목록 확인
/permissions allow Bash(conda:*)   # conda 명령만 허용
```

[oh-my-claudecode 가이드](../omc-guide/) 에서 `--yolo` / `--madmax` 같은
"모든 권한 자동 승인" 모드 설명도 참고. 격리된 환경에서만 쓰세요.

---

## 자주 묻는 질문

**Q. 영어로 시켜야 하나요?**<br>
A. 한국어 자연어로 잘 됩니다. 단, **도구·옵션 이름은 영어 원형** 으로 쓰세요.
"DESeq2 돌려" 가 "디지큐2 돌려" 보다 훨씬 잘 인식됩니다.

**Q. Mac M1/M2 에서도 되나요?**<br>
A. 네. 단 일부 bioconda 도구는 osx-arm64 빌드가 없습니다. Rosetta 또는 Linux Docker
컨테이너를 권장.

**Q. 인터넷 없는 환경(병원·연구기관 인트라넷)에서는?**<br>
A. Claude Code는 API 호출이 필수입니다. 인터넷 없는 환경이면 **온프레미스 LLM
(Llama 3.3, Qwen 2.5)** + **aider** 같은 오픈소스 코딩 에이전트 조합 검토.

**Q. 비용은 얼마나 들죠?**<br>
A. 일반적인 분석 세션 1시간 = $0.5 ~ $5 (API 직접 호출 기준). Max 구독은
사용량 큰 사람에게 유리.

---

## 다음

설치가 끝났다면:

→ **[2. 전문 Bio AI 모델 카탈로그](bio-ai-models)** — 코딩 에이전트 너머,
"단백질 구조 예측" 같은 specialized 모델로.
