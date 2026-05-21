---
title: Transcriptomics
parent: 데이터 레이어
grand_parent: AI → Bio 트랙
nav_order: 2
---

# Transcriptomics — RNA / Bulk RNA-seq
{: .no_toc }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 핵심 개념

**Transcriptomics** = 어떤 RNA가 얼마나 발현되는지 측정.

- **Bulk RNA-seq** — sample 단위로 RNA를 추출, 평균 발현 측정 (이 페이지)
- **Single-cell RNA-seq** — 세포 단위 → [별도 페이지](single-cell)
- **Spatial transcriptomics** — 조직 슬라이드 위 좌표 정보 + 발현 → [others](others)

---

## 데이터 형식

### Raw — FASTQ
**Paired-end** (`_R1`, `_R2`) 가 표준. Single-end는 거의 사라짐.

### Aligned — BAM
Splice-aware aligner (STAR, HISAT2) 가 필수. RNA는 intron을 건너뛰므로
일반 DNA aligner (BWA) 로 정렬하면 안 됨.

### Quantified — Count matrix

```
                sample1  sample2  sample3
ENSG00000141510  1234     1100     1300
ENSG00000012048  567      612      598
...
```

이 행렬이 **Transcriptomics의 핵심**.

도구별 출력:
- `featureCounts` (Subread) → tab-separated text
- `Salmon` → quant.sf (TPM, NumReads)
- `STAR --quantMode GeneCounts` → ReadsPerGene.out.tab

### 표현 단위

| 단위 | 의미 |
|---|---|
| **Counts** | raw read 개수 (정수) |
| **CPM** | Counts Per Million |
| **TPM** | Transcripts Per Million (길이 보정) |
| **FPKM** / **RPKM** | (옛날 단위, 비추천) |

**중요**: DEG 분석에는 **raw counts** 를 쓰고, 시각화나 비교에만 TPM 사용.
DESeq2 / edgeR 은 자체적으로 정규화.

---

## 대표 워크플로우 — Bulk RNA-seq

```
FASTQ ─→ [QC] fastqc/multiqc
       ─→ [Trim] trim_galore / fastp
       ─→ [Align] STAR / HISAT2 ─→ BAM
                                  ├→ [Count] featureCounts → count matrix
                                  └→ [Quant] Salmon (alignment-free 대안)
                                       ↓
                              [DEG] DESeq2 / edgeR / limma-voom
                                       ↓
                              [Enrichment] clusterProfiler / GSEA
                                       ↓
                                     해석
```

### Alignment-free 대안

`Salmon`, `Kallisto` — BAM 안 만들고 바로 transcript 정량. 10배 빠름.

```bash
salmon quant -i salmon_index -l A \
             -1 sample_R1.fq.gz -2 sample_R2.fq.gz \
             -p 16 -o quant_sample
```

---

## DEG 분석

Differentially Expressed Gene = 그룹 간 발현이 통계적으로 다른 유전자.

### 통계 모델

RNA-seq count는 **negative binomial 분포** 를 따름 (overdispersed Poisson).

| 도구 | 모델 |
|---|---|
| **DESeq2** | NB GLM, Wald test, shrinkage 잘 됨 |
| **edgeR** | NB GLM, quasi-likelihood |
| **limma-voom** | log-CPM에 weight 부여 후 linear model |

### 코드 — DESeq2

```r
library(DESeq2)
cts <- read.table("counts.txt", row.names=1, header=TRUE)
coldata <- data.frame(
  condition = factor(c("ctrl","ctrl","ctrl","trt","trt","trt")),
  row.names = colnames(cts)
)
dds <- DESeqDataSetFromMatrix(countData=cts, colData=coldata, design=~condition)
dds <- DESeq(dds)
res <- results(dds, contrast=c("condition","trt","ctrl"))

# 표준 필터
deg <- subset(res, padj < 0.05 & abs(log2FoldChange) > 1)
```

---

## AI / Foundation Model 의 역할

Bulk RNA-seq는 여전히 **전통 통계가 표준**. Foundation model은 보조적.

### 어디서 AI가 도움되나

1. **임상 표현형 예측** — RNA 발현 → 환자 생존, 약물 반응. Random Forest / DL.
2. **Cell type deconvolution** — bulk RNA를 scRNA-seq reference로 분해.
   `CIBERSORT`, `MuSiC`, `Bisque`. 최근엔 ML 기반 `Scaden`.
3. **Pathway 추론** — `Enformer` 같은 regulatory model로 입력 시퀀스 → 발현 예측.

### "왜 LM이 bulk에 덜 쓰이나"
- bulk = 평균. 노이즈 적고, 통계 가정이 잘 맞음.
- Foundation model의 진가는 sparse / high-dim (single-cell) 에서 드러남.

---

## 코드 예 — 전체 파이프라인

```bash
# STAR alignment
STAR --runThreadN 16 --genomeDir $STAR_IDX \
     --readFilesIn ${S}_R1.fq.gz ${S}_R2.fq.gz \
     --readFilesCommand zcat --outSAMtype BAM SortedByCoordinate \
     --outFileNamePrefix bam/${S}_ \
     --quantMode GeneCounts

# featureCounts
featureCounts -p --countReadPairs -T 16 -s 2 \
              -a gencode.v45.annotation.gtf -o counts.txt bam/*.bam
```

```python
# python으로 DESeq2 — PyDESeq2
from pydeseq2.dds import DeseqDataSet
from pydeseq2.ds import DeseqStats
import pandas as pd

counts = pd.read_csv("counts.txt", sep="\t", index_col=0)
metadata = pd.read_csv("metadata.csv", index_col=0)

dds = DeseqDataSet(counts=counts.T, metadata=metadata, design_factors="condition")
dds.deseq2()
stats = DeseqStats(dds, contrast=["condition","trt","ctrl"])
stats.summary()
```

---

## 자주 마주치는 함정

### 1. **Strandedness**
`-s 0` (unstranded) / `-s 1` (forward) / `-s 2` (reverse).
모르면 RSeQC의 `infer_experiment.py` 로 추정.

### 2. **rRNA contamination**
RNA 추출에서 rRNA depletion 안 했으면 reads의 70%가 rRNA. 정상 lib면 < 10%.

### 3. **Batch effect**
다른 날 시퀀싱한 샘플은 batch로 처리. `~ batch + condition` 으로 design에 넣기.

### 4. **Multiple testing**
`p.adjust` 안 하면 거의 모든 유전자가 "significant"로 보임.
DESeq2는 BH-FDR 자동, **`padj` 사용**.

### 5. **Low-count gene**
1-2 read짜리 유전자는 통계가 부정확. `rowSums(counts) > 10` 으로 사전 필터.

---

## 데이터 받을 수 있는 곳

- **GEO** (NCBI) — Series accession (GSE...) 단위
- **ArrayExpress / Biostudies** (EBI)
- **recount3** — 전처리된 count matrix (수천 GEO 시리즈)
- **TCGA** — 33가지 암, RNA-seq + 임상

---

## 다음

- 작업: [Expression analysis](../tasks/expression-analysis)
- 관련: [Single-cell](single-cell)
