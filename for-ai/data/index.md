---
title: 데이터 레이어
parent: AI → Bio 트랙
nav_order: 2
has_children: true
permalink: /for-ai/data/
---

# 데이터 레이어 — Omics 트리
{: .no_toc }

생물학 데이터를 데이터 종류 기준으로 잘라낸 트리. 같은 분석 작업이라도 데이터
종류에 따라 도구가 완전히 달라집니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## "Omics" 개념

`-omics` 접미사 = "전체를 한꺼번에 본다".

| Omics | 무엇의 전체 | AI 엔지니어 시선 |
|---|---|---|
| Genomics | DNA 전체 | 시퀀스 (3Gb) |
| Transcriptomics | RNA 발현 전체 | gene × sample matrix |
| Proteomics | 단백질 전체 | mass spec spectra → peptide |
| Metabolomics | 대사체 전체 | mass spec → molecule |
| Epigenomics | DNA 변형 (메틸화 등) 전체 | position × signal |
| Microbiome | 미생물 군집 전체 | taxon × sample |

---

## 페이지 트리

이 섹션은 다음 페이지들로 구성됩니다.

### 1. [Genomics](genomics)
DNA. FASTA / FASTQ / BAM / VCF. 변이 호출, 유전체 어셈블리, CRISPR.
**AI 모델**: Evo, Nucleotide Transformer, DNABERT-2, HyenaDNA, AlphaMissense.

### 2. [Transcriptomics](transcriptomics)
Bulk RNA-seq. Count matrix. DEG 분석. **AI 모델**: 전통 DESeq2/edgeR이 여전히 표준,
foundation model (Enformer 등) 은 보조.

### 3. [Single-cell](single-cell)
세포 단위 transcriptome. h5ad. UMAP/cluster/marker.
**AI 모델**: scGPT, Geneformer, UCE, scFoundation.

### 4. [Proteomics](proteomics)
질량분석 또는 단백질 시퀀스. mzML / FASTA.
**AI 모델**: ESM-2/3, ProtT5, AlphaMissense, DIA-NN.

### 5. [Structural](structural)
단백질 3D 구조. PDB / CIF.
**AI 모델**: AlphaFold 3, Boltz-1/2, ESMFold, RoseTTAFold All-Atom.

### 6. [기타](others)
Epigenomics (ATAC-seq, ChIP-seq, methylation), metabolomics, microbiome (16S, shotgun),
spatial transcriptomics, imaging (cryo-EM, histology) — 한 페이지에 묶어 개요.

---

## "어디부터 봐야 하나" 빠른 가이드

| 받은 파일 | 가능성 | 가서 볼 페이지 |
|---|---|---|
| `*.fastq.gz` | 무엇이든 시퀀싱 raw | 메타데이터 확인 필요 |
| `*.bam` | 정렬된 reads | [Sequence analysis](../tasks/sequence-analysis) |
| `*.vcf` / `*.vcf.gz` | 변이 | [Variant calling](../tasks/variant-calling) |
| `*.h5ad` | scRNA-seq | [Single-cell](single-cell) |
| `*.h5` (10x output) | scRNA-seq raw | [Single-cell](single-cell) |
| `counts.txt` / `count_matrix.csv` | bulk RNA-seq | [Transcriptomics](transcriptomics) |
| `*.pdb` / `*.cif` | 단백질 구조 | [Structural](structural) |
| `*.mzML` / `*.raw` | mass spec | [Proteomics](proteomics) |
| `*.bigwig` / `*.bedGraph` | signal track | [기타](others) (epigenomics) |
