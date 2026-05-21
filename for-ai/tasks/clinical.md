---
title: Clinical interpretation
parent: 작업 레이어
grand_parent: AI → Bio 트랙
nav_order: 5
---

# Clinical interpretation — 임상 해석
{: .no_toc }

VCF 한 줄 → "이게 환자의 증상을 설명하나? 치료 결정에 영향 있나?"
까지 가는 과정. 규제와 윤리가 가장 무거운 분야이기도 합니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 큰 그림

```
환자 sample (혈액, 조직)
   ↓
WGS / WES / panel seq
   ↓
[Variant calling]  GATK / DeepVariant → VCF
   ↓
[Annotation]  VEP / SnpEff / ANNOVAR → 유전자, 기능 결과, 빈도, 모델 점수
   ↓
[Filter]
   ├─ population frequency (gnomAD)
   ├─ predicted effect (HIGH/MODERATE)
   ├─ inheritance pattern (trio: de novo, comp het)
   └─ phenotype match (HPO)
   ↓
[Interpret]  ACMG/AMP 5-tier classification
   ├─ Pathogenic / Likely pathogenic   → 보고
   ├─ VUS                              → 모니터
   └─ Likely benign / Benign           → 제외
   ↓
[Report]  clinical-grade report → 임상의 → 환자
```

이 페이지는 **annotation 이후** 의 임상 해석 단계에 집중. variant calling은
[Variant calling](variant-calling), 데이터 형식은 [Variants](../data/variants) 참고.

{: .important }
> 임상 보고는 **인증된 검사실 (CLIA, CAP, KLAC)** 만 가능합니다. 이 페이지의
> 모든 코드는 **연구용**. 환자 결정에 사용 금지.

---

## Part 1 — ACMG/AMP 5-tier 가이드라인

[Richards et al. 2015](https://doi.org/10.1038/gim.2015.30) 가이드라인은
변이를 5개 클래스로 분류:

| 분류 | 의미 | 보고 |
|---|---|---|
| **Pathogenic (P)** | 질병 원인 거의 확실 (>99%) | 보고 |
| **Likely pathogenic (LP)** | 매우 가능 (>90%) | 보고 |
| **VUS (Variant of Uncertain Significance)** | 불확실 | 보고 가능, 결정 보류 |
| **Likely benign (LB)** | 양성 가능 (>90%) | 보통 비보고 |
| **Benign (B)** | 양성 확실 | 비보고 |

### 증거 코드 (Evidence codes)

분류는 28개 evidence code의 조합:

| 카테고리 | 코드 예 | 무엇 |
|---|---|---|
| Population | PM2, BA1, BS1 | gnomAD 빈도 |
| Computational | PP3, BP4 | in silico (CADD, REVEL, AlphaMissense) |
| Functional | PS3, BS3 | 실험 증거 |
| Segregation | PP1, BS4 | 가족 내 분리 |
| De novo | PS2, PM6 | 부모에 없음 |
| Allelic | PM3, BP2 | trans / cis configuration |
| Other DB | PP5, BP6 | ClinVar 등 (현재 사용 비추천) |

```
강도: BA1(독립) > BS(strong) > BP(supporting) — 양성
      PVS1(very strong) > PS > PM > PP                    — 병원성
```

### 조합 규칙 (단순화)

```
Pathogenic:
   1 × PVS1 + (≥1 PS) 또는 (≥2 PM) 또는 (1 PM + ≥1 PP) 또는 (≥2 PP)
   2 × PS
   1 × PS + (≥3 PM) 등...

Likely pathogenic:
   1 × PVS1 + 1 × PM
   1 × PS + (1-2 PM)
   ≥3 PM
   ...
```

자동 계산: [InterVar](https://wintervar.wglab.org), [Genoox Franklin],
[Varsome], [PathoMan]. 그러나 **자동 출력은 항상 사람이 검토**.

---

## Part 2 — 데이터베이스 한 표

### 변이 빈도

| DB | 무엇 | 코어 가치 |
|---|---|---|
| **gnomAD v4** (2024) | 807k exomes + 76k genomes | 인구별 AF, constraint score |
| **TOPMed Bravo** | 132k WGS | 미국 다양성 |
| **KoVariome** | 한국인 1700+ WGS | 한국인 baseline |
| **1000 Genomes** | 3500 WGS | 옛 표준, 작음 |

### 변이 해석

| DB | 무엇 | 핵심 사용 |
|---|---|---|
| **ClinVar** | 사람이 큐레이션한 임상 해석 | 일차 reference |
| **OMIM** | 유전 질환 ↔ 유전자 매핑 | 질환-유전자 매칭 |
| **HGMD** | 종합 변이 DB (상업) | 보조 |
| **DECIPHER** | CNV + clinical 데이터 | 희귀 발달질환 |
| **LOVD** | 유전자별 locus DB | 보조 |

### 암 (somatic) 전용

| DB | 무엇 |
|---|---|
| **COSMIC** | 종합 somatic mutation catalog |
| **OncoKB** | actionable 변이 (tier I-IV) |
| **CIViC** | community-curated 임상 증거 |
| **My Cancer Genome** | 약물-변이 매칭 |
| **CKB / JAX-CKB** | 임상 |

### 표현형

| DB | 무엇 |
|---|---|
| **HPO** (Human Phenotype Ontology) | 표현형 표준 용어 (~17k term) |
| **Orphanet** | 희귀 질환 분류 |
| **MONDO** | 통합 질환 온톨로지 |

### 약물유전체

| DB | 무엇 |
|---|---|
| **PharmGKB** | 약물 ↔ 유전자 ↔ 표현형 |
| **CPIC guidelines** | 임상 권고 (CYP2D6, TPMT, ...) |
| **DPWG** | 네덜란드 가이드라인 |

---

## Part 3 — VEP annotation

[Ensembl VEP](https://www.ensembl.org/vep) 는 가장 널리 쓰이는 annotator.

### 기본 명령

```bash
vep --input_file variants.vcf \
    --output_file annotated.vcf \
    --vcf --cache --dir_cache ~/.vep \
    --assembly GRCh38 \
    --offline \
    --everything \
    --pick \
    --plugin AlphaMissense,file=AlphaMissense_hg38.tsv.gz \
    --plugin REVEL,file=new_tabbed_revel_grch38.tsv.gz \
    --plugin CADD,snv=whole_genome_SNVs.tsv.gz,indels=gnomad.genomes.r4.0.indel.tsv.gz \
    --plugin SpliceAI,snv=spliceai_scores.raw.snv.hg38.vcf.gz \
    --custom gnomad.exomes.v4.0.sites.vcf.bgz,gnomADe,vcf,exact,0,AF,AF_popmax \
    --custom clinvar_20250501.vcf.gz,ClinVar,vcf,exact,0,CLNSIG,CLNDN
```

### 핵심 필드 (INFO/CSQ)

| 필드 | 의미 |
|---|---|
| `Consequence` | missense_variant, stop_gained, splice_donor_variant ... |
| `IMPACT` | HIGH / MODERATE / LOW / MODIFIER |
| `SYMBOL` | gene symbol |
| `HGVSc / HGVSp` | nomenclature (NM_...:c.123A>G, p.Lys41Arg) |
| `gnomADe_AF` | gnomAD exome AF |
| `gnomADe_AF_popmax` | 최대 인구 그룹 AF |
| `ClinVar_CLNSIG` | Pathogenic / Benign / VUS |
| `AlphaMissense` | 0.0~1.0 점수 + 라벨 |
| `REVEL` | 0.0~1.0 ensemble 점수 |
| `CADD_PHRED` | 1~99 PHRED-scaled |
| `SpliceAI` | 4개 점수 (acceptor/donor gain/loss) |

### Python 후처리 (cyvcf2)

```python
from cyvcf2 import VCF

vcf = VCF("annotated.vcf.gz")
csq_fields = vcf.header_iter()  # CSQ 필드 순서 파싱

for v in vcf:
    csq_raw = v.INFO.get("CSQ", "")
    for txn in csq_raw.split(","):
        fields = txn.split("|")
        # 인덱스로 SYMBOL, Consequence, gnomADe_AF, ClinVar_CLNSIG 추출
        ...
```

---

## Part 4 — In silico 변이 효과 예측 모델

### Missense 변이 (단백질 변경)

| 모델 | 출시 | 어떻게 동작 | 점수 |
|---|---|---|---|
| **AlphaMissense** | 2023 (DeepMind) | AlphaFold 백본 + structure context | 0~1, "likely benign / pathogenic / ambiguous" |
| **REVEL** | 2016 | 13개 도구 ensemble | 0~1 |
| **CADD** | 2014 / 갱신 | 모든 변이 통합 (SNV+indel) | PHRED 1~99 |
| **EVE** | 2021 | VAE on MSA | 0~1 + "uncertain" |
| **PrimateAI / PrimateAI-3D** | 2018 / 2023 | 영장류 polymorphism + 구조 | 0~1 |
| **ESM-1v / ESM-2 zero-shot** | 2021/2022 | LM mask probability | log-likelihood ratio |

{: .tip }
> 2026년 기준 **AlphaMissense + REVEL + SpliceAI** 조합이 대부분 임상 파이프라인의
> 기본. 한 모델만 믿지 말고 여러 점수를 같이 보기.

### ClinGen recommended thresholds (PP3/BP4 calibration)

[Pejaver et al. 2022](https://doi.org/10.1016/j.ajhg.2022.10.013) 권고:

| 점수 | Supporting pathogenic (PP3) | Supporting benign (BP4) |
|---|---|---|
| REVEL | ≥ 0.773 | ≤ 0.290 |
| AlphaMissense | ≥ 0.792 | ≤ 0.099 |
| CADD | (단독 사용 비추천) | - |

### Splicing

| 모델 | 핵심 |
|---|---|
| **SpliceAI** | DeepMind/Illumina, 4개 delta score |
| **Pangolin** | SpliceAI 후속, 더 정확 |
| **MMSplice** | modular 모델 |

### Regulatory / non-coding

| 모델 | 핵심 |
|---|---|
| **Enformer** | DNA → tissue-specific expression |
| **Borzoi** | Enformer 확장, longer context |
| **DeepSEA / Beluga** | regulatory feature 예측 |
| **CADD** | non-coding 포함 |

비유전자 영역 변이는 모델이 여전히 약함. 발견되어도 보고가 보수적.

### AlphaMissense — API / 파일 사용

DeepMind가 **모든 인간 missense 변이 (71M개) 사전 계산** 을 공개.

```python
import pandas as pd
am = pd.read_csv("AlphaMissense_hg38.tsv.gz", sep="\t", comment="#")
# columns: CHROM POS REF ALT genome uniprot_id transcript_id protein_variant am_pathogenicity am_class
hit = am.query("CHROM == 'chr17' and POS == 7674220 and REF == 'G' and ALT == 'A'")
```

API는 따로 없음 (전수 파일이 더 효율적). gnomAD나 UCSC에 통합되어 있음.

---

## Part 5 — HPO 기반 표현형-genotype 매칭

희귀 질환 진단의 핵심.

### HPO 구조

표준 용어 17,000+ 개. 부모-자식 관계 (DAG):

```
HP:0000118  Phenotypic abnormality
  └─ HP:0000707  Abnormality of the nervous system
      └─ HP:0011446  Abnormality of higher mental function
          └─ HP:0001249  Intellectual disability
              └─ HP:0010864  Intellectual disability, severe
```

### 환자 표현형 → 후보 유전자

```python
import phenopackets as pp
# 환자 HPO term 리스트:
patient_hpo = ["HP:0001249", "HP:0001250", "HP:0001263"]  # ID, seizures, GDD

# Exomiser / LIRICAL / Phen2Gene / GADO 등으로 유전자 랭킹
```

도구:
- **Exomiser** — VCF + HPO → ranked variant list (Sanger)
- **LIRICAL** — likelihood ratio 기반
- **Phen2Gene** — HPO만으로 유전자 priors
- **GADO** — gene network 기반

### LLM의 자리 (2026)

자유 텍스트 임상 노트 → HPO term 추출은 **RAG + LLM** 으로 자동화가 흔해짐.

```python
# 패턴: 임상 노트 → LLM이 HPO term suggestion → 큐레이터 확정
prompt = """
다음 임상 노트에서 HPO term을 JSON으로 추출:
"6-month-old male, severe global developmental delay, seizures since 4 months,
hypotonia, no head control..."
"""
# 출력: [{"HP:0011344": "global developmental delay, severe"}, ...]
```

[Phenotagger](https://github.com/ncbi-nlp/PhenoTagger), `clinphen`,
NCBI의 PubTator도 비슷한 일을 함.

---

## Part 6 — Trio 분석 (de novo, comp het)

희귀 질환에서 가장 강력한 필터.

```
                Proband (affected)
              GT: 0/1  (variant present)
                /     \
       Father          Mother
       0/0 (no)        0/0 (no)
                ↓
       → De novo 의심 (PS2 evidence)
```

### De novo 추출 — GATK / Hail

```python
import hail as hl
mt = hl.import_vcf("trio.vcf.bgz")
ped = hl.Pedigree.read("trio.ped")
mt = hl.de_novo(mt, pedigree=ped, pop_frequency_prior=mt.info.AF)
denovo_hi = mt.filter_rows(mt.de_novo.p_de_novo > 0.99)
```

### Compound heterozygous

두 다른 변이가 **trans** (다른 부모에서) 로 같은 유전자에 → 효과적으로 biallelic
(열성 모델).

```
Gene X:
  variant A: father het, mother ref → proband het
  variant B: father ref, mother het → proband het
```

Phasing (read-based 또는 trio-based) 으로 확인.

도구: `hail.compound_het`, `slivar`, `Exomiser` 가 자동 추출.

---

## Part 7 — 약물유전체 (Pharmacogenomics, PGx)

### 표준 워크플로우

```
WGS / WES / PGx panel
   ↓
[Star allele caller]  CYP2D6 같은 복잡 유전자 → *1/*4, *17/*41 등
   ↓
[Diplotype → Phenotype]  PharmGKB / CPIC table
   ↓
[Drug recommendation]  CPIC guideline
   예: codeine ↔ CYP2D6 ultrarapid → 회피
```

### Star allele 호출 도구

| 도구 | 핵심 |
|---|---|
| **Stargazer** | CYP2D6 등 12개+ 유전자 |
| **Aldy** | WGS / WES / amplicon |
| **PharmCAT** | VCF → CPIC report 자동 생성 |
| **Cypiripi / Astrolabe** | 옛 도구들 |

### PharmCAT — 가장 쉬운 시작

```bash
java -jar pharmcat.jar -vcf sample.vcf.gz -o pharmcat_out/
# pharmcat_out/sample.report.html ← CPIC 가이드라인 자동 매핑된 보고서
```

### CPIC tier별

- **Level A** — strong evidence, action 권고 (예: CYP2C19/clopidogrel, TPMT/thiopurine,
  HLA-B*57:01/abacavir)
- **Level B / C / D** — evidence 약함

---

## Part 8 — 종양 (somatic) variant 해석

생식세포 (germline) 와 다른 framework.

### Somatic tier 분류 — AMP/ASCO/CAP 2017

| Tier | 의미 |
|---|---|
| **I** | Strong clinical significance (승인 약물 표적) |
| **II** | Potential significance (clinical trial, off-label) |
| **III** | Unknown significance |
| **IV** | Benign / likely benign |

### 핵심 DB
- **OncoKB** — tier 1~4 mapping
- **CIViC** — community curation
- **JAX-CKB**, **My Cancer Genome**

### 워크플로우

```
Tumor-only or Tumor-Normal pair
   ↓
Somatic caller (Mutect2 / Strelka2 / DeepVariant somatic)
   ↓
[Filter]
   ├─ gnomAD AF > 0.001 → germline 의심 → 제외
   ├─ low VAF (<5%) → 노이즈/subclonal
   └─ Panel of Normals (PoN) 으로 artifact 제거
   ↓
[Annotate]  VEP + OncoKB API
   ↓
[Interpret]  Tier I~IV
```

### OncoKB API

```bash
curl -X GET "https://www.oncokb.org/api/v1/annotate/mutations/byProteinChange" \
  -H "Authorization: Bearer $ONCOKB_TOKEN" \
  -G --data-urlencode "hugoSymbol=BRAF" \
     --data-urlencode "alteration=V600E" \
     --data-urlencode "tumorType=Melanoma"
```

---

## Part 9 — LLM의 역할 (2026 현황)

임상 해석에서 LLM은 **보조** 입니다. 결정은 사람.

### 잘 되는 일

| 작업 | 예 |
|---|---|
| 임상 노트 → HPO 추출 | "GDD, seizures" → HP:0001263, HP:0001250 |
| OMIM / GeneCards 자동 조회 + 요약 | RAG로 |
| Literature 검색 (PubMed 한 변이) | 변이 ID 입력 → 관련 논문 요약 |
| 분류 reasoning draft | ACMG 증거 자동 수집 → 사람 검토 |

### 권장 패턴 (RAG)

```
[Query]  "이 환자 표현형과 변이를 같이 보면 어떤 질환 후보?"
   ↓
[Retrieve]  HPO + ClinVar + OMIM + PubMed 발췌
   ↓
[LLM]  context + 환자 정보 → reasoning + 후보 랭킹
   ↓
[Human curator]  검증 → 보고
```

도구: [Mendelian.co](https://mendelian.co), [Face2Gene + GeneGPT],
오픈소스로는 [PhenoBrain](https://github.com/), 자체 RAG 파이프라인.

### 안 되는 일 (2026-05 기준)

- **숫자 정확도**: AF, REVEL 점수 등을 LLM이 hallucinate
- **최신 ClinVar 상태**: knowledge cutoff 때문에 옛 분류 줄 수 있음
- **법적 책임**: LLM은 임상 결정의 책임 주체가 못 됨

---

## Part 10 — 함정 / 주의

### 1. 인구 stratification
gnomAD AF는 전체 평균. 특정 인구에서는 다름. **AF_popmax** 와 **한국인 (KoVariome)**
도 같이 보기.

### 2. Allele frequency bias
"gnomAD에 없으니 rare" → 그러나 gnomAD 자체가 European 편향. **AF=0** 이
rare 보장 아님 (단순히 안 본 것).

### 3. VUS 해석 강요
임상의가 "yes or no" 를 원해도 VUS는 VUS. 강제로 P/LP로 reclassify 하지 않기.
시간이 지나며 증거가 쌓이면 재분류 (ClinVar 자동 트래킹).

### 4. Secondary findings (ACMG SF v3.2)
WGS/WES 분석 중 **요청 외 발견** (예: BRCA1 P 변이가 우연히 보임). ACMG는 81개
유전자 (2024)에서 secondary finding 보고 권고. **사전 동의서에 반드시 포함**.

### 5. Variant nomenclature 혼동
같은 변이가 transcript에 따라 다른 HGVS. NM_000546.6:c.215C>G 와
NM_001126112.2:c.215C>G 는 다른 위치일 수 있음. **MANE Select transcript** 우선.

### 6. Germline vs Somatic 혼동
같은 변이도 germline (유전, 가족 영향) 과 somatic (종양 한정) 의미가 다름.
Tumor-only 시퀀싱이면 분리 어려움 → **paired normal 권장**.

### 7. Mosaicism
일부 세포에서만 변이 (VAF 5~30%). 표준 caller가 놓치기 쉬움. 의심되면
deep sequencing + 전용 caller (`MosaicHunter`, `DeepMosaic`).

### 8. 인플리케이션 vs 액션
"BRCA1 P 변이가 있다" ≠ "예방적 수술" 자동 의미 아님. 임상의 + 유전상담사 +
환자 결정.

---

## Part 11 — 규제, 윤리

### 검사실 인증

| 지역 | 인증 |
|---|---|
| 미국 | CLIA + CAP |
| 한국 | KLAC (대한임상검사정도관리협회), 진단검사의학회 |
| EU | IVDR (2022~), ISO 15189 |

연구 결과를 임상 결정에 쓰려면 **인증된 검사실에서 재확인 (Sanger 등)** 필수.

### 환자 데이터 보호

| 규제 | 지역 |
|---|---|
| **HIPAA** | 미국 |
| **개인정보보호법 + 생명윤리법** | 한국 |
| **GDPR** | EU (유전 정보는 special category) |

### IRB / 동의서

- **연구 시퀀싱** = IRB 승인 필요
- **임상 시퀀싱** = 사전 동의서 (secondary findings 포함)
- **소아** = 부모 동의 + assent (가능한 경우)

### 결과 통보 (Result return)

- **Actionable** 결과 (P/LP, 치료 가능 질환) → 통보 권고
- **VUS** → 보통 통보 안 함, 또는 재분류 시 통보
- **Secondary finding** → 사전 동의에 따름

{: .important }
> 한국에서는 **유전자 검사를 의료기관 외에서 직접 환자에게 제공 (DTC)** 하는
> 항목이 법으로 제한됩니다 (생명윤리법). 임상 변이 해석은 진단검사의학과 + 유전
> 상담 전문의 책임 영역.

---

## 실용 워크플로우 — Trio 희귀질환 진단

```
[입력]
   - Proband WGS (BAM/CRAM)
   - Father, Mother WGS
   - HPO terms (10~30개)
[1] Joint calling (GATK HaplotypeCaller → GenotypeGVCFs)
   ↓
[2] VEP annotation + AlphaMissense + REVEL + SpliceAI + gnomAD + ClinVar
   ↓
[3] Quality filter (DP, GQ, AB)
   ↓
[4] Frequency filter (gnomAD AF_popmax < 0.001 for dominant,
                      < 0.01 for recessive)
   ↓
[5] Inheritance filter
      ├─ de novo (autosomal dominant 의심)
      ├─ homozygous (consanguinity)
      └─ compound het (autosomal recessive)
   ↓
[6] Phenotype matching (Exomiser / LIRICAL with HPO)
   ↓
[7] ACMG/AMP classification (InterVar + 수동 큐레이션)
   ↓
[8] Top 5~10 후보 → 임상의 검토 → Sanger 확인
   ↓
[9] 보고
```

평균 진단율: 약 30~40% (WES, well-phenotyped rare disease).
재분석 (reanalysis) 으로 2~3년 후 추가 10% 확보.

---

## 다음

- 데이터: [Variants (VCF)](../data/variants)
- 작업: [Variant calling](variant-calling) · [Expression analysis](expression-analysis)
- 통합: [Mental model](../mental-model) · [Workflows](../../for-bio/workflows-tools)
