---
title: Genomics
parent: 데이터 레이어
grand_parent: AI → Bio 트랙
nav_order: 1
---

# Genomics — DNA 데이터
{: .no_toc }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 핵심 개념

**Genomics** = 개체의 DNA 전체를 다루는 분야.

| 약어 | 의미 | 데이터 양 |
|---|---|---|
| **WGS** | Whole Genome Sequencing — 전체 게놈 | ~100 GB / sample (30x) |
| **WES** | Whole Exome Sequencing — 단백질 코딩 영역만 (~2%) | ~10 GB / sample |
| **Targeted** | 특정 패널만 | < 1 GB |
| **Long-read** | PacBio HiFi / ONT | 50-100 GB |

---

## 데이터 형식

### FASTA (`.fa`, `.fasta`)
시퀀스 자체만. **Reference genome** 이 이 형식.

```
>chr1
NNNNNNNNTAACCCTAACCCTAACCCTAACCCTAA...
>chr2
...
```

### FASTQ (`.fq`, `.fastq[.gz]`)
시퀀서 raw output. 시퀀스 + per-base quality.

```
@SRR1234.1 read1/1
ACGTACGTACGTACGT
+
IIIIIIHHHHHHGGGG    ← Phred quality (ASCII - 33)
```

**Quality (Phred Q)**: `Q = -10 log10(error_prob)`. Q30 = 1/1000 에러율.

### BAM / SAM (`.bam`, `.sam`)
**Aligned reads**. FASTQ를 reference에 정렬한 결과. BAM = binary 압축.

```
read1   16   chr1   12345   60   100M   *   0   0   ACGT...   IIII...
```

주요 컬럼: read name, flag (방향·페어), chrom, position, MAPQ, CIGAR, sequence, quality.

```bash
# 보기
samtools view file.bam | head
samtools flagstat file.bam   # 통계
samtools index file.bam      # .bai 인덱스 생성
```

### VCF (`.vcf[.gz]`)
**Variant call format**. reference 대비 변이 목록.

```
##fileformat=VCFv4.2
#CHROM  POS     ID      REF  ALT  QUAL  FILTER  INFO         FORMAT      sample1
chr1    12345   rs123   A    G    99    PASS    AF=0.5       GT:DP:GQ    0/1:30:99
```

GT (genotype): `0/0` = hom-ref, `0/1` = het, `1/1` = hom-alt.

---

## Reference Genomes

| 종 | 최신 빌드 | 어디서 |
|---|---|---|
| Human | GRCh38.p14 (= hg38) | Gencode v45+, Ensembl, UCSC |
| Human (old) | GRCh37 (= hg19) | 많은 임상 자료 여전히 hg19 |
| Mouse | GRCm39 (= mm39) | |
| Human T2T | T2T-CHM13 v2.0 | 2022, 진정한 "완성본" |

**중요**: 어떤 빌드인지 항상 확인. chr1:12345는 hg19와 hg38에서 다른 위치를 가리킵니다.

---

## 대표 도구

### Alignment
| 도구 | 용도 |
|---|---|
| `bwa-mem2` | 짧은 reads (Illumina), 표준 |
| `minimap2` | long reads (PacBio, ONT) + 짧은 reads도 |
| `STAR` / `HISAT2` | RNA-seq splice-aware |
| `bowtie2` | 짧은 reads, ChIP/ATAC |

### Variant Calling
| 도구 | 특징 |
|---|---|
| **GATK HaplotypeCaller** | 표준, germline / somatic |
| **DeepVariant** (Google) | CNN 기반, 정확도 ↑ |
| **Strelka2** | 빠름, somatic 강함 |
| **DeepSomatic** | DeepVariant의 somatic 버전 |

### Manipulation
- `samtools` — BAM 조작
- `bcftools` — VCF 조작
- `bedtools` — interval (BED) 산술
- `seqkit` — FASTA/FASTQ 일반 유틸

---

## AI 모델

### DNA Language Models

| 모델 | 컨텍스트 | 강점 |
|---|---|---|
| **Evo** / **Evo-2** | 131k / 1M bp | 장거리, 유전체 일반 |
| **Nucleotide Transformer** | 6 kb | InstaDeep, 안정 |
| **DNABERT-2** | 변동 BPE | 가벼움, 미세조정 쉬움 |
| **HyenaDNA** | 1M bp | 작은 모델, 긴 컨텍스트 |
| **Caduceus** | 131 kb | bi-directional SOTA |

**용도**:
- Promoter / enhancer 분류
- Splice site 예측 (SpliceAI 의 후계자들)
- 변이 효과 (CADD, REVEL 의 후계자)

### Variant Effect

| 모델 | 입력 | 출력 |
|---|---|---|
| **AlphaMissense** | missense 변이 | pathogenicity score |
| **CADD** | 모든 변이 | combined annotation score |
| **SpliceAI** | DNA | splice donor/acceptor 변화 |
| **Enformer** | 196 kb DNA | regulatory 효과 |

---

## 자주 마주치는 함정 (AI 엔지니어 대상)

### 1. **Strand**
DNA는 이중나선. 시퀀스 표기는 항상 + strand 기준. 도구 옵션 (`--strand`) 매번 의식.

### 2. **0-based vs 1-based 좌표**
- BAM / VCF / GFF: 1-based
- BED / UCSC : 0-based, half-open

이걸 헷갈리면 모든 좌표가 1씩 어긋남.

### 3. **Chromosome 이름**
`chr1` vs `1` — 같은 reference 여도 도구마다 표기 다름. `chrM` vs `chrMT` 도 자주 혼동.

### 4. **Compression**
대부분 `.gz` (gzip) 또는 `.bgz` (block gzip, indexable). bgzip된 VCF는 `tabix` 로 인덱싱.

### 5. **Read 그룹**
GATK는 BAM에 `@RG` 헤더가 없으면 거부. `picard AddOrReplaceReadGroups` 로 보강.

---

## 데이터 받을 수 있는 곳

| 출처 | 무엇 | 접근 |
|---|---|---|
| **SRA** (NCBI) | 모든 raw sequencing | `sra-tools`, `pysradb` |
| **ENA** (EBI) | SRA 미러 + Europe 데이터 | 직접 FTP / `enaBrowserTools` |
| **GEO** | Functional genomics (RNA-seq, ChIP-seq, ...) | `GEOparse` |
| **1000 Genomes** | 인간 variation | FTP |
| **gnomAD** | variant 빈도 | https://gnomad.broadinstitute.org |
| **UK Biobank** | ~500k 영국 코호트 | 신청 필요 |
| **TCGA** | 암 유전체 | GDC portal |
| **All of Us** | 미국 1M 코호트 | 신청 |

---

## 작은 실습 코드

```python
import pysam

# BAM 읽기
bam = pysam.AlignmentFile("sample.bam", "rb")
for read in bam.fetch("chr1", 1000000, 1001000):
    print(read.query_name, read.reference_start, read.mapping_quality)

# VCF
vcf = pysam.VariantFile("variants.vcf.gz")
for rec in vcf.fetch("chr1", 1000000, 1001000):
    print(rec.pos, rec.ref, rec.alts, rec.samples["sample1"]["GT"])
```

```python
# Evo (DNA LM) 임베딩
from evo import Evo
model, tok = Evo("evo-1-131k-base").from_pretrained()
seq = "ATCG" * 1000
emb = model.embed(seq)
```

---

## 다음 단계

- 작업 측면: → [Variant calling](../tasks/variant-calling), [Sequence analysis](../tasks/sequence-analysis)
- 관련 데이터: → [Transcriptomics](transcriptomics) (RNA), [Structural](structural) (단백질)
