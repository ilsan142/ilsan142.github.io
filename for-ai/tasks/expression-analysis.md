---
title: Expression analysis
parent: 작업 레이어
grand_parent: AI → Bio 트랙
nav_order: 3
---

# Expression analysis — 발현 분석
{: .no_toc }

Count matrix에서 "어떤 유전자가 어디서 더 많이 발현되나" 까지. Bulk DEG와
single-cell 클러스터링/annotation을 한 페이지에 모았습니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 큰 그림

발현 분석은 두 갈래로 갈립니다.

```
Count matrix
├─ Bulk RNA-seq (sample × gene)
│   └─ DEG: DESeq2 / edgeR / limma-voom
│       └─ Enrichment: GSEA / clusterProfiler / decoupleR
│
└─ Single-cell (cell × gene, sparse)
    └─ QC → normalize → HVG → PCA → neighbors → UMAP → leiden
        ├─ Marker gene → cell type annotation
        └─ Foundation model: scGPT / Geneformer / UCE
```

데이터 표현은 [Transcriptomics](../data/transcriptomics) ·
[Single-cell](../data/single-cell) 참고. 이 페이지는 **분석 단계** 에 집중.

---

## Part 1 — Bulk DEG

### 통계 모델 한 줄 요약

RNA-seq count는 **negative binomial (NB) 분포** 를 따릅니다. Poisson은
"variance = mean" 가정이지만 실제 RNA-seq는 그보다 분산이 큽니다 (overdispersion).
세 도구 모두 NB GLM 변형.

| 도구 | 코어 | shrinkage | 추천 상황 |
|---|---|---|---|
| **DESeq2** | NB GLM + Wald | ✓ (LFC shrink) | 첫 선택, 표본 적을 때 강함 |
| **edgeR** | NB GLM + QL F-test | ✓ (dispersion) | DESeq2와 거의 동급 |
| **limma-voom** | log-CPM + weight + linear | mean-variance trend | 표본 많을 때, 복잡한 design |

{: .tip }
> 어떤 걸 쓸까 고민 시간에 그냥 **DESeq2 부터** 돌리고, 의심되면 다른 도구로
> sanity check. 세 도구 결과는 보통 90%+ 겹칩니다.

### 표준 파이프라인 (의사결정 트리)

```
Count matrix (genes × samples, 정수)
   ↓
[1] Pre-filter: rowSums(counts) >= 10 인 유전자만 남기기
   ↓
[2] 디자인 행렬 작성:  ~ batch + condition
   ↓
[3] DESeq() 또는 estimateDisp() → glmQLFit() 또는 voom()
   ↓
[4] Contrast 지정: c("condition", "trt", "ctrl")
   ↓
[5] padj < 0.05 & |log2FC| > 1 필터
   ↓
[6] GSEA / over-representation
```

### DESeq2 — R 코드

```r
library(DESeq2)

# 1. 입력
cts <- read.table("counts.txt", row.names=1, header=TRUE, check.names=FALSE)
coldata <- read.csv("metadata.csv", row.names=1)
coldata$condition <- factor(coldata$condition, levels=c("ctrl","trt"))  # 순서 중요
coldata$batch     <- factor(coldata$batch)

# 2. 객체 생성 (design에 batch 포함)
dds <- DESeqDataSetFromMatrix(
  countData = cts,
  colData   = coldata,
  design    = ~ batch + condition
)

# 3. Pre-filter
keep <- rowSums(counts(dds) >= 10) >= 3   # 최소 3 샘플에서 count>=10
dds  <- dds[keep, ]

# 4. 추정
dds <- DESeq(dds)

# 5. Contrast (마지막 변수의 두 레벨 비교)
res <- results(dds, contrast=c("condition","trt","ctrl"), alpha=0.05)

# 6. LFC shrinkage (시각화/랭킹용)
res_shrunk <- lfcShrink(dds, coef="condition_trt_vs_ctrl", type="apeglm")

# 7. DEG 추출
deg <- as.data.frame(res_shrunk) |>
  subset(padj < 0.05 & abs(log2FoldChange) > 1)
write.csv(deg, "DEG.csv")
```

### PyDESeq2 — 같은 결과를 파이썬에서

```python
import pandas as pd
from pydeseq2.dds import DeseqDataSet
from pydeseq2.ds  import DeseqStats

counts   = pd.read_csv("counts.txt",   sep="\t", index_col=0)  # genes × samples
metadata = pd.read_csv("metadata.csv",          index_col=0)   # samples

# DeseqDataSet은 cells × features 형식을 받으므로 전치
dds = DeseqDataSet(
    counts=counts.T,
    metadata=metadata,
    design_factors=["batch", "condition"],
    refit_cooks=True,
)
dds.deseq2()

stats = DeseqStats(dds, contrast=["condition", "trt", "ctrl"])
stats.summary()
res = stats.results_df
deg = res.query("padj < 0.05 and abs(log2FoldChange) > 1")
```

{: .note }
> `pydeseq2` 는 R DESeq2의 충실한 포팅. 결과가 R 버전과 거의 동일 (소수점 오차).
> 단, **shrinkage 종류 (apeglm/ashr)** 는 일부 지원 안 됨.

### 시각화 표준 셋

| 그림 | 무엇을 보여주나 | R | Python |
|---|---|---|---|
| MA plot | log2FC vs mean expression | `plotMA(res)` | `seaborn` 직접 |
| Volcano | log2FC vs -log10(padj) | `EnhancedVolcano` | `bioinfokit` / `seaborn` |
| Heatmap (top DEG) | 샘플별 정규화 발현 | `pheatmap` | `seaborn.clustermap` |
| PCA | 샘플 군집 sanity check | `plotPCA(vsd)` | `sklearn.decomposition.PCA` |

```r
library(EnhancedVolcano)
EnhancedVolcano(res_shrunk,
  lab = rownames(res_shrunk),
  x = "log2FoldChange", y = "padj",
  pCutoff = 0.05, FCcutoff = 1)
```

---

## Part 2 — Pathway / Enrichment

DEG 리스트 → "어떤 경로가 영향을 받았나".

### 두 가지 접근

| 접근 | 입력 | 도구 |
|---|---|---|
| **Over-representation (ORA)** | DEG 리스트 (cut-off 후) | `clusterProfiler::enrichGO/KEGG`, `enrichr` |
| **GSEA (rank-based)** | 모든 유전자의 ranked score | `clusterProfiler::gseGO`, `fgsea` |
| **Activity / footprint** | 발현 matrix | `decoupleR`, `PROGENy`, `DoRothEA` |

### GSEA 코드

```r
library(clusterProfiler)
library(org.Hs.eg.db)

# stat 기반 ranking (DESeq2 res)
gene_list <- res$stat
names(gene_list) <- rownames(res)
gene_list <- sort(gene_list[!is.na(gene_list)], decreasing=TRUE)

gsea <- gseGO(geneList=gene_list, OrgDb=org.Hs.eg.db,
              ont="BP", keyType="ENSEMBL", minGSSize=10)
dotplot(gsea, showCategory=20)
```

```python
# fgsea의 파이썬 친구: gseapy
import gseapy as gp
pre = gp.prerank(rnk=ranked_df, gene_sets="GO_Biological_Process_2023",
                 outdir=None, permutation_num=1000)
```

### decoupleR — TF / pathway activity

```python
import decoupler as dc
net = dc.get_collectri(organism="human")
acts = dc.run_ulm(mat=expr_df.T, net=net, source="source", target="target",
                  weight="weight", verbose=True)
```

DEG 보다 **upstream regulator** (TF, kinase) 활성을 보는 게 해석에 종종 더 유용.

---

## Part 3 — Single-cell 표준 흐름

### 의사결정 트리

```
.h5 / .h5ad / mtx 로드
   ↓
[QC]  n_genes, total_counts, pct_mt 분포 → 컷오프 정함
   ↓
[Doublet]  scrublet / scDblFinder
   ↓
[Ambient RNA]  SoupX / decontX (선택)
   ↓
[Normalize]  CP10k → log1p   (또는 sctransform)
   ↓
[HVG]  상위 2000~3000 유전자
   ↓
[Scale]  per-gene z-score (선택, scanpy는 PCA 전에 함)
   ↓
[PCA]  보통 30~50 PC
   ↓
[Batch integration]  Harmony / scVI / scANVI   (batch 있을 때만)
   ↓
[Neighbors → UMAP]
   ↓
[Cluster]  Leiden(resolution=0.5~1.5)
   ↓
[Marker]  rank_genes_groups (wilcoxon)
   ↓
[Annotate]  marker / reference / foundation model
```

### scanpy 풀 파이프라인

```python
import scanpy as sc
import scrublet as scr

sc.settings.set_figure_params(dpi=80, frameon=False)

# 1. Load
adata = sc.read_10x_h5("filtered_feature_bc_matrix.h5")
adata.var_names_make_unique()

# 2. QC
adata.var["mt"]   = adata.var_names.str.startswith("MT-")
adata.var["ribo"] = adata.var_names.str.startswith(("RPS","RPL"))
sc.pp.calculate_qc_metrics(adata, qc_vars=["mt","ribo"],
                           percent_top=None, log1p=False, inplace=True)

adata = adata[(adata.obs.n_genes_by_counts > 200) &
              (adata.obs.n_genes_by_counts < 6000) &
              (adata.obs.pct_counts_mt < 10), :].copy()

# 3. Doublet
scrub = scr.Scrublet(adata.X)
doublet_scores, predicted = scrub.scrub_doublets()
adata = adata[~predicted, :].copy()

# 4. Normalize → log1p
adata.layers["counts"] = adata.X.copy()   # raw 보존
sc.pp.normalize_total(adata, target_sum=1e4)
sc.pp.log1p(adata)

# 5. HVG
sc.pp.highly_variable_genes(adata, n_top_genes=2000, flavor="seurat_v3",
                            layer="counts", batch_key="sample")
adata.raw = adata
adata = adata[:, adata.var.highly_variable].copy()

# 6. Scale + PCA
sc.pp.scale(adata, max_value=10)
sc.tl.pca(adata, n_comps=50)

# 7. Batch correction (선택)
import scanpy.external as sce
sce.pp.harmony_integrate(adata, key="sample")

# 8. Neighbors → UMAP → Leiden
sc.pp.neighbors(adata, n_neighbors=15, n_pcs=40, use_rep="X_pca_harmony")
sc.tl.umap(adata)
sc.tl.leiden(adata, resolution=0.8, flavor="igraph", n_iterations=2)

# 9. Marker
sc.tl.rank_genes_groups(adata, "leiden", method="wilcoxon")
sc.pl.rank_genes_groups_dotplot(adata, n_genes=5)
```

### Seurat 등가 (R)

```r
library(Seurat)
obj <- CreateSeuratObject(counts=Read10X_h5("matrix.h5"), min.cells=3, min.features=200)
obj[["percent.mt"]] <- PercentageFeatureSet(obj, pattern="^MT-")
obj <- subset(obj, nFeature_RNA > 200 & nFeature_RNA < 6000 & percent.mt < 10)

obj <- SCTransform(obj, vars.to.regress="percent.mt")
obj <- RunPCA(obj) |> FindNeighbors(dims=1:30) |>
       FindClusters(resolution=0.8) |> RunUMAP(dims=1:30)

markers <- FindAllMarkers(obj, only.pos=TRUE, min.pct=0.25, logfc.threshold=0.25)
```

---

## Part 4 — Foundation Model 활용

### 언제 foundation model로 가야 하나

| 상황 | 전통 | Foundation model |
|---|---|---|
| PBMC 같은 잘 알려진 조직 cell type annotation | marker gene으로 충분 | 굳이 |
| Rare / novel cell state | 어려움 | **scGPT, Geneformer** zero-shot |
| 종 간 cell type 매핑 (사람↔쥐) | ortholog 매핑 번거로움 | **UCE** |
| Perturbation 결과 예측 (KO 효과) | 거의 불가능 | **Geneformer, scGPT** |
| Batch correction | Harmony / scVI 우수 | scGPT 임베딩도 가능 |

### scGPT — cell type annotation (fine-tune)

```python
from scgpt.tasks import CellTypeAnnotation
from scgpt.model import load_pretrained

model = load_pretrained("scGPT_human")
task  = CellTypeAnnotation(model)

task.fit(train_adata, label_key="celltype", epochs=10)
pred = task.predict(query_adata)
query_adata.obs["scgpt_pred"] = pred
```

### Geneformer — in silico perturbation

```python
from geneformer import InSilicoPerturber

isp = InSilicoPerturber(
    perturb_type="delete",       # gene KO 시뮬레이션
    genes_to_perturb=["TP53"],
    model_directory="Geneformer-V2-104M",
)
results = isp.perturb(input_dataset="my_tokenized.dataset",
                     output_directory="isp_out/")
```

{: .warning }
> Foundation model 결과는 **반드시 실험 검증** 또는 hold-out 검증. 특히
> in silico perturbation은 학습 분포에 강하게 의존.

### "정말 도움되나" 현황 (2026-05)

[Kedzierska et al. 2023](https://doi.org/10.1101/2023.10.16.561085),
[Boiarsky et al. 2024](https://doi.org/10.1101/2024.04.04.587987) 등에서
**zero-shot scGPT/Geneformer가 단순 PCA + KNN을 못 이기는 경우** 가 보고됐습니다.
2026년 기준 합의:

- **Fine-tune** 했을 때는 대체로 도움.
- **Zero-shot** 은 도메인 (조직, 종, 기술) 이 학습 데이터와 비슷할 때만 신뢰.
- **Sanity check** 로 PCA + Harmony + KNN baseline 도 항상 같이 돌리기.

---

## Cell Type Annotation — 실전 워크플로우

```
Leiden cluster
   ↓
[A] Marker-based     → CellMarker DB, PanglaoDB로 매핑
[B] Reference-based  → CellTypist, Symphony, scArches
[C] Foundation model → scGPT / Geneformer zero-shot
   ↓
교차 검증 (세 방법이 비슷한 답을 주는가?)
   ↓
이상한 cluster (어디에도 안 맞음) → marker gene 수동 검토
   ↓
"unknown" 도 valid한 답. 강제로 분류하지 말기.
```

### CellTypist — 가장 빠른 시작

```python
import celltypist
model = celltypist.models.Model.load(model="Immune_All_Low.pkl")
pred  = celltypist.annotate(adata, model=model, majority_voting=True)
adata = pred.to_adata()
```

12개 이상의 사전학습 모델 (immune, lung, gut, ...) 이 있어서 PBMC라면 바로 됨.

---

## 자주 마주치는 함정

### Bulk

#### 1. Batch effect를 design에서 빠뜨림
`~ condition` 만 쓰면 batch 효과가 condition으로 흡수됨. **항상** `~ batch + condition`.
Batch와 condition이 완전히 confound 됐다면 (모든 ctrl이 batch1, trt가 batch2) 분석 불가.

#### 2. Multiple testing 무시
`p < 0.05` 가 아니라 `padj < 0.05`. DESeq2 / edgeR 은 BH-FDR 자동.

#### 3. Low-count gene
1-2 read짜리 유전자는 통계가 부정확. **pre-filter** (`rowSums(counts) >= 10`)
또는 DESeq2의 자동 independent filtering 신뢰.

#### 4. Log fold change 만 보고 흥분
log2FC = 5 인데 padj = 0.5 → 노이즈. **padj와 LFC 둘 다** 봐야 함.

#### 5. Contrast 방향 헷갈림
`contrast=c("condition","trt","ctrl")` → log2FC > 0 = trt에서 높음.
반대로 적으면 부호 반대. **factor level 순서를 명시적으로** 지정 (`factor(.., levels=...)`).

### Single-cell

#### 6. Doublet 제거 빼먹음
두 세포가 한 droplet에 들어간 인공물. **항상 scrublet/scDblFinder 돌리기**.

#### 7. Ambient RNA
세포 외 RNA가 droplet에 섞임. SoupX / decontX로 보정 (선택, 데이터 따라).

#### 8. Sparse → dense 실수
`adata.X.toarray()` 한 번 호출이 100GB RAM 잡아먹을 수 있음.
`scipy.sparse` 형태 유지.

#### 9. Resolution 임의 선택
`leiden(resolution=0.5)` vs `1.5` 결과가 매우 다름. 여러 resolution을 비교하고
`clustree` 로 안정성 시각화.

#### 10. Cell cycle confound
증식 세포가 cell cycle gene 때문에 따로 묶임. `sc.tl.score_genes_cell_cycle`
로 점수 매긴 뒤 회귀로 보정.

#### 11. Marker가 cluster 사이 공유됨
`rank_genes_groups` 의 top gene이 여러 cluster에서 동시에 marker로 잡힘.
`sc.tl.dendrogram` 으로 cluster 관계 보고, 너무 비슷하면 merge 고려.

---

## "Pseudobulk" — single-cell ↔ bulk 다리

scRNA-seq 데이터로 DEG를 할 때 **세포 단위 t-test는 false positive가 많습니다**.
권장 패턴은 **샘플 × cell type 별로 합쳐서 bulk처럼 DESeq2**:

```python
import decoupler as dc

# adata.obs에 sample, condition, celltype 컬럼이 있다고 가정
pdata = dc.get_pseudobulk(adata, sample_col="sample",
                          groups_col="celltype", layer="counts",
                          min_cells=10, min_counts=1000)

# 이제 pdata는 sample×celltype 단위 bulk count matrix.
# 각 cell type별로 DESeq2 돌리기.
```

`muon / pertpy` 도 같은 패턴 지원.

---

## 다음

- 데이터: [Transcriptomics (bulk)](../data/transcriptomics) · [Single-cell](../data/single-cell)
- 작업: [Variant calling](variant-calling) · [Structure prediction](structure-prediction)
- 통합: [Mental model](../mental-model)
