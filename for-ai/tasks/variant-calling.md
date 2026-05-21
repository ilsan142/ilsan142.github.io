---
title: Variant calling
parent: 작업 레이어
grand_parent: AI → Bio 트랙
nav_order: 2
---

# Variant calling — BAM에서 VCF로
{: .no_toc }

정렬된 reads에서 reference와 다른 지점 — 변이(variant) — 를 추출하는 작업.
genomics 분석의 가장 큰 단일 작업이자, AI 모델이 가장 먼저 SOTA를 빼앗은 영역.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 한 줄 정의

**Variant calling** = `BAM` (정렬된 reads) → `VCF` (reference 대비 변이 목록).

같은 BAM이라도 **무슨 변이를 찾느냐** 에 따라 도구가 완전히 달라집니다.

---

## 변이 종류와 의사결정 트리

```
변이의 종류 / 시나리오?
│
├─ Germline (개체의 타고난 변이, 모든 세포 동일)
│   │
│   ├─ short variant (SNV / small indel, <50 bp)
│   │   ├─ short-read Illumina  ──→  GATK HaplotypeCaller / DeepVariant
│   │   └─ long-read PacBio/ONT ──→  Clair3 / DeepVariant (PacBio mode)
│   │
│   └─ structural variant (SV, ≥50 bp)
│       ├─ short-read ──→  Manta, Delly, Lumpy
│       └─ long-read  ──→  Sniffles2, CuteSV, pbsv
│
├─ Somatic (암 등 후천적 변이, tumor만)
│   │
│   ├─ tumor + matched normal ──→  Mutect2 (GATK), Strelka2, DeepSomatic
│   ├─ tumor-only             ──→  Mutect2 tumor-only (PON 필수)
│   └─ low VAF (<5%) deep-seq ──→  VarDict, DeepSomatic
│
└─ CNV (copy number variant)
    ├─ from WGS/WES   ──→  CNVkit, GATK gCNV, Control-FREEC
    ├─ from SNP array ──→  PennCNV
    └─ from scRNA-seq ──→  inferCNV, copyKAT
```

### 종류별 한 줄 요약

| 종류 | 크기 / 위치 | 비유 |
|---|---|---|
| **SNV** (single nucleotide variant) | 1 bp 치환 | typo 한 글자 |
| **Small indel** | 1-50 bp 삽입/결실 | 한두 단어 추가/삭제 |
| **SV** (structural variant) | ≥50 bp — inversion, translocation, large del/dup | 문단 통째로 이동 |
| **CNV** (copy number variant) | 큰 영역의 카피수 변화 | 같은 챕터 2~3번 복사 |
| **Germline** | 부모로부터 물려받음, 모든 세포 동일 | 책 자체의 오탈자 |
| **Somatic** | 후천적 (보통 암), 일부 세포만 | 책 인쇄 중 일부 페이지 손상 |

---

## GATK 표준 파이프라인 — Germline short variant

업계 사실상 표준. 30분 안에 그림 머릿속에 박아두기.

```
BAM (sorted, dedup)
   │
   ▼
[1] BQSR — Base Quality Score Recalibration
   │   (BaseRecalibrator → ApplyBQSR)
   │   known-sites: dbSNP, Mills indels
   ▼
recal.bam
   │
   ▼
[2] HaplotypeCaller — per-sample, GVCF mode
   │   (-ERC GVCF)
   ▼
sample.g.vcf.gz   (× N samples)
   │
   ▼
[3] GenomicsDBImport — joint database
   │   (또는 CombineGVCFs)
   ▼
[4] GenotypeGVCFs — joint genotyping
   │
   ▼
joint.vcf.gz
   │
   ▼
[5] VQSR (≥30 exomes) 또는 hard filter (작은 N)
   │   VariantRecalibrator → ApplyVQSR
   ▼
filtered.vcf.gz
   │
   ▼
[6] Annotation (다음 섹션)
```

### 실전 — single sample germline

```bash
SAMPLE=patient01
REF=GRCh38.primary_assembly.fa

# 1. BQSR
gatk BaseRecalibrator \
  -I bam/${SAMPLE}.dedup.bam -R ${REF} \
  --known-sites dbsnp_146.hg38.vcf.gz \
  --known-sites Mills_and_1000G_gold_standard.indels.hg38.vcf.gz \
  -O recal/${SAMPLE}.table

gatk ApplyBQSR \
  -I bam/${SAMPLE}.dedup.bam -R ${REF} \
  --bqsr-recal-file recal/${SAMPLE}.table \
  -O recal/${SAMPLE}.recal.bam

# 2. HaplotypeCaller (GVCF)
gatk HaplotypeCaller \
  -I recal/${SAMPLE}.recal.bam -R ${REF} \
  -O gvcf/${SAMPLE}.g.vcf.gz \
  -ERC GVCF

# 3-4. Joint genotyping (single sample도 동일 호출)
gatk GenotypeGVCFs \
  -R ${REF} -V gvcf/${SAMPLE}.g.vcf.gz \
  -O vcf/${SAMPLE}.vcf.gz

# 5. Hard filter (작은 N일 때 — VQSR 대신)
gatk VariantFiltration \
  -R ${REF} -V vcf/${SAMPLE}.vcf.gz \
  --filter-expression "QD < 2.0" --filter-name "QD2" \
  --filter-expression "FS > 60.0" --filter-name "FS60" \
  --filter-expression "MQ < 40.0" --filter-name "MQ40" \
  -O vcf/${SAMPLE}.filtered.vcf.gz
```

{: .tip }
> Trio (proband + 부모) 면 세 샘플의 GVCF를 하나로 묶어 `GenotypeGVCFs`.
> 그러면 Mendelian inconsistency / de novo 분석이 가능해집니다.

---

## AI 모델 — variant calling 의 SOTA

전통 도구 (GATK) 가 표준이지만, **정확도 1위는 대부분 AI 모델** 입니다.
PrecisionFDA 등 벤치마크에서 매년 상위권은 DeepVariant 계열.

### Variant caller 모델 비교

| 모델 | 만든 곳 | 입력 | 강점 | 비용 |
|---|---|---|---|---|
| **DeepVariant** | Google | short-read, PacBio HiFi | CNN, 정확도 SOTA. germline 표준 후보 | GPU 권장, CPU도 OK |
| **DeepSomatic** | Google | tumor (± normal) | somatic용 DeepVariant. low VAF 강함 | GPU 권장 |
| **Clair3** | HKU | ONT, PacBio | RNN 기반. long-read SOTA | CPU도 빠름 |
| **PEPPER-Margin-DeepVariant** | UCSC | ONT | ONT germline 표준 파이프라인 | GPU |
| **NVIDIA Parabricks** | NVIDIA | (GATK·DeepVariant GPU 가속) | 6시간 → 30분. 알고리즘 동일 | A100/H100 |

### DeepVariant — 실전

```bash
# CPU 실행 (작은 영역)
docker run --rm \
  -v $(pwd):/data \
  google/deepvariant:1.6.0 \
  /opt/deepvariant/bin/run_deepvariant \
    --model_type=WGS \
    --ref=/data/GRCh38.fa \
    --reads=/data/bam/patient01.dedup.bam \
    --output_vcf=/data/vcf/patient01.dv.vcf.gz \
    --output_gvcf=/data/gvcf/patient01.dv.g.vcf.gz \
    --num_shards=16
```

`model_type` 옵션:
- `WGS` — Illumina whole genome
- `WES` — Illumina whole exome (different model)
- `PACBIO` — PacBio HiFi
- `ONT_R104` — Oxford Nanopore R10.4 chemistry

### Clair3 — long-read

```bash
# ONT R10.4
run_clair3.sh \
  --bam_fn=ont.bam \
  --ref_fn=GRCh38.fa \
  --output=clair3_out \
  --threads=16 \
  --platform="ont" \
  --model_path="/opt/models/r1041_e82_400bps_sup_v500"
```

### NVIDIA Parabricks — GPU 가속

전체 GATK best practices + DeepVariant 가 GPU에서 한 명령으로:

```bash
pbrun deepvariant \
  --ref GRCh38.fa \
  --in-bam patient01.dedup.bam \
  --out-variants patient01.dv.vcf.gz \
  --num-gpus 4
# 30x WGS — A100 4장 기준 ~25분
```

---

## Somatic variant calling

### Mutect2 — tumor + matched normal

```bash
# Panel of Normals (PoN) — 같은 시퀀싱 배치 normal sample 모음에서
gatk Mutect2 \
  -R GRCh38.fa \
  -I tumor.bam -tumor TUMOR_SM \
  -I normal.bam -normal NORMAL_SM \
  --panel-of-normals pon.vcf.gz \
  --germline-resource af-only-gnomad.hg38.vcf.gz \
  -O somatic.vcf.gz

# Filter
gatk FilterMutectCalls \
  -R GRCh38.fa -V somatic.vcf.gz \
  --contamination-table contamination.table \
  -O somatic.filtered.vcf.gz
```

### Mutect2 — tumor only

matched normal이 없으면 false positive 폭증 → PoN 필수, gnomAD germline filter 필수.

### DeepSomatic — AI 기반

```bash
docker run --rm -v $(pwd):/data \
  google/deepsomatic:1.6.0 \
  run_deepsomatic \
    --model_type=WGS \
    --ref=/data/GRCh38.fa \
    --reads_tumor=/data/tumor.bam \
    --reads_normal=/data/normal.bam \
    --output_vcf=/data/somatic.ds.vcf.gz \
    --sample_name_tumor=TUMOR --sample_name_normal=NORMAL \
    --num_shards=16
```

low VAF (5% 미만) tumor heterogeneity 영역에서 DeepSomatic이 Mutect2보다 강한 경향.

---

## SV / CNV

### Structural variant

| 도구 | 입력 | 강점 |
|---|---|---|
| **Manta** | short-read paired | split-read + paired-end. WGS 표준 |
| **Delly** | short-read | smaller SV 강함 |
| **Lumpy** / **smoove** | short-read | (legacy) |
| **Sniffles2** | long-read (PacBio/ONT) | long-read SV 표준 |
| **CuteSV** | long-read | Sniffles2 대안, 일부 종류에 정확 |
| **pbsv** | PacBio HiFi | PacBio 공식 |

```bash
# Manta — WGS germline SV
configManta.py --bam patient01.dedup.bam --referenceFasta GRCh38.fa \
               --runDir manta_run
manta_run/runWorkflow.py
# → manta_run/results/variants/diploidSV.vcf.gz
```

### CNV

| 도구 | 입력 | 특징 |
|---|---|---|
| **CNVkit** | WES/WGS (target enrichment 친화) | exome용 표준 |
| **GATK gCNV** | WES/WGS cohort | cohort 필요 (~30+) |
| **Control-FREEC** | WGS/WES | tumor-normal |
| **inferCNV** / **copyKAT** | scRNA-seq | single-cell 안에서 CNV 추정 |

---

## AlphaMissense — missense 변이의 pathogenicity

### 무엇

DeepMind / Google (2023). **missense 변이** (단백질 코딩 영역의 SNV) 가 단백질 기능에
해로운지 (pathogenic) 양성인지 (benign) 0-1 score 예측. AlphaFold 2 백본 + supervised
fine-tuning.

- 인간 단백질 모든 가능한 missense 변이 ~71M 개 사전 계산본 공개
- ClinGen, ClinVar 와 높은 일치
- VCF의 missense 컬럼 옆에 score 한 줄 붙이면 임상 우선순위에 즉시 활용 가능

### 왜 variant calling 페이지에 있나

기술적으로는 **annotation** 이지만, 임상 variant 워크플로우의 마지막 필터로 거의
필수가 됐기 때문에 여기에 같이 둡니다.

### 실전 — VEP 플러그인으로 붙이기

```bash
# 사전 계산된 score 다운로드 (~1 GB)
# https://console.cloud.google.com/storage/browser/dm_alphamissense
wget https://storage.googleapis.com/dm_alphamissense/AlphaMissense_hg38.tsv.gz
tabix -s 1 -b 2 -e 2 -f -S 1 AlphaMissense_hg38.tsv.gz

vep -i sample.vcf.gz -o sample.vep.vcf \
  --cache --offline --assembly GRCh38 \
  --plugin AlphaMissense,file=AlphaMissense_hg38.tsv.gz \
  --vcf
```

이러면 VCF INFO 필드에 `am_pathogenicity` 추가됨.

### 동급 / 인접 모델

| 모델 | 대상 | 만든 곳 |
|---|---|---|
| **AlphaMissense** | missense (protein-coding SNV) | DeepMind |
| **CADD** | 모든 변이 (combined annotation) | UW (전통, 비AI) |
| **REVEL** | missense (앙상블) | (전통) |
| **PrimateAI-3D** | missense | Illumina |
| **ESM Variants** | missense, zero-shot from ESM-2 | Meta |
| **SpliceAI** | splice region | Illumina |
| **AlphaGenome** | regulatory + splicing + 종합 | DeepMind (2026) |

{: .tip }
> 2026 현재 임상 missense 해석은 보통 **AlphaMissense + ClinVar + gnomAD 빈도** 3종
> 조합. AlphaMissense P 점수 (>0.564) + ClinVar P/LP + gnomAD AF < 0.001
> 이 가장 흔한 필터.

---

## 후처리 — Filter → Annotation → 매칭

### 필터링

```bash
# bcftools — PASS 만, FILTER 컬럼 기준
bcftools view -f PASS sample.vcf.gz -Oz -o sample.pass.vcf.gz

# QUAL / DP / 영역 필터
bcftools view -i 'QUAL>=30 && INFO/DP>=10' sample.pass.vcf.gz \
  -R targets.bed -Oz -o sample.high.vcf.gz

# Multi-allelic split (annotation 전 필수)
bcftools norm -m- -f GRCh38.fa sample.high.vcf.gz -Oz -o sample.norm.vcf.gz
```

### Annotation 도구 비교

| 도구 | 만든 곳 | 강점 | 단점 |
|---|---|---|---|
| **VEP** (Variant Effect Predictor) | Ensembl | 표준, 플러그인 생태계 (AlphaMissense 등) | 첫 setup 복잡 |
| **snpEff** | (open source) | 간단, 빠름 | 플러그인 부족 |
| **ANNOVAR** | (academic) | 가벼움, 명령형 | 라이선스 |

### 외부 DB 매칭

| DB | 무엇 | 용도 |
|---|---|---|
| **gnomAD** v4 | ~800k 개체 variant 빈도 | 흔한 변이 제거 (AF < 0.001 같은 필터) |
| **ClinVar** | 임상 큐레이션된 변이 분류 | P / LP / VUS / LB / B 라벨 |
| **dbSNP** | 알려진 SNP rsID | ID 부여 |
| **COSMIC** | somatic mutation in cancer | 암 변이 정황 |
| **OMIM** | 유전질환-유전자 매핑 | 표현형 연결 |
| **HPO** | 표현형 온톨로지 | 환자 증상 → 유전자 우선순위 |

---

## 자주 만나는 함정

### 1. Mendelian inconsistency

Trio (부, 모, 자식) joint calling에서 자식 GT가 부모로부터 불가능한 조합 (예:
부=0/0, 모=0/0, 자식=1/1) 이 나오면 calling 또는 sample swap 문제.
정상 비율 < 1%, 5% 넘으면 의심.

### 2. Contamination

다른 sample의 DNA가 섞이면 false het 폭증. GATK `CalculateContamination` 또는
`VerifyBamID2` 로 추정. >1% 면 분석 보류.

### 3. Batch effect

서로 다른 시퀀싱 배치 / 키트 / 시기의 샘플을 joint calling 하면 batch별로
artifact가 다름. PCA 또는 PC plot 으로 batch 분리되는지 확인.

### 4. Low VAF somatic

암에서 VAF (variant allele frequency) 5% 미만 변이는 PCR error / sequencing error 와
구분 어려움. duplex sequencing, UMI consensus, DeepSomatic / VarDict 같은 low-VAF
특화 caller 필요.

### 5. CHIP (clonal hematopoiesis)

나이 든 환자에서 혈액 세포 일부의 후천적 변이 (DNMT3A, TET2 등) 가 "germline like"
로 보임. tumor-only somatic calling 시 normal blood를 normal로 쓰면 진짜 somatic이
빠짐. ctDNA / 액체생검 분야에서 특히 큰 함정.

### 6. Multi-allelic 미정규화

`A → C,T` 같은 multi-allelic site를 그대로 두면 annotation / joining이 깨짐.
**항상** `bcftools norm -m-` 으로 분해 후 분석.

### 7. Reference build 혼동 (재강조)

VCF의 `##contig` 헤더와 다음 단계 도구의 reference가 다르면 결과가 침묵 속에서 망함.
파이프라인 첫 줄에서 강제 확인.

```bash
bcftools view -h sample.vcf.gz | grep ^##contig | head -3
```

### 8. PoN 부재 (tumor-only)

matched normal 없이 PoN도 없으면 germline 변이를 somatic으로 잘못 부르기 쉬움.
같은 시퀀싱 배치 normal 30+ 샘플 모아 GATK CreateSomaticPanelOfNormals 권장.

---

## 작은 코드 — VCF 분석

```python
import pysam

vcf = pysam.VariantFile("patient01.vep.vcf.gz")

# 통계: PASS only, biallelic, missense + AlphaMissense P > 0.564
n = 0
for rec in vcf.fetch():
    if "PASS" not in rec.filter.keys(): continue
    if len(rec.alts) != 1: continue
    csq = rec.info.get("CSQ")
    if not csq: continue
    # VEP CSQ는 |로 split된 다중 transcript
    for entry in csq:
        fields = entry.split("|")
        # CSQ 순서: VEP 헤더 line에서 확인 (Consequence, Impact, am_pathogenicity, ...)
        consequence = fields[1]
        am = fields[28] if len(fields) > 28 else ""
        if "missense_variant" in consequence and am:
            try:
                if float(am) > 0.564:
                    n += 1
                    break
            except ValueError:
                pass

print(f"AlphaMissense-P missense: {n}")
```

---

## 다음

- VCF를 임상적으로 해석하려면 → [Clinical interpretation](clinical)
- BAM 만들기로 돌아가려면 → [Sequence analysis](sequence-analysis)
- 데이터 형식 복습 → [Genomics 페이지](../data/genomics)
- 단백질 측면의 변이 효과 → [Structure prediction](structure-prediction) (변이가 구조에 미치는 영향)
