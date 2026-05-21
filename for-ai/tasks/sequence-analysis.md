---
title: Sequence analysis
parent: 작업 레이어
grand_parent: AI → Bio 트랙
nav_order: 1
---

# Sequence analysis — FASTQ에서 BAM까지
{: .no_toc }

시퀀서가 뱉어낸 raw reads를 reference에 정렬해 "사용 가능한 데이터" 로 바꾸는 단계.
대부분의 후속 분석이 여기서 시작됩니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 한 줄 정의

**Sequence analysis** = `FASTQ` (raw reads) → `BAM` (정렬된 reads)
또는 → `quant` (RNA-seq 의 transcript-level abundance).

이 단계는 거의 모든 후속 작업 — variant calling, DEG, ChIP/ATAC peak, methylation —
의 **공통 전처리**. 그래서 가장 먼저 익혀야 합니다.

---

## 파이프라인 개요

```
┌──────────┐   ┌────┐   ┌──────┐   ┌─────────┐   ┌──────┐   ┌──────────────┐
│ FASTQ    │──→│ QC │──→│ trim │──→│ align   │──→│ sort │──→│ dedup + index │──→ BAM
│ (.fq.gz) │   │    │   │      │   │         │   │      │   │               │
└──────────┘   └────┘   └──────┘   └─────────┘   └──────┘   └──────────────┘
                  │         │           │
              fastqc     trim_galore  bwa-mem2 / minimap2
              fastp      fastp        STAR / HISAT2 / bowtie2
              multiqc    cutadapt
```

각 단계 한 줄 정리.

| 단계 | 목적 | 대표 도구 |
|---|---|---|
| **QC** | read 품질·adapter contamination·중복도·GC bias 파악 | `fastqc`, `multiqc`, `fastp` |
| **Trim** | adapter 제거, 3' 끝 low-quality 자르기 | `trim_galore`, `fastp`, `cutadapt` |
| **Align** | reads를 reference 좌표에 매핑 | `bwa-mem2`, `minimap2`, `STAR`, `HISAT2`, `bowtie2` |
| **Sort** | 좌표 순으로 정렬 | `samtools sort` |
| **Dedup** | PCR duplicate 표시·제거 | `picard MarkDuplicates`, `samtools markdup`, `umi_tools` |
| **Index** | random access 위한 `.bai` 생성 | `samtools index` |

---

## QC — 데이터를 믿기 전에

### 무엇을 보나

| 지표 | 정상 | 위험 신호 |
|---|---|---|
| Per-base quality | 끝쪽 일부 감소 OK, 평균 Q30+ | 중간부터 Q20 미만 → trim 필수 |
| Per-sequence GC | 종 평균에 단봉 | 다봉 → contamination |
| Sequence duplication | RNA-seq은 높음 정상, WGS는 낮아야 | WGS에서 50% 초과 → over-amplified |
| Adapter content | trimming 전엔 보임 | 후에도 남으면 trimming 실패 |
| Overrepresented sequences | 0~소수 | rRNA / adapter / contaminant 의심 |

### 실전 — fastqc + multiqc

```bash
# 모든 FASTQ에 fastqc
mkdir -p qc/fastqc
fastqc -t 8 -o qc/fastqc *.fastq.gz

# multiqc로 한 페이지 리포트
multiqc -o qc qc/fastqc
# → qc/multiqc_report.html 한 파일로 모든 샘플 비교
```

### 더 빠르게 — fastp (QC + trim 한방)

```bash
fastp \
  -i sample_R1.fastq.gz -I sample_R2.fastq.gz \
  -o trim/sample_R1.fq.gz -O trim/sample_R2.fq.gz \
  --detect_adapter_for_pe \
  --qualified_quality_phred 20 \
  --length_required 36 \
  --thread 8 \
  --html qc/fastp/sample.html --json qc/fastp/sample.json
```

`fastp` 한 번이 `fastqc` + `trim_galore` 를 거의 대체합니다. 대용량에서 압도적으로 빠름.

---

## Trimming — 무엇을 자르나

| 종류 | 왜 |
|---|---|
| **Adapter** | 라이브러리 제작 때 붙은 짧은 합성 시퀀스. 정렬 실패의 주범 |
| **Low-quality 3'** | Illumina는 read 끝쪽 품질 떨어짐 |
| **PolyG (NovaSeq)** | 신호 없는 부분이 G로 잘못 읽힘 |
| **PolyA tail (RNA-seq)** | 정렬 방해 |

### Trim 도구 비교

| 도구 | 특징 |
|---|---|
| `trim_galore` | `cutadapt` wrapper. 표준 옵션 자동, 보고서 깔끔 |
| `fastp` | 매우 빠름. QC + trim 동시. 기본 선택지로 추천 |
| `cutadapt` | 가장 유연. adapter 시퀀스 직접 지정 가능 |
| `Trimmomatic` | 옛 표준. 여전히 자주 보임. Java 의존 |

```bash
# trim_galore (Illumina TruSeq adapter 자동 감지)
trim_galore --paired --quality 20 --length 36 \
  -o trim/ sample_R1.fastq.gz sample_R2.fastq.gz
```

{: .warning }
> Paired-end 데이터에서 R1, R2 를 따로 trim 하면 read 짝이 깨집니다.
> 반드시 `--paired` (trim_galore) 또는 `-i/-I` (fastp) 옵션으로 페어로 처리.

---

## Aligner 선택 — 가장 중요한 결정

**입력 데이터 종류** 와 **목적** 에 따라 완전히 다른 aligner를 씁니다.

### 결정 트리

```
시퀀싱 플랫폼?
│
├─ Illumina short reads (50-300 bp)
│   │
│   ├─ DNA (WGS, WES, ChIP, ATAC) ──→ bwa-mem2 (사실상 표준)
│   │                                   대안: bowtie2 (특히 ChIP/ATAC)
│   │
│   └─ RNA-seq (splice-aware 필요) ──→ STAR (정확) 또는 HISAT2 (메모리 효율)
│                                       대안: salmon / kallisto (alignment-free quant)
│
├─ PacBio HiFi (15-25 kb, Q30+)     ──→ minimap2 -ax map-hifi
│                                       (variant이면 pbmm2 wrapper)
│
└─ Oxford Nanopore (1 kb ~ 1 Mb)    ──→ minimap2 -ax map-ont
                                        DNA: -ax map-ont
                                        RNA: -ax splice (direct RNA / cDNA)
```

### Aligner 비교 표

| Aligner | 입력 | Splice-aware | 메모리 | 속도 | 언제 쓰나 |
|---|---|---|---|---|---|
| `bwa-mem2` | Illumina short DNA | ❌ | ~30 GB (human idx) | 빠름 | **DNA 표준**. WGS/WES/ChIP/ATAC |
| `bowtie2` | Illumina short | ❌ | ~3 GB | 빠름 | 작은 게놈, ChIP-seq, ATAC-seq |
| `STAR` | Illumina RNA | ✅ | ~30 GB | 빠름 | **RNA-seq 표준**. splice junction 정확 |
| `HISAT2` | Illumina RNA | ✅ | ~8 GB | 매우 빠름 | RNA-seq + 메모리 부족할 때 |
| `minimap2` | long reads (PacBio/ONT) + short | ✅ (옵션) | ~10 GB | 매우 빠름 | **long-read 표준**. multi-purpose |
| `salmon` / `kallisto` | Illumina RNA | (pseudo-align) | ~3 GB | 매우 빠름 | RNA-seq quant **만** 필요할 때. BAM 안 나옴 |

{: .note }
> `bwa-mem2` vs `bwa` — 같은 알고리즘인데 `bwa-mem2` 가 SIMD 최적화로 ~2배 빠름.
> 2026년 새 프로젝트는 무조건 `bwa-mem2`. 옛 파이프라인은 `bwa mem` 그대로 유지해도 결과 동일.

---

## 실전 — bwa-mem2 + samtools 파이프

WGS / WES 표준 DNA 정렬.

```bash
# 0. index (한 번만)
bwa-mem2 index -p idx/GRCh38 GRCh38.primary_assembly.fa
samtools faidx GRCh38.primary_assembly.fa

# 1. 정렬 + sort (한 파이프, 디스크 절약)
SAMPLE=patient01
bwa-mem2 mem -t 16 \
  -R "@RG\tID:${SAMPLE}\tSM:${SAMPLE}\tLB:lib1\tPL:ILLUMINA" \
  idx/GRCh38 \
  trim/${SAMPLE}_R1.fq.gz trim/${SAMPLE}_R2.fq.gz \
  | samtools sort -@ 8 -o bam/${SAMPLE}.sorted.bam -

# 2. duplicate 표시
samtools markdup -@ 8 bam/${SAMPLE}.sorted.bam bam/${SAMPLE}.dedup.bam

# 3. index
samtools index bam/${SAMPLE}.dedup.bam

# 4. 통계
samtools flagstat bam/${SAMPLE}.dedup.bam
samtools stats bam/${SAMPLE}.dedup.bam | grep ^SN
```

**Read group (`-R`) 가 왜 중요한가**: GATK / Mutect2 / 대부분의 variant caller는
`@RG` 헤더 없는 BAM을 거부합니다. 정렬 시점에 넣는 게 가장 깔끔.

---

## 실전 — STAR (splice-aware RNA-seq)

```bash
# 0. index — Gencode GTF 가 splice junction 제공
STAR --runMode genomeGenerate \
     --genomeDir star_idx \
     --genomeFastaFiles GRCh38.primary_assembly.fa \
     --sjdbGTFfile gencode.v45.annotation.gtf \
     --sjdbOverhang 100 \
     --runThreadN 16

# 1. 정렬 (2-pass 모드가 splice junction 정확도 ↑)
STAR --runThreadN 16 \
     --genomeDir star_idx \
     --readFilesIn trim/${SAMPLE}_R1.fq.gz trim/${SAMPLE}_R2.fq.gz \
     --readFilesCommand zcat \
     --outSAMtype BAM SortedByCoordinate \
     --twopassMode Basic \
     --outFileNamePrefix bam/${SAMPLE}_

# 2. index
samtools index bam/${SAMPLE}_Aligned.sortedByCoord.out.bam
```

**Quant 만 필요하면** salmon / kallisto가 STAR보다 10배 빠르고 정확도 동등.
DEG 분석만 할 거면 BAM 자체가 필요 없습니다.

```bash
# salmon — alignment-free quant
salmon index -t gencode.v45.transcripts.fa -i salmon_idx -k 31
salmon quant -i salmon_idx -l A \
  -1 trim/${SAMPLE}_R1.fq.gz -2 trim/${SAMPLE}_R2.fq.gz \
  --validateMappings -p 16 -o quant/${SAMPLE}
```

---

## 실전 — minimap2 (long-read)

```bash
# PacBio HiFi (genome)
minimap2 -t 16 -ax map-hifi GRCh38.fa hifi_reads.fq.gz \
  | samtools sort -@ 8 -o sample.hifi.bam -
samtools index sample.hifi.bam

# ONT (genome)
minimap2 -t 16 -ax map-ont GRCh38.fa ont_reads.fq.gz \
  | samtools sort -@ 8 -o sample.ont.bam -

# ONT direct RNA (splice)
minimap2 -t 16 -ax splice -uf -k14 GRCh38.fa rna_reads.fq.gz \
  | samtools sort -@ 8 -o sample.rna.bam -
```

---

## BAM 후처리 한 묶음

| 단계 | 도구 | 명령 |
|---|---|---|
| Sort | `samtools sort` | `samtools sort -@ 8 -o out.bam in.bam` |
| Mark/remove duplicates | `samtools markdup` / `picard MarkDuplicates` | `samtools markdup -@ 8 in.bam out.bam` |
| Index | `samtools index` | `samtools index out.bam` → `out.bam.bai` |
| Quality filter | `samtools view -q` | `samtools view -bq 30 in.bam > q30.bam` |
| Region extract | `samtools view` | `samtools view -b in.bam chr1:1000-2000` |
| Coverage | `samtools depth` / `mosdepth` | `mosdepth -t 4 sample in.bam` |

**Dedup 주의**: UMI (Unique Molecular Identifier) 가 있는 라이브러리는
`umi_tools dedup` 또는 `fgbio` 를 써야 합니다. 일반 markdup은 UMI를 모름.

---

## AI 시대 — 정렬에서 AI의 위치

**솔직한 현실**: 정렬 자체에서 AI는 거의 SOTA가 아닙니다. `bwa-mem2`, `minimap2`,
`STAR` 같은 전통 도구가 여전히 표준이고, 속도·정확도 모두 충분합니다.

AI 가 강한 곳은 **정렬 전 단계** 입니다.

### 1. Basecalling (ONT)

Oxford Nanopore raw signal → DNA 시퀀스 변환은 **딥러닝이 SOTA**.

| 도구 | 만든 곳 | 특징 |
|---|---|---|
| **Dorado** | ONT 공식 | 2026년 표준. GPU 필수. accurate (sup), fast (hac) 모드 |
| **Bonito** | ONT (실험) | 새 아키텍처 실험장. 연구 목적 |
| **Guppy** | ONT (legacy) | 옛 표준. Dorado로 마이그레이션 권장 |

```bash
# Dorado — POD5 raw → FASTQ
dorado basecaller sup pod5_files/ > reads.bam
samtools fastq reads.bam | gzip > reads.fq.gz
```

### 2. Consensus / pre-processing

| 도구 | 입력 | 출력 | 이득 |
|---|---|---|---|
| **DeepConsensus** (Google) | PacBio subreads | HiFi-quality reads | Q30+ consensus, 정확도 ↑ |
| **DeepVariant pileup** | BAM | 변이 후보 | (다음 페이지: variant calling) |

### 3. Adapter / contamination detection

전통 도구 (fastp 등) 가 충분하지만, 메타지놈 contamination 같은
복잡한 케이스에선 ML 분류기 (Kraken2 + 자체 모델) 가 사용됨.

{: .note }
> **요약**: FASTQ → BAM 까지의 표준 파이프라인에서 AI를 끼우는 지점은 ONT basecalling
> 정도입니다. AI 모델이 폭발하는 건 **다음 단계** (variant calling, structure prediction,
> single-cell annotation) 부터. 이 페이지는 AI 엔지니어가 처음 만나는 "전통의 영역"
> 으로 익혀두세요.

---

## 자주 만나는 함정

### 1. Read group 누락

```
ERROR: SAM/BAM file does not have read groups
```

→ `bwa-mem2 mem -R "@RG\tID:...\tSM:...\tLB:...\tPL:ILLUMINA"` 로 정렬 시점에 추가.
이미 정렬했으면 `picard AddOrReplaceReadGroups` 로 보강.

### 2. Paired-end mismatch

`_R1.fastq.gz` 에는 100M reads, `_R2.fastq.gz` 에는 99.5M reads — 다르면 정렬 깨짐.
trim 단계에서 paired 처리 안 했을 가능성 큼. `fastp --paired_end_check` 또는
`seqkit pair` 로 정합 확인.

### 3. Reference build 혼동

- BAM 헤더의 `@SQ` 가 `chr1` 인데 VCF는 `1` 로 시작 → 모든 도구가 에러
- hg19 reads를 hg38 reference에 정렬 → 좌표 다 어긋남, 그러나 에러 안 남
- **항상**: `samtools view -H file.bam | grep @SQ | head` 로 reference 확인

### 4. Strand 정보 잃기 (RNA-seq)

라이브러리 키트가 stranded인데 unstranded로 quant 하면 antisense reads 잘못 셈.
모르면 `RSeQC infer_experiment.py` 로 추정:

```bash
infer_experiment.py -i sample.bam -r gencode.v45.bed
# → "1++,1--,2+-,2-+: 0.05" 식으로 stranded 정보 추정
```

### 5. PCR duplicate vs optical duplicate

`MarkDuplicates` 는 두 종류를 합쳐 표시. 패턴드 flow cell (NovaSeq) 은 optical
duplicate가 많아서 `--OPTICAL_DUPLICATE_PIXEL_DISTANCE 2500` 옵션 필요.

### 6. `.bai` vs `.csi` 인덱스

매우 큰 염색체 (식물, T2T 인간) 는 `.bai` 가 인덱싱 불가. `samtools index -c` 로
`.csi` 생성. 일부 도구가 `.csi` 미지원이니 확인.

### 7. RNA-seq에서 multi-mapper 처리

repeat / paralog 에 다중 매핑되는 reads. STAR 기본은 10번까지 허용 (`--outFilterMultimapNmax 10`).
이걸 그대로 quant 하면 over-counting. featureCounts는 기본적으로 multi-mapper
제외 (`-M` 옵션 켜야 포함).

---

## 작은 코드 — pysam으로 BAM 탐색

```python
import pysam

bam = pysam.AlignmentFile("sample.dedup.bam", "rb")

# 헤더 확인
print(bam.header.references[:5])   # chrom 이름
print(bam.header.get("RG"))        # read group

# 특정 영역 reads
n_mapped, n_dup = 0, 0
for read in bam.fetch("chr1", 1_000_000, 1_100_000):
    if read.is_unmapped: continue
    n_mapped += 1
    if read.is_duplicate: n_dup += 1
print(f"mapped={n_mapped}, dup={n_dup}")

# CIGAR로 splice (N) 확인 — RNA-seq 인지 빠른 판별
for read in bam.fetch("chr1", 0, 10_000_000):
    if read.cigartuples and any(op == 3 for op, _ in read.cigartuples):
        print("splice read:", read.query_name)
        break
```

---

## 다음

- 정렬이 끝났으면 → [Variant calling](variant-calling) (BAM → VCF) 으로
- 또는 → [Expression analysis](expression-analysis) (BAM/quant → DEG / clusters)
- 데이터 형식 복습 → [Genomics 페이지](../data/genomics)
- 단백질 시퀀스 다루기 → [Proteomics](../data/proteomics) + [Structure prediction](structure-prediction)
