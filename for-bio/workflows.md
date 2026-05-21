---
title: 4. 실전 워크플로우
parent: Bio → AI 트랙
nav_order: 4
---

# 실전 워크플로우
{: .no_toc }

이론은 끝. 실제 분석을 **AI 없이** vs **AI와 함께** 비교로.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 어디까지 위임할 수 있나

원칙 한 줄: **"숫자가 나오는 단계는 사람이 검증, 그 사이 보일러플레이트는 위임."**

| 단계 | 위임 가능? | 이유 |
|---|---|---|
| 도구 설치 / 환경 구성 | ✅ 거의 100% | 시간 낭비, 실수 잦음 |
| 데이터 다운로드 / 정리 | ✅ 90% | 단, accession 검증 필요 |
| QC / 전처리 | ⚠️ 70% | 임계치 결정은 사람 |
| Alignment / quantification | ✅ 90% | 도구 옵션 자동화 가능 |
| **통계 분석 (DEG, GWAS)** | ⚠️ 50% | **반드시 사람 검증** |
| 시각화 | ✅ 95% | 빠른 반복 가능 |
| 결론 / 해석 | ❌ 거의 0% | 도메인 지식 필요 |

---

## 워크플로우 1 — Bulk RNA-seq (DEG)

### 시나리오

대조군 3개 · 처리군 3개의 paired-end FASTQ를 받았다. Differentially expressed gene
(DEG) 리스트와 enrichment 결과를 내야 한다.

### 전통 방식 — 일주일

```bash
# Day 1 — 환경 구성
conda create -n rnaseq python=3.11
conda install -c bioconda fastqc trim-galore star samtools subread
Rscript -e 'BiocManager::install(c("DESeq2","clusterProfiler","org.Hs.eg.db"))'

# Day 2 — QC
for f in *.fastq.gz; do fastqc $f; done
multiqc .

# Day 3 — Trim
for s in sample{1..6}; do
  trim_galore --paired ${s}_R1.fastq.gz ${s}_R2.fastq.gz
done

# Day 3-4 — STAR index (10GB 다운로드 + 1시간 build)
STAR --runMode genomeGenerate --genomeDir star_idx \
     --genomeFastaFiles GRCh38.fa --sjdbGTFfile gencode.v45.gtf

# Day 4 — Alignment
for s in sample{1..6}; do
  STAR --runThreadN 16 --genomeDir star_idx \
       --readFilesIn ${s}_R1_val_1.fq.gz ${s}_R2_val_2.fq.gz \
       --readFilesCommand zcat --outSAMtype BAM SortedByCoordinate \
       --outFileNamePrefix bam/${s}_
done

# Day 5 — Count
featureCounts -p --countReadPairs -T 16 -a gencode.v45.gtf \
              -o counts.txt bam/*.bam

# Day 5-6 — DESeq2 in R (별도 스크립트)
# Day 7 — clusterProfiler enrichment + 그림
```

각 단계마다 에러 · 옵션 검색 · 트러블슈팅으로 시간이 무한정 늘어남.

### 에이전트 방식 — 반나절

```bash
mkdir rnaseq-test && cd rnaseq-test
# FASTQ를 이 폴더에 두거나, GEO accession만 알려줘도 됨

claude
```

```
> 이 폴더에 있는 paired-end FASTQ들(sample{1..6}_R{1,2}.fastq.gz)을 STAR + featureCounts로
> 정렬·카운팅해 줘. 메타데이터는 sample1-3이 control, sample4-6이 treatment.
> GRCh38 / Gencode v45 사용. DESeq2 DEG (|log2FC| > 1, padj < 0.05) 까지 R로 돌리고,
> clusterProfiler로 GO BP enrichment까지. 모든 결과는 results/ 에 저장.
```

에이전트가:
1. 도구 부재시 conda 설치
2. STAR index 캐시 확인 (이미 있으면 재사용)
3. QC → trim → align → count
4. R 스크립트 자동 생성 후 실행
5. 결과 파일 + 요약 마크다운 작성

**사람의 역할**:
- 메타데이터 정확성 검증 (sample-condition 매핑)
- DESeq2 결과의 dispersion plot, MA plot 시각 검토
- enrichment 결과의 생물학적 타당성

### 위임 시 주의점

```
> 결과 요약 줘
```
보다는

```
> 결과 요약 + dispersion plot, PCA plot, sample correlation heatmap, MA plot을
> results/qc/에 저장하고 각각 어떻게 봐야 하는지 코멘트 남겨 줘.
```

이렇게 시키면 **사람이 검증할 자료** 까지 함께 준비됩니다.

---

## 워크플로우 2 — Single-cell RNA-seq QC + 클러스터링

### 시나리오

10x Chromium 데이터 (filtered_feature_bc_matrix). 세포 ~10,000개. PBMC.
QC → 클러스터링 → 마커 유전자 → cell type annotation 까지.

### 에이전트 위임 + foundation model 활용

```
> ./pbmc_10k/filtered_feature_bc_matrix/ 의 10x 데이터로 scanpy 분석.
> 1) 표준 QC (n_genes, percent_mt, doublets)
> 2) normalize → log1p → HVG → scale → PCA → harmony(필요시) → neighbors → UMAP → leiden
> 3) 클러스터별 marker gene (rank_genes_groups, top 10)
> 4) Geneformer로 zero-shot cell type annotation 시도
> 5) marker 기반 manual annotation도 따로
> 6) 두 annotation 비교 표
```

여기서 **Geneformer 단계** 는 [bio-ai-models](bio-ai-models#4-단일세포-single-cell-foundation-models)
에서 다룬 모델을 실제로 끼우는 지점. 에이전트가 `pip install` 부터 임베딩 추출까지 처리.

### 위임 시 사람 검증 포인트

- **QC 임계치** — `percent_mt < 10%` 가 적절한가? 조직마다 다름
- **클러스터 수** — leiden resolution. 너무 잘게 나뉘면 의미 없음
- **마커 유전자 생물학적 일관성** — T cell 클러스터에 CD3 없으면 의심
- **Geneformer 결과 일관성** — manual annotation과 크게 어긋나면 둘 다 의심

---

## 워크플로우 3 — Variant Calling + 임상 해석

### 시나리오

전장 엑솜 (WES) FASTQ. 환자 1명, 부모 2명 trio. 의심 변이 후보를 좁히고 싶다.

### 파이프라인

```
> ./trio/ 폴더의 proband.{R1,R2}.fastq.gz, mother.{R1,R2}.fastq.gz,
> father.{R1,R2}.fastq.gz 로 WES variant calling.
>
> 1) bwa-mem2로 GRCh38에 정렬
> 2) MarkDuplicates → BQSR (GATK)
> 3) HaplotypeCaller → GenomicsDBImport → GenotypeGVCFs (trio joint)
> 4) VQSR 또는 hard filter
> 5) Mendelian inconsistency 체크
> 6) De novo 후보 추출 (proband에만 있는 변이)
> 7) AlphaMissense로 missense 변이 영향 예측
> 8) gnomAD 빈도 < 0.001 + ClinVar P/LP 필터
> 9) HPO 표현형 (HP:0001250, HP:0001263) 기반 우선순위
```

여기는 **여러 specialized 모델** 이 들어갑니다:
- **DeepVariant** (선택지) — GATK HC 대신
- **AlphaMissense** — missense 변이 pathogenicity
- **AlphaFold 3** — 변이가 단백질 구조에 어떤 영향 미치는지 시각화

### 사람만 할 수 있는 일

이 워크플로우의 마지막 — **"이 변이가 환자의 표현형을 설명하는가"** —
는 명백히 사람의 판단입니다. 에이전트는 후보를 100개에서 5개로 줄여 줄 뿐.

---

## 워크플로우 4 — 단백질-리간드 도킹 스크리닝

### 시나리오

타겟 단백질 (예: CDK9) 의 알려진 구조에 화합물 라이브러리 1,000개를 도킹.

### 전통 vs AI

| 단계 | 전통 | 2026년 |
|---|---|---|
| 단백질 구조 | PDB에서 검색 / 없으면 X | AlphaFold 3 / Boltz-1 |
| 결합 사이트 추정 | P2Rank, fpocket | (동일) |
| 도킹 | AutoDock Vina, Glide | DiffDock, **Boltz-2**, AlphaFold 3 |
| 친화도 | scoring function | Boltz-2 친화도 헤드 |
| 후속 합성 | Vendor 카탈로그 | REINVENT 4 (유사체 생성) |

### 에이전트 위임

```
> 타겟: human CDK9 (UniProt P50750). 화합물 SMILES 리스트는 compounds.csv.
>
> 1) AlphaFold 3 또는 Boltz-1로 apo 구조 예측 (이미 PDB 5L1Z 있음 → 그거 사용)
> 2) ATP binding site에 화합물 1000개 도킹 (DiffDock + Boltz-2 친화도)
> 3) 결과 docking score 분포 + 상위 50개 visualize
> 4) 각 상위 화합물의 ADMET을 ChemBERTa-2 + DeepPurpose로 예측
> 5) 상위 10개 후보의 합성 가능성을 AiZynthFinder로 평가
```

이건 **이상적인 미래 시나리오** 입니다 — 2026년 현재 도구들이 다 갖춰져 있긴 하지만
원클릭은 아닙니다. 에이전트가 도구 간 글루(glue) 코드를 자동 생성하는 것이 핵심.

---

## 실패 모드 체크리스트

에이전트가 "성공" 이라 말해도 자주 깨지는 지점:

### 1. 잘못된 reference

에이전트는 종종 GRCh38 / hg19 / GRCm39 를 혼동합니다. 특히 마우스 vs 인간.
**프롬프트에 반드시 명시**:
```
> ... GRCh38.p14 / Gencode v45 (human, primary assembly only) ...
```

### 2. Paired vs Single

`_R1` / `_R2` 패턴이 일관적이지 않으면 single로 처리할 수 있음. 첫 출력에서
**read count가 절반** 으로 보이면 의심.

### 3. Stranded vs Unstranded

`featureCounts -s` 옵션. 모르면 `infer_experiment.py` (RSeQC) 로 추정 시키는 것이
안전합니다.

### 4. Multiple testing

DEG에서 padj 대신 raw pvalue로 필터하면 가짜 양성 폭증. 결과의 컬럼 헤더를
사람이 확인.

### 5. Batch effect 무시

여러 batch가 섞여 있는데 보정 안 하면 클러스터링이 batch로 갈라짐. PCA에서
batch가 PC1 잡으면 위험 신호.

### 6. Mendelian 위반

Trio variant calling에서 de novo 비율이 종종 너무 높게 나옵니다 — 보통 한 명당
1-3개가 정상. 100개 나오면 calling 문제 의심.

---

## "위임할 수 없는 것" 의 가치

이 모든 자동화의 끝에서, **남는 일** 이 사람의 본질적인 일입니다:

1. **가설 만들기** — 어떤 질문을 던질지
2. **데이터의 한계 인식** — 왜 이 결과를 믿으면 안 되는지
3. **생물학적 맥락** — 통계 유의성이 곧 의미가 아니라는 점
4. **다음 실험 디자인** — 컴퓨터가 답할 수 없는 부분을 wet lab으로

코딩 에이전트는 보일러플레이트를 외주화합니다. 그래서 사람은 **위 4가지에 더 많은
시간을 쓸 수 있게 됩니다.**

---

## 다음

→ [Bridge 케이스 스터디](../bridge/) — 위 워크플로우들의 풀 케이스 스터디.
