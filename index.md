---
title: 홈
layout: default
nav_order: 1
description: "Bio 전공자가 AI를 잘 쓰고, AI 엔지니어가 Bio를 잘 이해하게 하는 양방향 가이드."
permalink: /
---

# Bioinformatics × AI Engineers
{: .fs-9 }

생물학을 코드처럼, 코드를 생물학처럼 다루는 사람들을 위한 양방향 가이드.
{: .fs-6 .fw-300 }

[Bio 전공자 → AI](for-bio/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[AI 엔지니어 → Bio](for-ai/){: .btn .fs-5 .mb-4 .mb-md-0 .mr-2 }
[통합 케이스](bridge/){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## 왜 이 사이트인가

2024년 이후 생명정보학(Bioinformatics)은 두 개의 거대한 흐름과 동시에 부딪쳤습니다.

1. **AlphaFold 3 · ESM-3 · Boltz · Evo** 같은 **specialized bio foundation models** 의 일반화.
2. **Claude Code · Cursor · Codex** 같은 **coding agent** 와 **MCP harnessing** 의 폭발적 성장.

결과적으로 한쪽에는 "AI 도구가 너무 많아서 어디부터 손대야 할지 모르겠는" 생물학자가,
다른 쪽에는 "유전체 데이터를 받았는데 FASTQ가 뭔지부터 막힌" AI 엔지니어가 있습니다.

이 사이트는 그 사이의 다리를 놓습니다.

{: .note }
> **이 사이트의 입장**
> 생명정보학을 가르치는 자료는 이미 많고, AI 도구 튜토리얼도 넘칩니다.
> 이 사이트의 역할은 **둘을 잇는 운영 매뉴얼**입니다 — 어떤 작업에 어떤 모델을 쓰고,
> 어떤 데이터에 어떤 에이전트가 적합한지를 한 화면에서 비교할 수 있게.

---

## 두 트랙

### 🧬 → 🤖  [Bio 전공자를 위한 AI 트랙](for-bio/)

세포배양·시퀀싱·Bash 스크립트를 만지던 사람이 **AI를 일상 도구로** 만드는 법.

- **[Coding Agents 입문](for-bio/coding-agents)** — Claude Code, Cursor, Codex.
  내 노트북에서 `bwa mem` 한 줄 외우는 대신 자연어로 파이프라인 짜기.
- **[전문 Bio AI 모델](for-bio/bio-ai-models)** — AlphaFold, ESM-2/3, Boltz-1, Evo,
  scGPT, Geneformer. 어떤 문제에 어떤 모델인가.
- **[Agent · MCP · Harness](for-bio/agents-mcp)** — 한 번에 한 명령이 아니라,
  "분석 전체"를 위임하는 자율 에이전트 아키텍처.
- **[실전 워크플로우](for-bio/workflows)** — RNA-seq, variant calling, 단백질 설계 ·
  실제 사용 패턴.

### 🤖 → 🧬  [AI 엔지니어를 위한 Bio 트랙](for-ai/)

PyTorch와 Transformers는 익숙하지만 BAM 파일을 처음 보는 사람을 위한 입문.

체계는 **이중 트리** — 같은 데이터라도 어떤 작업을 하느냐에 따라 다른 도구가 쓰입니다.

| 축 | 무엇 | 페이지 |
|---|---|---|
| **데이터** | DNA·RNA·단백질·세포 단위의 데이터 형태 | [데이터 레이어](for-ai/data/) |
| **작업** | 정렬·변이 호출·발현 분석·구조 예측·임상 해석 | [작업 레이어](for-ai/tasks/) |

먼저 **[멘탈 모델](for-ai/mental-model)** 부터 — DNA에서 표현형까지의
information flow 한 장 요약.

### 🌉  [통합 케이스 스터디](bridge/)

추상적인 설명 말고, 실제로 두 분야를 묶었을 때 어떻게 돌아가는지.

- **[RNA-seq을 Claude Code로](bridge/case-rnaseq)** — 셸에 익숙하지 않은
  생물학자가 에이전트로 처음 끝까지 파이프라인을 돌리는 과정.
- **[Inverse Folding 루프](bridge/case-protein-design)** — AI 엔지니어가
  ESM-IF/ProteinMPNN을 받아 wet-lab 피드백 루프를 코드로 닫는 과정.

---

## 빠른 시작

자신에게 해당하는 한 줄을 따라가세요.

> "나는 **세포생물학 전공**, 파이썬은 조금. ChatGPT 정도 써본 정도." <br>
> → [Coding Agents 입문](for-bio/coding-agents) 부터.

> "나는 **NGS·시퀀싱 데이터 다루는 박사과정**. 셸은 익숙하지만 AI 모델은 처음." <br>
> → [Bio AI 모델 카탈로그](for-bio/bio-ai-models) → [Agent · MCP](for-bio/agents-mcp).

> "나는 **AI/ML 엔지니어**, Transformers·PyTorch 잘 다룸. Bio는 학부 생물 이후 처음." <br>
> → [멘탈 모델](for-ai/mental-model) → [데이터 레이어](for-ai/data/).

> "나는 **의공학·계산화학 출신**, 단백질 구조 모델은 안다. 시퀀싱 데이터는 약함." <br>
> → [데이터: Genomics](for-ai/data/genomics) → [작업: Variant calling](for-ai/tasks/variant-calling).

---

## 사이트는 진행형

이 가이드는 살아 있는 문서입니다. 빠르게 변하는 두 분야를 다루는 만큼,
빠진 모델·새 도구·잘못된 설명은 [Issues](https://github.com/) 로 알려주세요.

{: .tip }
> 모든 페이지의 **마지막 갱신일** 과 **출처/추가 읽을거리** 는 페이지 하단에 있습니다.
> 빠르게 낡는 분야이므로, 6개월 이상 갱신이 없는 정보는 비판적으로 받아들이세요.
