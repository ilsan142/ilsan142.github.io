---
title: 작업 레이어
parent: AI → Bio 트랙
nav_order: 3
has_children: true
permalink: /for-ai/tasks/
---

# 작업 레이어 — Task 트리
{: .no_toc }

생물학 데이터를 "무엇을 알아내고 싶은가" 의 작업(task) 기준으로 잘라낸 트리.
같은 BAM 파일이라도 무엇을 묻느냐에 따라 도구가 완전히 달라집니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## "데이터 축" 과 "작업 축" 이 다른 이유

[데이터 레이어](../data/) 는 **"내가 받은 파일이 무엇인가"** 를 축으로 한다면,
이 작업 레이어는 **"그 파일로 무엇을 풀려는가"** 를 축으로 합니다.

같은 데이터 — 다른 작업의 예시.

| 같은 입력 | 작업 A | 작업 B | 작업 C |
|---|---|---|---|
| BAM (정렬된 reads) | variant calling (→ VCF) | expression quantification (→ counts) | peak calling (→ BED) |
| FASTA (단백질) | 구조 예측 (→ PDB) | function annotation (→ GO term) | mutation 효과 (→ score) |
| Count matrix | DEG 분석 | clustering | trajectory inference |

→ **도구 선택은 항상 "데이터 × 작업" 의 교차점** 에서 결정됩니다.

---

## 자식 페이지 한 줄 요약

| # | 페이지 | 입력 → 출력 | 핵심 도구 |
|---|---|---|---|
| 1 | [Sequence analysis](sequence-analysis) | FASTQ → BAM (또는 quant) | bwa-mem2, minimap2, STAR, HISAT2 |
| 2 | [Variant calling](variant-calling) | BAM → VCF | GATK, DeepVariant, Mutect2, Clair3 |
| 3 | [Expression analysis](expression-analysis) | count matrix → DEG / clusters | DESeq2, edgeR, Seurat, scanpy |
| 4 | [Structure prediction](structure-prediction) | 단백질 시퀀스 → 3D | AlphaFold 3, Boltz-1/2, ESMFold |
| 5 | [Clinical interpretation](clinical) | VCF → 진단·report | VEP, AlphaMissense, ClinVar, LLM RAG |

---

## "어디부터 시작해야 하나" — 입력 기준 가이드

받은 입력 파일로부터 가야 할 페이지를 찾는 빠른 트리.

```
받은 파일?
│
├─ *.fastq[.gz]  ──→  먼저 [Sequence analysis] (정렬)
│                     그 다음 목적에 따라:
│                       · DNA → [Variant calling]
│                       · RNA → [Expression analysis]
│
├─ *.bam / *.cram ──→  목적이 무엇?
│                       · 변이 찾기      → [Variant calling]
│                       · 발현 정량      → [Expression analysis]
│                       · ChIP/ATAC peak → (별도, 추후 추가)
│
├─ *.vcf[.gz]    ──→  [Clinical interpretation] (해석)
│                     또는 annotation 정리
│
├─ count matrix  ──→  [Expression analysis]
│  (counts.txt, h5ad, mtx)
│
└─ FASTA (단백질) ──→  [Structure prediction]
   또는 PDB 있음        또는 docking (Structure prediction 안에)
```

{: .tip }
> 입력이 무엇인지 헷갈리면 먼저 [데이터 레이어](../data/) 의 형식 표를 보세요.
> "이 확장자가 어떤 데이터인지" 부터 잡고 와야 작업 페이지가 의미 있게 읽힙니다.

---

## 목적 기준 가이드 — "내가 풀려는 질문은?"

| 질문 | 가야 할 페이지 |
|---|---|
| "이 환자의 유전 변이는?" | [Variant calling](variant-calling) → [Clinical](clinical) |
| "약물 처리 전후 어떤 유전자가 변했나?" | [Expression analysis](expression-analysis) (bulk DEG) |
| "샘플 안에 어떤 세포 타입이 있나?" | [Expression analysis](expression-analysis) (single-cell) |
| "이 단백질은 어떻게 생겼나?" | [Structure prediction](structure-prediction) |
| "이 약물이 단백질에 붙을까?" | [Structure prediction](structure-prediction) (docking) |
| "FASTQ 받았는데 일단 정렬부터" | [Sequence analysis](sequence-analysis) |
| "VCF의 missense 변이가 해롭나?" | [Clinical](clinical) (AlphaMissense) |

---

## AI 도구가 들어가는 지점

각 작업에서 **AI 모델이 SOTA인 부분** 과 **여전히 전통 도구가 표준인 부분** 의 구분.

| 작업 | 전통 도구가 표준 | AI가 SOTA 또는 강력한 보조 |
|---|---|---|
| Sequence analysis | 정렬 (bwa-mem2, minimap2) | Basecalling (Dorado, Bonito), DeepConsensus |
| Variant calling | germline 표준 (GATK) | DeepVariant, Clair3, AlphaMissense (해석) |
| Expression (bulk) | DESeq2, edgeR | (limited — 전통이 강함) |
| Expression (single-cell) | Seurat, scanpy 파이프라인 | Foundation models (scGPT, Geneformer) |
| Structure prediction | (전통은 사실상 끝) | **AlphaFold 3, Boltz-1/2 — 압도** |
| Clinical | VEP, ClinVar 매칭 | AlphaMissense, LLM RAG, AlphaGenome |

→ 자세한 비교는 각 자식 페이지에서.

---

## 다음

→ [Sequence analysis](sequence-analysis) 부터 시작 (FASTQ → BAM)
→ 데이터 형식이 헷갈리면 [데이터 레이어](../data/) 로 돌아가기
