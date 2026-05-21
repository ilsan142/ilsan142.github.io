---
title: Bio → AI 트랙
layout: default
nav_order: 2
has_children: true
permalink: /for-bio/
---

# Bio 전공자를 위한 AI 트랙
{: .no_toc }

세포배양 · 시퀀싱 · 단백질 정제 · R / Bash 스크립트를 만지던 사람이 AI를
**일상 도구로** 받아들이는 법.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 이 트랙이 가정하는 독자

- 생물학·약학·의학·생명공학·분자생물학 등을 전공 또는 연구
- R, Python, Bash 중 적어도 하나는 어색하지 않게 다룸
- ChatGPT 정도는 써 봤지만, **에이전트나 API는 거의 처음**
- 분석을 "코드로 자동화" 하는 수준까지 가본 적은 별로 없음

해당되는 분이면 아래 4개 챕터를 순서대로 보길 권장합니다.

---

## 4개 챕터

### 1. [Coding Agents 입문](coding-agents)

`bwa mem`, `samtools view`, `STAR --runThreadN ...` 같은 명령을 외우는 시대는 지나갔습니다.
**Claude Code · Cursor · Codex** 세 도구를 비교하고, 생물학자의 일상에 맞춰 설치·
세팅·첫 번째 미니 분석까지 끌고 갑니다.

> 결과물: 본인 노트북에서 자연어로 FASTQ 한 쌍을 받아 QC 리포트까지 자동 생성.

### 2. [전문 Bio AI 모델 카탈로그](bio-ai-models)

LLM이 모든 걸 해주지는 않습니다. **특정 생물학 문제에 특화된 foundation model** 들을
한 표로 비교 — 단백질 구조(AlphaFold 3, Boltz-1, ESMFold), 단백질 시퀀스(ESM-2/3,
ProtT5), 유전체(Evo, Nucleotide Transformer, DNABERT-2), 단일세포(scGPT, Geneformer,
UCE), 약물 분자(ChemBERTa, MolFormer).

> 결과물: 자기 문제에 어떤 모델이 맞는지 판단할 수 있는 의사결정 트리.

### 3. [Agent · MCP · Harness 아키텍처](agents-mcp)

한 번에 한 명령이 아닌, **분석 전체를 위임**하는 자율 에이전트 시대.
oh-my-claudecode 같은 harness, MCP 서버, multi-agent orchestration 의 원리.

> 결과물: "이 GEO ID 받아서 DEG 분석부터 enrichment까지 돌려" 한 줄 위임.

### 4. [실전 워크플로우](workflows)

이론은 잠깐, 실제로 자주 마주치는 시나리오:
RNA-seq, single-cell QC, variant calling, 단백질-리간드 도킹.
각각 **"AI 없이"** vs **"AI와 함께"** 의 차이를 코드 단위로 비교.

---

## 이 트랙이 다루지 않는 것

- 머신러닝 기초 (선형회귀·CNN·Transformer 구조 등) — Coursera/3blue1brown 권장
- LLM API의 토큰 가격 최적화 같은 운영 디테일 — [oh-my-claudecode 가이드](../omc-guide/) 참고
- AI 윤리·생물안보 이슈 (별도 페이지로 분리 예정)

---

## 다음

위 1번부터 순서대로 진행하거나, 본인 상황에 맞는 곳부터:

| 상황 | 추천 시작점 |
|---|---|
| 노트북에 Claude Code 깔아본 적 없음 | [1. Coding Agents 입문](coding-agents) |
| AI 도구는 좀 쓰는데 단백질 모델 분야가 너무 빠르게 변함 | [2. Bio AI 모델 카탈로그](bio-ai-models) |
| 이미 Claude Code/Cursor 일상 사용, 자동화 더 깊이 | [3. Agent · MCP](agents-mcp) |
| "내 데이터로 실제 돌려보고 싶다" | [4. 실전 워크플로우](workflows) |
