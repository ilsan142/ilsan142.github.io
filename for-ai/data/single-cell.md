---
title: Single-cell
parent: 데이터 레이어
grand_parent: AI → Bio 트랙
nav_order: 3
---

# Single-cell RNA-seq (scRNA-seq)
{: .no_toc }

AI 엔지니어가 가장 빨리 진입할 수 있는 분야. 데이터가 행렬이고, foundation model이
이미 표준화되어 있고, 벤치마크가 있습니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 핵심 개념

**Single-cell RNA-seq** = 한 번에 수천~수십만 세포 각각의 RNA 발현을 측정.

| Bulk RNA-seq | Single-cell |
|---|---|
| sample 단위 평균 | 세포 단위 분포 |
| 노이즈 적음 | 노이즈·dropout 많음 |
| 통계 표준화됨 | foundation model 분야 |

기술: **10x Genomics Chromium** 이 시장의 80%. 대안: Drop-seq, SMART-seq, Parse Bio.

---

## 데이터 형식

### Raw — 10x Cell Ranger output

```
filtered_feature_bc_matrix/
├── barcodes.tsv.gz       # 세포 ID
├── features.tsv.gz       # 유전자 ID + name
└── matrix.mtx.gz         # sparse count matrix
```

또는 `filtered_feature_bc_matrix.h5` (HDF5 단일 파일).

### 분석용 — AnnData (`.h5ad`)

Python `anndata` 표준 포맷. **이게 사실상 표준** 입니다.

```python
import anndata as ad
adata = ad.read_h5ad("pbmc.h5ad")
# adata.X         : sparse count matrix (cells × genes)
# adata.obs       : 세포 메타데이터 (cell type, sample, ...)
# adata.var       : 유전자 메타데이터
# adata.obsm      : 임베딩 (PCA, UMAP, ...)
# adata.layers    : raw, normalized, ... 여러 표현
```

R 진영은 **`SingleCellExperiment` (Bioconductor)** 또는 **`Seurat`** 객체.

---

## 표준 워크플로우 (scanpy)

AI 엔지니어가 가장 익숙해야 할 순서:

```python
import scanpy as sc

# 1. 로드
adata = sc.read_10x_h5("filtered_feature_bc_matrix.h5")

# 2. QC
sc.pp.calculate_qc_metrics(adata, percent_top=None, log1p=False, inplace=True)
adata.var["mt"] = adata.var_names.str.startswith("MT-")
sc.pp.calculate_qc_metrics(adata, qc_vars=["mt"], percent_top=None,
                            log1p=False, inplace=True)

# 필터: 너무 적게 발현된 세포, 미토콘드리아 비율 높은 세포 제거
adata = adata[adata.obs.n_genes_by_counts > 200, :]
adata = adata[adata.obs.pct_counts_mt < 10, :]

# 3. Doublet 제거 (선택)
import scrublet as scr
scrub = scr.Scrublet(adata.X)
doublet_scores, predicted_doublets = scrub.scrub_doublets()
adata = adata[~predicted_doublets, :]

# 4. Normalize → log1p → HVG
sc.pp.normalize_total(adata, target_sum=1e4)
sc.pp.log1p(adata)
sc.pp.highly_variable_genes(adata, n_top_genes=2000)
adata = adata[:, adata.var.highly_variable]

# 5. Scale → PCA
sc.pp.scale(adata, max_value=10)
sc.tl.pca(adata, n_comps=50)

# 6. (선택) Batch correction
import scanpy.external as sce
sce.pp.harmony_integrate(adata, key="batch")

# 7. Neighbors → UMAP → cluster
sc.pp.neighbors(adata, n_neighbors=15, n_pcs=40, use_rep="X_pca_harmony")
sc.tl.umap(adata)
sc.tl.leiden(adata, resolution=0.5)

# 8. Marker gene
sc.tl.rank_genes_groups(adata, "leiden", method="wilcoxon")
sc.pl.rank_genes_groups(adata, n_genes=20)
```

---

## Foundation Models — 이 분야의 핵심

scRNA-seq는 **생물학 분야 중 foundation model이 가장 활발한 곳**.

### scGPT (Cui et al., 2023)
- 33M 세포 사전학습 Transformer
- 다재다능: classification / batch integration / perturbation 예측
- [GitHub](https://github.com/bowang-lab/scGPT)

```python
from scgpt.tasks import GeneEmbedding
emb = GeneEmbedding.from_pretrained("scGPT_human")
gene_emb = emb.get_embeddings(["CD3D", "CD8A", "MS4A1"])
```

### Geneformer (Theodoris et al., Nature 2023)
- BERT 형. 30M 세포 사전학습
- 입력 = **gene rank** (발현 순위) — 매우 영리한 토큰화
- in silico perturbation (특정 유전자 mask → 다른 유전자 변화 예측) 가능

### UCE (Universal Cell Embedding, Rosen et al., 2024)
- **종 간 비교** 가능 (사람-쥐-제브라피쉬)
- 36M 세포, 8개 종

### scFoundation, CellPLM, CELLama
- 차세대 후보들. 벤치마크 아직 흔들림.

### "정말 도움되나" 논쟁

[Kedzierska et al. 2023](https://www.biorxiv.org/content/10.1101/2023.10.16.561085) 에서
"zero-shot foundation model이 단순 PCA보다 항상 낫지는 않다" 보고. 분야 합의는 진행 중.

**실용적 결론**:
- Cell type annotation, perturbation 예측 → foundation model 시도해 볼 가치
- 단순 클러스터링 / DE → 전통 (scanpy) 이 여전히 견고

---

## Cell Type Annotation

### Marker-based (전통)

PBMC 예:

| Cluster marker | Cell type |
|---|---|
| CD3D, CD3E, CD4 | CD4+ T |
| CD3D, CD3E, CD8A, CD8B | CD8+ T |
| MS4A1 (=CD20), CD79A | B |
| GNLY, NKG7 | NK |
| CD14, LYZ | Monocyte |
| FCGR3A | CD16+ Mono |
| CST3, FCER1A | DC |
| PPBP | Platelet |

### Reference-based

- **Symphony**, **scArches**, **CellTypist** — 큰 reference atlas 와 매핑
- **Geneformer / scGPT zero-shot** — pretrained 모델로 직접 분류

### 자동화 vs 수동

자동 도구는 **상위 cell type**(T cell vs B cell)에서는 잘 맞고, **sub-type**
(CD4 naive vs CD4 effector memory) 에서는 사람 검증 필요.

---

## 자주 마주치는 함정

### 1. **Doublet**
두 세포가 한 droplet에 들어간 인공물. `scrublet`, `DoubletFinder`, `scDblFinder` 로 제거.

### 2. **Ambient RNA**
세포 외부 RNA가 droplet에 섞임. `SoupX`, `decontX` 로 보정.

### 3. **Batch effect**
다른 날, 다른 lane, 다른 환자 = 모두 batch. Harmony / scVI / scANVI 로 통합.

### 4. **Sparse**
10x 데이터의 매트릭스는 90%+ zero. dense로 변환하면 OOM. 항상 `scipy.sparse` 유지.

### 5. **Cell cycle effect**
증식 세포는 cell cycle gene이 강해서 따로 묶일 수 있음. `sc.tl.score_genes_cell_cycle`
로 점수 매기고 회귀로 보정.

### 6. **Cluster 수 = resolution 임의 선택**
`leiden(resolution=...)`. 정답이 없습니다. 여러 값으로 안정성 본 후 결정.

---

## 데이터 받을 수 있는 곳

- **CellxGene** (CZI) — 1500만+ 세포, 표준화된 h5ad. **AI 엔지니어 입문 최강**.
- **Human Cell Atlas** — 인간 모든 조직
- **Tabula Sapiens / Tabula Muris** — 전신 atlas
- **scAtlas** / **PanglaoDB** — 정제된 데이터셋 모음
- **GEO** — 개별 연구 raw 데이터

---

## 다음

- 작업: [Expression analysis](../tasks/expression-analysis)
- 관련 데이터: [Transcriptomics (bulk)](transcriptomics), [기타: Spatial](others#spatial-transcriptomics)
