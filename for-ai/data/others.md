---
title: 기타 Omics
parent: 데이터 레이어
grand_parent: AI → Bio 트랙
nav_order: 6
---

# 기타 Omics — Epigenomics / Metabolomics / Microbiome / Spatial / Imaging
{: .no_toc }

별도 페이지로 깊게 다루기에는 무게가 다르지만, AI 엔지니어가 한 번씩은 마주칠
omics 들. 각 분야의 entry point만 단단히.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 한눈 비교

| 분야 | 측정 대상 | 대표 형식 | AI 진입 난이도 |
|---|---|---|---|
| Epigenomics | DNA 변형 / 접근성 / TF 결합 | BED, bigWig, BAM | 중 (Enformer 등 성숙) |
| Metabolomics | 작은 분자 | mzML, MGF | 상 (도메인 지식 필요) |
| Microbiome | 미생물 군집 | FASTQ → taxonomy table | 중 |
| Spatial transcriptomics | 위치 + 발현 | h5ad + 좌표 | 중 (scRNA 확장) |
| Imaging — cryo-EM | 전자현미경 이미지 | MRC, STAR | 상 (큰 파일, GPU 헤비) |
| Imaging — histology | 광학 / WSI | TIFF, SVS | 중 (CV 친숙) |

---

## 1. Epigenomics

DNA 시퀀스는 같아도 세포마다 "어느 부분이 켜져 있나" 가 다릅니다. 이걸 측정.

### 세 갈래

| 기술 | 무엇을 보나 | 대표 출력 |
|---|---|---|
| **ATAC-seq** | 열린 chromatin (접근 가능 영역) | peak BED |
| **ChIP-seq** | 특정 TF / 히스톤 marker 결합 위치 | peak BED |
| **Bisulfite-seq** (WGBS / RRBS) | DNA methylation (5mC) | per-CpG methylation level |
| **Cut&Run / Cut&Tag** | ChIP의 후계, less input | peak BED |
| **Hi-C / Micro-C** | 3D contact (염색체 접힘) | contact matrix |

### 데이터 형식

```
.bam            정렬된 reads
.bed / .narrowPeak / .broadPeak    peak 좌표
.bigWig / .bw   per-base signal track (genome browser용)
.bedGraph       텍스트 signal
.cool / .mcool  Hi-C contact matrix (HDF5)
```

### Peak calling

```bash
# ATAC-seq / ChIP-seq 표준
macs3 callpeak -t treat.bam -c input.bam -f BAMPE \
               -g hs -n sample --outdir peaks/ \
               -q 0.05 --nomodel --shift -75 --extsize 150
```

`-g hs` = human effective genome size. ATAC는 `--nomodel --shift -75 --extsize 150` 이 표준.

### Methylation

Bisulfite로 처리하면 unmethylated C → U → T. methylated C는 그대로. 따라서 align 결과의
C/T 비율 = methylation level.

| 도구 | 용도 |
|---|---|
| **Bismark** | bisulfite alignment 표준 |
| **methylKit** | DMR (differentially methylated region) |
| **DeepSignal-plant / Nanopolish / Remora** | ONT direct methylation calling |

### AI 모델

| 모델 | 입력 → 출력 | 용도 |
|---|---|---|
| **Enformer** (DeepMind 2021) | 196 kb DNA → 5,313 epigenomic tracks (CAGE/DNase/ChIP) | 변이 → regulatory 효과 |
| **Borzoi** (Calico 2023) | 524 kb DNA → RNA-seq + ATAC + ChIP | Enformer 후계, 더 긴 컨텍스트 |
| **Sei** | 4 kb DNA → 21,907 chromatin profile | sequence class annotation |
| **scBasset** | DNA → single-cell ATAC | scATAC 표준 |
| **ChromBPNet** | DNA → ATAC bias-corrected | TF motif discovery |

```python
# Enformer 추론 (sequence → 5313 tracks)
import tensorflow_hub as hub, tensorflow as tf, numpy as np

enformer = hub.load("https://tfhub.dev/deepmind/enformer/1").model
# 입력: (1, 196608, 4) one-hot DNA, 출력: dict with 'human': (1, 896, 5313)
seq_oh = np.eye(4, dtype=np.float32)[np.random.randint(0, 4, 196608)][None]
preds = enformer.predict_on_batch(tf.constant(seq_oh))
print(preds["human"].shape)   # (1, 896, 5313)
```

### 데이터 받기

- **ENCODE** — 인간/쥐 정제 reference (https://encodeproject.org)
- **Roadmap Epigenomics** — 111 cell type
- **GTRD** — TF binding 통합
- **TCGA methylation** — 암 환자 methylation array
- **4DN** — Hi-C portal

---

## 2. Metabolomics

세포 / 혈청 안의 **작은 분자** (대사체) 측정. 단백질이 아닌 sugar, lipid, amino acid,
nucleotide, drug 등.

### 측정 방식

| 기술 | 특징 |
|---|---|
| **LC-MS** | Liquid chromatography + mass spec. untargeted 표준 |
| **GC-MS** | volatile compound (지방산 등) |
| **NMR** | 비파괴, 정량 정확, 감도 낮음 |
| **CE-MS** | 친수성 대사체 |

### 데이터 형식

[Proteomics](proteomics) 와 동일 mass spec 형식 사용:
- `.mzML` / `.mzXML` — 오픈 표준
- `.raw` (Thermo), `.d` (Bruker) — 벤더 raw
- `.mgf` — peak list

**차이**: proteomics는 peptide (수십~수백 Da, 알려진 building block) → 단백질로 환원.
metabolomics는 **identity 자체가 불확실** (수만 가능 화합물, DB 부족).

### 워크플로우

```
.mzML ─→ [Peak picking] XCMS / MZmine / OpenMS
       ─→ [Alignment / grouping] across samples
       ─→ [Annotation] m/z + RT → 화합물 후보
                       │
            ┌──────────┴──────────┐
       MS¹ matching            MS²/MSⁿ matching
       (m/z + isotope)         (fragmentation tree)
            │                       │
         HMDB / KEGG          GNPS / MoNA / MassBank
                                    │
                              SIRIUS, MetFrag, DiffMS
```

### 도구

| 도구 | 용도 |
|---|---|
| **XCMS** (R) | peak picking 표준 |
| **MZmine 3** | GUI + headless |
| **OpenMS / pyOpenMS** | C++ 코어, Python 바인딩 |
| **GNPS** | molecular networking (UCSD) |
| **MetFrag** | in-silico fragmentation 매칭 |
| **SIRIUS + CSI:FingerID** | formula → fingerprint → structure |
| **DiffMS** (2025) | diffusion model 기반 MS²→구조 |
| **MS2DeepScore** | spectrum embedding |

### AI 모델

| 모델 | 입력 → 출력 |
|---|---|
| **MIST** | MS² → fingerprint |
| **MS2DeepScore** | 두 spectrum → similarity (cosine 대체) |
| **CFM-ID 4** | structure → predicted MS² |
| **DiffMS** | MS² → 분자 graph (de novo) |
| **MassFormer** | MS² → fingerprint, transformer |

### 데이터 받기

- **MetaboLights** (EBI)
- **Metabolomics Workbench** (NIH)
- **GNPS / MassIVE** — public spectra library
- **HMDB** — Human Metabolome DB (~220k 화합물)
- **KEGG** — pathway + 화합물

{: .warning }
> Metabolomics 의 가장 큰 함정은 **"내가 본 m/z 가 정확히 무슨 분자냐"** 가 답이 안 나오는
> 경우가 많다는 것. annotation confidence level (Sumner et al. 2007, 4단계) 을 항상 명시.

---

## 3. Microbiome

샘플 (장 내용물, 토양, 바닷물) 안에 어떤 미생물이 얼마나 있나.

### 두 방식

| 방식 | 측정 | 비용 | 해상도 |
|---|---|---|---|
| **16S rRNA** | 16S 유전자 한 영역만 PCR + 시퀀싱 | 저 | genus 수준 (종 구분 약함) |
| **Shotgun metagenomics** | 모든 DNA를 시퀀싱 | 고 | species / strain + 기능 (KEGG, CAZy) |

### 워크플로우 — 16S

```
FASTQ ─→ [Primer 제거] cutadapt
       ─→ [DADA2 / Deblur] denoise → ASV (Amplicon Sequence Variant)
       ─→ [Taxonomy] SILVA / GTDB 매칭
       ─→ [Diversity] alpha (Shannon) / beta (UniFrac)
```

### 워크플로우 — Shotgun

```
FASTQ ─→ [Host 제거] bowtie2 → 인간 reference
       ─→ [Taxonomy] Kraken2 / MetaPhlAn4 / Centrifuge
       ─→ [Assembly] (선택) megahit / SPAdes → contigs
                                              ├→ [Binning] MetaBAT2 → MAGs
                                              └→ [Function] Prokka, eggNOG
```

### 분류기

| 도구 | 알고리즘 | 메모리 |
|---|---|---|
| **Kraken2** | k-mer + LCA | 큼 (~50 GB for full DB) |
| **Bracken** | Kraken2 결과 + Bayesian 재분배 | 작음 |
| **MetaPhlAn 4** | clade-specific marker gene | 중 |
| **Centrifuge** | BWT 압축 | 작음 |
| **mOTUs3** | universal single-copy marker | 작음 |

### 분석 플랫폼

| 도구 | 형태 |
|---|---|
| **QIIME 2** | end-to-end 16S 표준, Python |
| **mothur** | 클래식, C++ |
| **PICRUSt2** | 16S → 추정 functional profile |
| **HUMAnN 3** | shotgun → pathway abundance |

### AI 모델

이 분야는 sequence model보다 **임베딩 + 다운스트림 ML** 이 흔함.

| 모델 / 접근 | 용도 |
|---|---|
| **DeepMicrobes** | shotgun read → taxonomy (CNN+LSTM) |
| **MetaBERT / DNABERT-S** | metagenome 시퀀스 임베딩 |
| **TaxonProf** / **MAG-GPT** | MAG → taxonomy / function |
| **BiomeBERT** | 임상 microbiome + metadata 연관 |
| **TabPFN / Random Forest on taxa table** | 임상 결과 예측의 실용 표준 |

### 데이터 받기

- **MGnify** (EBI) — 통합 microbiome 분석 결과
- **HMP** — Human Microbiome Project
- **GTDB** — taxonomy reference (whole-genome 기반)
- **SILVA / Greengenes2** — 16S reference
- **curatedMetagenomicData** (R) — 정제된 shotgun 코호트

---

## 4. Spatial Transcriptomics

scRNA-seq + 위치 정보. 어떤 cell type이 어디에 있는지 + 이웃 세포와의 관계.

### 기술 비교

| 플랫폼 | 해상도 | 유전자 수 | 비고 |
|---|---|---|---|
| **10x Visium** | 55 μm spot (~10 세포) | 전체 transcriptome | 가장 흔함, sequencing 기반 |
| **Visium HD** | 2 μm bin | 전체 | 2024~, 진정 단세포 해상도 |
| **10x Xenium** | sub-cellular | 100~5000 패널 | imaging 기반 (in situ) |
| **MERFISH** (Vizgen) | sub-cellular | 100~500 패널 | imaging |
| **Slide-seqV2** | 10 μm bead | 전체 | academic |
| **Stereo-seq** (BGI) | 0.5 μm | 전체 | nanoball, mouse embryo 유명 |
| **GeoMx DSP** (NanoString) | ROI 단위 | 패널 | 임상 친화 |

### 데이터 형식

```python
import scanpy as sc

# Visium - 10x Space Ranger output
adata = sc.read_visium("space_ranger_output/")
# adata.obs = spot 메타 (in_tissue, array_row, array_col)
# adata.obsm["spatial"] = (n_spots, 2) 픽셀 좌표
# adata.uns["spatial"] = H&E 이미지 (lowres, hires)

sc.pl.spatial(adata, color="leiden", img_key="hires")
```

`squidpy` 가 spatial 전용 분석 라이브러리.

### AI 모델

| 모델 | 무엇 |
|---|---|
| **SpatialDE / SpaGCN** | 공간적 가변 유전자 / spatial domain |
| **STAGATE** | graph attention → spatial embedding |
| **Tangram** | scRNA reference → spot 매핑 / deconvolution |
| **cell2location** | Bayesian deconvolution |
| **GraphST**, **SpaceFlow** | graph + time / trajectory |
| **Hist2ST**, **HE2RNA** | H&E 이미지 → 발현 예측 |
| **NicheNet / CellChat** | cell-cell communication (spatial로 확장) |

```python
# STAGATE
import STAGATE
STAGATE.Cal_Spatial_Net(adata, rad_cutoff=150)
adata = STAGATE.train_STAGATE(adata, n_epochs=1000)
# adata.obsm["STAGATE"] 임베딩 사용
```

### 데이터 받기

- **10x Datasets** — 공식 Visium / Xenium 데모
- **STOmicsDB** (BGI Stereo-seq)
- **Spatial Omics DB**, **SODB**
- **HuBMAP / HTAN** — 인간 조직 spatial atlas

---

## 5. Imaging

### 5a. Cryo-EM (Cryo-Electron Microscopy)

분자를 액체질소로 얼린 뒤 전자현미경으로 찍어 3D 구조 재구성. **2017년 노벨상**, X-ray
crystallography 의 후속.

#### 워크플로우

```
Movies (.mrc, .tiff)        # raw, 한 frame당 ~수 MB, 한 dataset 수 TB
   ↓ motion correction
Micrographs
   ↓ CTF estimation         # 광학 보정
   ↓ Particle picking       # CNN: crYOLO, Topaz
Particles (.star, .mrcs)
   ↓ 2D classification
   ↓ 3D reconstruction      # back-projection + refinement
3D volume (.mrc, .map)       # 보통 2-4 Å 분해능
   ↓ Model building
Atomic model (.pdb / .cif)
```

#### 도구

| 도구 | 용도 | 비고 |
|---|---|---|
| **RELION 5** | end-to-end pipeline | open source, GPU |
| **cryoSPARC** | end-to-end + GUI | 상용 (academic free) |
| **CTFFIND4 / Gctf** | CTF estimation | |
| **MotionCor2** | motion correction | |
| **Topaz** | DL particle picker | |
| **crYOLO** | DL particle picker (YOLO 기반) | |
| **ModelAngelo** (DeepMind 2023) | density map → atomic model 자동 | AlphaFold 의 cryo-EM 판 |
| **CryoDRGN** | heterogeneous reconstruction, VAE | conformational landscape |

#### AI 모델

- **ModelAngelo** — density map + sequence → 완성된 atomic model. 수동 model building 1주일 → 수 시간.
- **CryoDRGN / CryoDRGN-AI** — 입자별 conformation latent space 학습.
- **Topaz / crYOLO** — particle picking. 라벨 수십~수백 입자로도 fine-tune.

#### 데이터 받기

- **EMPIAR** — raw cryo-EM dataset (~수 TB / 단위)
- **EMDB** — density map (.map)
- **PDB** — cryo-EM 유래 atomic model (전체 PDB의 30%+)

### 5b. Histology / Optical Microscopy

조직 슬라이드 (H&E, IHC, IF). AI 엔지니어에게 가장 친숙 — **그냥 이미지 분류 / segmentation**.

#### 파일 포맷

| 포맷 | 무엇 |
|---|---|
| **TIFF / OME-TIFF** | 표준, multi-channel / multi-z |
| **SVS** (Aperio), **NDPI** (Hamamatsu), **CZI** (Zeiss) | 벤더별 WSI (Whole Slide Image) |
| **ZARR / OME-ZARR** | 클라우드 친화 chunk |

WSI는 한 슬라이드 ~수 GB, **pyramid** 구조 (여러 해상도). 직접 PIL로 열면 안 됨.

```python
import openslide
slide = openslide.OpenSlide("sample.svs")
print(slide.level_count, slide.level_dimensions)
# tile 단위로 읽기
tile = slide.read_region((10000, 10000), level=0, size=(512, 512))
```

#### 도구

| 도구 | 용도 |
|---|---|
| **CellProfiler** | 클래식 cell segmentation pipeline, GUI |
| **QuPath** | WSI 분석 표준, Groovy 스크립트 |
| **Fiji / ImageJ** | 만능 이미지 분석 |
| **Cellpose 3** | DL cell / nucleus segmentation, generalist |
| **StarDist** | 별 모양 nucleus segmentation |
| **HoVer-Net / Mesmer** | 핵 + 세포 동시 |
| **MONAI** | 의료영상 DL 프레임워크 (PyTorch) |

#### AI 모델 — Foundation Models

| 모델 | 무엇 |
|---|---|
| **UNI** (Mahmood lab, 2024) | 100k WSI pretrain, histology embedding |
| **CONCH** | vision-language histology |
| **GigaPath** (Microsoft 2024) | 1.3B param slide-level encoder |
| **Virchow / Virchow2** (Paige 2024) | 1.5M WSI, pathology foundation |
| **Prov-GigaPath** | longitudinal patient |
| **PLIP** | CLIP for pathology |

```python
# Cellpose 3 — 한 줄 segmentation
from cellpose import models
model = models.Cellpose(model_type="cyto3", gpu=True)
masks, flows, styles, diams = model.eval(img, channels=[0, 0], diameter=None)
```

#### 데이터 받기

- **TCGA Diagnostic Slides** — 암 조직 ~30k WSI
- **PAIP**, **CAMELYON16/17** — 챌린지 데이터
- **TCIA** (The Cancer Imaging Archive) — radiology + pathology
- **PathML / HuggingFace pathology datasets**

---

## 분야별 빠른 entry point 요약

| 시작하려면 | 첫 도구 | 첫 데이터셋 |
|---|---|---|
| Epigenomics (peak) | MACS3 + deepTools | ENCODE GM12878 ATAC |
| Epigenomics (model) | Enformer (TF Hub) | Basenji2 / ENCODE |
| Metabolomics | XCMS + GNPS | MetaboLights MTBLS1 |
| Microbiome 16S | QIIME 2 tutorial | "Moving Pictures" |
| Microbiome shotgun | MetaPhlAn 4 + Kraken2 | HMP2 IBD |
| Spatial | scanpy + squidpy | 10x Visium Mouse Brain demo |
| Cryo-EM | RELION tutorial | EMPIAR-10025 (TRPV1) |
| Histology | Cellpose + QuPath | CAMELYON16 |

---

## 자주 마주치는 함정 (전체)

### 1. **Genome build 불일치** (epigenomics)
ENCODE 옛 데이터는 hg19, 새 것은 hg38. `liftOver` 또는 처음부터 같은 build로 통일.

### 2. **Adduct 무지** (metabolomics)
같은 분자가 `[M+H]+`, `[M+Na]+`, `[M-H2O+H]+` 등 다른 m/z로 보일 수 있음. CAMERA, MS-DIAL
의 adduct deconvolution 필수.

### 3. **Compositional data** (microbiome)
abundance 는 "비율" — 한 taxon이 늘면 다른 게 자동으로 줄어 보임. CLR (centered log-ratio)
변환 후 일반 통계 사용. `ANCOM-BC`, `ALDEx2`.

### 4. **Spot ≠ cell** (Visium)
Visium spot은 ~10 세포 평균. 단세포 결론 내리려면 deconvolution (cell2location, Tangram)
또는 Visium HD / Xenium 사용.

### 5. **Cryo-EM 분해능 정의**
지역마다 다름. `Local resolution map` 항상 확인. 평균 2.5 Å 이라도 active site는 4 Å 일 수 있음.

### 6. **Stain variability** (histology)
같은 조직도 병원/스캐너/날짜마다 색이 다름. `Macenko`, `Vahadane`, `StainNet` 으로 정규화.

### 7. **Multi-scale storage** (WSI)
512×512 tile 학습이 표준. 슬라이드 전체를 메모리에 올리지 마세요.

---

## 다음

- 작업: [Expression analysis](../tasks/expression-analysis), [Sequence analysis](../tasks/sequence-analysis)
- 관련 데이터: [Single-cell](single-cell) (spatial 의 자매), [Proteomics](proteomics) (mass spec 공유)
