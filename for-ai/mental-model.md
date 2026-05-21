---
title: 멘탈 모델
parent: AI → Bio 트랙
nav_order: 1
---

# 멘탈 모델 — DNA에서 표현형까지
{: .no_toc }

이 페이지를 건너뛰면 나머지가 안 읽힙니다. **30분만 투자하세요.**
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Central Dogma — 한 장 요약

```
        [복제]
          ↻
        ┌───┐
        │ DNA │ ── 전사 ─→  ┌───┐
        └───┘   transcribe  │ RNA │── 번역 ─→ ┌──────┐
                            └───┘  translate │ Protein │── 폴딩 ─→ 3D structure
                                              └──────┘                 ↓
                                                                   세포 기능
                                                                       ↓
                                                                  표현형 (Phenotype)
```

### AI 엔지니어 관점의 매핑

| 생물학 layer | "토큰" 단위 | 길이 스케일 | 대표 데이터 형식 |
|---|---|---|---|
| DNA | A/T/C/G 4종 | 3 Gb (인간 한 게놈) | FASTA, FASTQ, BAM, VCF |
| RNA | A/U/C/G 4종 | ~kb (mRNA), Mb (long ncRNA) | FASTQ, count matrix |
| Protein | 20종 아미노산 | 수십~수천 aa | FASTA, PDB |
| 세포 | gene × cell 행렬 | ~20k 유전자 × N cells | h5ad (anndata) |

**핵심 인사이트**: 생물학 데이터는 본질적으로 **시퀀스(seq)** 거나 **행렬(matrix)** 입니다.
둘 다 Transformer 가 잘 하는 일.

---

## 인간 유전체의 스케일

AI 엔지니어가 자주 놓치는 직관:

```
인간 유전체 = 3,000,000,000 base pairs
            = 약 750 MB (4 bp/byte 압축 시)
            = 약 25,000 유전자 (전체의 ~2%만 단백질 코딩)
            = 약 100,000 transcripts (alternative splicing 포함)
            = 약 20,000 단백질
```

LLM 토큰 환산:
- 인간 유전체 = ~3B tokens (염기 단위)
- 인간 proteome 전체 = ~10M tokens (아미노산 단위)
- 단일 단백질 평균 = ~400 tokens
- 단일 mRNA = ~2,000 tokens

이게 왜 중요한가: **컨텍스트 길이가 곧 모델의 능력 한계** 입니다.
DNA 모델이 1M bp 컨텍스트를 갖는다는 게 (Evo-2, HyenaDNA) 왜 큰 일인지 이해가 됩니다.

---

## DNA — 더 자세히

### 분자 구조

- **이중나선** (double helix). A↔T, C↔G 상보적 결합
- **방향성** 있음: 5' → 3' (시퀀스는 항상 이 방향으로 표기)
- **유전체 (genome)** = 모든 염색체의 DNA 합

### 유전자 (gene) 구조

```
5' ─[ promoter ]─[ 5'UTR ]─[ exon1 ]─[ intron1 ]─[ exon2 ]─[ intron2 ]─[ exon3 ]─[ 3'UTR ]─[ polyA ]── 3'
                                  └────── transcript (RNA) ─────────┘
                                        └──── 번역되는 부분 (CDS) ────┘
                                                  ↓
                                              Protein
```

**Splicing**: 전사된 pre-mRNA에서 intron이 제거되고 exon만 이어 붙음.
**Alternative splicing**: 같은 유전자에서 exon 조합이 달라져 여러 단백질이 나옴.

### 변이 (Variant)

| 종류 | 예 | 영향 |
|---|---|---|
| SNP (Single Nucleotide Polymorphism) | A→G | 단일 염기 변화 |
| Indel | AT→ATGG | 짧은 삽입/결실 |
| CNV (Copy Number Variant) | 유전자 1→3카피 | 큰 영역 복제/결실 |
| SV (Structural Variant) | 염색체 일부 inversion | 매우 큰 재배열 |

**Variant calling** = BAM (정렬된 reads) → VCF (변이 목록). 이게 [genomics 작업](tasks/variant-calling) 의 핵심.

---

## RNA — DNA의 작업 사본

DNA가 "원본" 이라면 RNA는 **그 세포가 지금 무엇을 발현하고 있는지** 의 스냅샷.

### 종류

| 종류 | 길이 | 역할 |
|---|---|---|
| **mRNA** | 수백~수천 nt | 단백질 코딩 |
| **rRNA** | varies | 리보솜 구성 (전체 RNA의 ~80%!) |
| **tRNA** | ~80 nt | 아미노산 운반 |
| **miRNA** | ~22 nt | 발현 조절 |
| **lncRNA** | >200 nt | 조절·구조 |
| **circRNA** | varies | 원형 RNA, 안정성 |

### RNA-seq

세포에서 RNA를 추출 → 시퀀싱 → 어느 유전자가 얼마나 발현됐는지 정량.

```
샘플 N개의 RNA → 시퀀싱 → reads (FASTQ) → 정렬 (STAR) → 유전자별 count → 행렬
                                                                    ↓
                                                         gene × sample matrix
                                                                    ↓
                                                          [Differential Expression]
                                                          control vs treatment
                                                                    ↓
                                                              DEG list
```

AI 엔지니어에게 친숙한 말로:
**RNA-seq count matrix = (gene × sample) 매트릭스, 각 cell이 count.**
DESeq2/edgeR이 negative binomial GLM으로 분석.

### Single-cell RNA-seq (scRNA-seq)

이번엔 sample 단위가 아니라 **세포 단위**:

```
gene × cell matrix  (보통 20,000 genes × 10,000 cells = sparse matrix)
```

- 매우 sparse (대부분 0)
- 노이즈 큼 (dropout: 실제 발현되지만 0으로 측정)
- → **이 행렬을 표현(embedding)으로 만드는 게 foundation model 의 일** (scGPT, Geneformer)

---

## 단백질 — DNA의 결과물

### 시퀀스

20종 아미노산의 1차원 시퀀스. 각 아미노산은 한 글자 코드:

```
ACDEFGHIKLMNPQRSTVWY
```

예 (인간 헤모글로빈 베타 사슬):
```
MVHLTPEEKSAVTALWGKVNVDEVGGEALGRLLVVYPWTQRFFESFGDLSTPDAVMGNPK
VKAHGKKVLGAFSDGLAHLDNLKGTFATLSELHCDKLHVDPENFRLLGNVLVCVLAHHFG
KEFTPPVQAAYQKVVAGVANALAHKYH
```

### 구조

1D 시퀀스 → 폴딩 → 3D 구조. 이 3D 구조가 **기능을 결정**.

**Levels**:
- Primary: 1D 시퀀스
- Secondary: α-helix, β-sheet
- Tertiary: 3D 폴드
- Quaternary: 여러 사슬의 복합체

### 왜 어려운 문제였나

같은 시퀀스의 폴딩을 시뮬레이션으로 풀면 **천문학적 시간**. 자연은 ms 단위로 함.
**AlphaFold 2** (2021) 가 이걸 사실상 해결했고, **AlphaFold 3** (2024) 가 단백질 외에
DNA / RNA / 리간드까지 확장.

AI 엔지니어 관점: AlphaFold는 시퀀스 + MSA(다중 정렬) → 3D 좌표 회귀. Transformer +
geometric attention.

---

## 데이터 형식 — 한 표 요약

AI 엔지니어가 처음 보면 당황하는 파일들.

| 확장자 | 무엇 | 압축? | 비유 |
|---|---|---|---|
| `.fasta` / `.fa` | 시퀀스만 (DNA, RNA, 단백질) | text | "토큰 시퀀스 .txt" |
| `.fastq` / `.fq` | 시퀀스 + quality score (Phred) | text/gz | "토큰 + per-token confidence" |
| `.bam` / `.sam` | 정렬된 reads (BAM = binary) | binary | "tokenized + aligned to reference" |
| `.vcf` | 변이 목록 | text/gz | "diff 파일" |
| `.gtf` / `.gff` | 유전자 어노테이션 | text | "토큰 → 의미 매핑" |
| `.pdb` / `.cif` | 단백질 3D 좌표 | text | "토큰 → 3D 좌표" |
| `.h5ad` | AnnData (scRNA-seq) | HDF5 | "sparse matrix + metadata" |
| `.mtx` | Matrix Market sparse | text | "scipy sparse format" |

세부 페이지 (각 데이터 별)에서 더 다룹니다. 우선은 **이런 게 있다는 것만** 기억.

---

## "Bioinformatics 한 줄 요약"

생물학자에게 다음 한 줄만 이해해도 절반은 됩니다:

> **모든 생물학 데이터는 (1) 시퀀스 또는 (2) 카운트 매트릭스로 환원된다.
> 그래서 NLP·Transformer가 그렇게 잘 먹힌다.**

- DNA → 4-token 시퀀스 → DNA LM (Evo)
- 단백질 → 20-token 시퀀스 → Protein LM (ESM)
- 발현 → gene×sample 매트릭스 → DESeq2 (전통) / scGPT (모던)

---

## 다음 단계

이제 두 갈래:

### [데이터 레이어로](data/)
"내가 받은 데이터가 무엇인지" 부터 알고 싶다면.

- [Genomics (DNA)](data/genomics)
- [Transcriptomics (RNA)](data/transcriptomics)
- [Single-cell](data/single-cell)
- [Proteomics (단백질)](data/proteomics)
- [Structural (3D 구조)](data/structural)
- [기타 (epigenome · metabolome · imaging · microbiome)](data/others)

### [작업 레이어로](tasks/)
"내가 풀려는 문제부터" 알고 싶다면.

- [Sequence analysis](tasks/sequence-analysis)
- [Variant calling](tasks/variant-calling)
- [Expression analysis](tasks/expression-analysis)
- [Structure prediction](tasks/structure-prediction)
- [Clinical interpretation](tasks/clinical)
