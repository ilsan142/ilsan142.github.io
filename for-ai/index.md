---
title: AI → Bio 트랙
layout: default
nav_order: 3
has_children: true
permalink: /for-ai/
---

# AI 엔지니어를 위한 Bioinformatics 트랙
{: .no_toc }

PyTorch · Transformers · MLOps는 익숙하지만 BAM 파일을 처음 보는 사람을 위한 입문.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 왜 AI 엔지니어가 Bioinformatics를 알아야 하나

AI 산업에서 **단일 분야 중 가장 큰 미해결 도메인 데이터셋** 이 생물학에 있습니다.
GPT가 텍스트를 먹어치웠다면, 그다음에 먹어치울 것은 단백질·유전체·세포 데이터입니다.
지난 3년 동안 일어난 일:

- **AlphaFold 2/3** (DeepMind) — 단백질 구조 예측을 사실상 해결
- **ESM-2/3** (Meta) — 단백질 언어 모델, 65M~98B 파라미터
- **Evo / Evo-2** (Arc Institute) — DNA foundation model, 7B → 40B
- **scGPT / Geneformer / UCE** — 단일세포 foundation model
- **Boltz-1** (MIT) — 오픈소스 AlphaFold 3 급 구조 모델

이 전부가 **2022년 이후** 일입니다. AI 엔지니어가 진입하기 가장 좋은 시기.

---

## 학습 경로

### Step 1. [멘탈 모델](mental-model)

DNA → RNA → 단백질 → 세포 → 표현형. 데이터 형태(FASTQ/FASTA/BAM/VCF/h5ad)와
정보 흐름의 매핑. **여기를 안 보면 나머지가 안 읽힙니다.**

### Step 2. 이중 트리

같은 데이터라도 어떤 질문을 던지느냐에 따라 다른 도구·다른 모델이 쓰입니다.
그래서 이 사이트는 두 축으로 자릅니다.

#### 데이터 축 — [데이터 레이어](data/)

| 데이터 | 무엇 | 대표 포맷 | 대표 AI 모델 |
|---|---|---|---|
| [Genomics](data/genomics) | DNA 시퀀스·변이 | FASTQ, BAM, VCF | Evo, Nucleotide Transformer, DNABERT-2 |
| [Transcriptomics](data/transcriptomics) | RNA 발현량 | FASTQ, count matrix | DESeq2, edgeR (전통) / scGPT (modern) |
| [Single-cell](data/single-cell) | 세포 단위 발현 | h5ad, mtx | scGPT, Geneformer, UCE |
| [Proteomics](data/proteomics) | 단백질 시퀀스·정량 | FASTA, mzML | ESM-2/3, ProtT5 |
| [Structural](data/structural) | 단백질 3D 구조 | PDB, mmCIF | AlphaFold 3, Boltz-1, ESMFold |
| [기타](data/others) | 후성유전·대사·이미징·미생물군 | varies | varies |

#### 작업 축 — [작업 레이어](tasks/)

| 작업 | 입력 → 출력 | 전통 도구 | AI 도구 |
|---|---|---|---|
| [Sequence analysis](tasks/sequence-analysis) | reads → alignment | BWA, minimap2, STAR | (호환) |
| [Variant calling](tasks/variant-calling) | BAM → VCF | GATK, DeepVariant | DeepVariant, NVIDIA Parabricks |
| [Expression analysis](tasks/expression-analysis) | count → DEG | DESeq2, edgeR, Seurat | scGPT, Geneformer |
| [Structure prediction](tasks/structure-prediction) | 시퀀스 → 3D | Rosetta | AlphaFold 3, Boltz-1, RoseTTAFold |
| [Clinical](tasks/clinical) | VCF → 진단 | ClinVar, OMIM | LLM RAG, AlphaMissense |

### Step 3. [Bridge — 통합 케이스](../bridge/)

이론 한 번 훑은 뒤에 실제 워크플로우 두 개.

---

## 이 트랙이 가정하는 배경

- Python 능숙, PyTorch 또는 JAX 둘 중 하나 익숙
- Transformers 동작 원리는 안다 (논문은 안 읽어도 됨)
- 기초 통계 (p-value, multiple testing) 안다
- 생물 지식은 **고등학교 수준** 이거나 그 이하여도 OK

---

## 이 트랙이 다루지 않는 것

- 분자생물학·세포생물학의 깊은 이론 — 교과서 권장 (Alberts, *Molecular Biology of the Cell*)
- Wet-lab 실험 프로토콜
- 임상시험 통계·규제 (별도 자료 권장)

---

## 다음

[1. 멘탈 모델](mental-model) 부터 시작 → 본인 관심 데이터/작업 페이지로 이동.
