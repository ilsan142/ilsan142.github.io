---
title: Bridge 케이스 스터디
layout: default
nav_order: 4
has_children: true
permalink: /bridge/
---

# Bridge — 통합 케이스 스터디
{: .no_toc }

"AI도 알고 Bio도 안다" 의 진짜 의미는 두 분야가 한 워크플로우 안에서
**서로의 약점을 메워줄 때** 드러납니다. 이 섹션은 그 실제 모습.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 왜 케이스 스터디인가

앞의 두 트랙은 **카탈로그** 입니다. 어떤 도구가 있고, 어떤 데이터가 있는지.
하지만 실제 연구는 카탈로그를 외운다고 굴러가지 않습니다.

이 섹션은 다음 형식으로 구성됩니다:

1. **문제 정의** — 실제 연구실에서 생기는 시나리오
2. **데이터/도구** — 무엇을 받고 무엇으로 푸는지
3. **AI 없이** — 전통적으로 어떻게 했는지
4. **AI와 함께** — Claude Code · agent · bio foundation model 을 어떻게 끼우는지
5. **실패 모드** — 어떤 식으로 깨지고, 어떻게 회복하는지

---

## 케이스 목록

### [Case A. RNA-seq 파이프라인을 Claude Code로](case-rnaseq)
**대상**: 생물학 전공자, 셸 능숙도 중하 <br>
**주제**: GEO에서 데이터 받아 STAR → featureCounts → DESeq2 → enrichment 까지,
손으로 명령 외우지 않고 에이전트에 위임하는 법. 어디까지 위임 가능하고 어디서
사람이 검증해야 하는지.

### [Case B. Inverse Folding 설계 루프](case-protein-design)
**대상**: AI 엔지니어, ML 능숙 / Bio 입문 <br>
**주제**: ProteinMPNN · ESM-IF · AlphaFold 3 를 묶어 "원하는 구조 → 시퀀스"
역설계 루프 만들기. wet-lab 피드백을 코드 루프에 어떻게 끼우는지.

---

## 앞으로 추가할 케이스 (계획)

- **Case C. Variant → Clinical** : VCF + 전자의무기록 RAG로 희귀질환 후보 좁히기
- **Case D. Single-cell trajectory** : scRNA-seq 데이터에서 scGPT + Pseudotime
- **Case E. Cryo-EM density → atomic model** : AlphaFold 3 + ModelAngelo
- **Case F. CRISPR guide design** : DNABERT-2 임베딩으로 off-target 예측

기여 환영 — Issues / PR 로.
