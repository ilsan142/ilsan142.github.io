---
title: Case A. RNA-seq을 Claude Code로
parent: Bridge 케이스 스터디
nav_order: 1
---

# Case A. RNA-seq 파이프라인을 Claude Code로
{: .no_toc }

셸이 두려운 생물학자가 GEO accession 한 줄로 DEG · enrichment 까지 도달하기.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 0. 케이스 한눈에

| 항목 | 내용 |
|---|---|
| 대상 독자 | 생물학 전공 · 셸 능숙도 중하 · R 약간 |
| 입력 | GEO accession `GSE164073` (control 3 · LPS treatment 3, paired-end) |
| 출력 | `results/deg_table.csv`, `results/go_bp.csv`, `results/figures/*.pdf` |
| 전통 방식 소요 | 5~7일 (도구 설치 · 옵션 검색 포함) |
| 에이전트 방식 소요 | 4~6시간 (대기 시간 포함) |
| 사람 검증 시간 | 1~2시간 (메타데이터 · QC plot · 해석) |
| 핵심 도구 체인 | `fastqc` → `trim-galore` → `STAR` → `featureCounts` → `DESeq2` → `clusterProfiler` |

{: .important }
> 이 케이스는 [for-bio/workflows](../for-bio/workflows#워크플로우-1--bulk-rna-seq-deg) 의
> 짧은 예시를 **풀 케이스 스터디로 확장한 것** 입니다. 거기서 한 문장이었던 단계가
> 여기서는 한 섹션이 됩니다.

---

## 1. 문제 정의

### 시나리오

당신은 면역학 박사후 연구원입니다. 지도교수는 어제 회의에서 이렇게 말했습니다.

> "최근 Nature Immunology에 LPS 자극 macrophage paper가 나왔는데, 우리 데이터랑
> 비교하고 싶어요. GEO에 raw FASTQ가 올라와 있다니까 DEG 표만 좀 뽑아 줘요. 이번 주 안에."

당신이 해야 할 일:

1. GEO에서 `GSE164073` (가상 accession; 실제로는 본인 데이터/논문 데이터로 치환) raw FASTQ 다운로드
2. control 3 vs LPS treatment 3, paired-end 데이터를 정렬·카운팅
3. DEG (DESeq2, `|log2FC| > 1`, `padj < 0.05`) 추출
4. GO BP enrichment + KEGG pathway
5. Volcano · MA · heatmap · PCA 시각화
6. 결과를 슬라이드 한 장으로 만들 수 있게 마크다운으로 정리

당신의 진짜 고민:
- STAR 인덱스를 만든 적이 없다
- DESeq2 라이브러리 콜이 헷갈린다 (`DESeqDataSetFromMatrix`?)
- 어제 conda 환경이 깨졌다
- 지난 학기 강의 노트가 어디 있는지 모른다

이 케이스는 **그 모든 것을 외주화** 하는 방법입니다.

---

## 2. 데이터 · 도구

### 입력

| 파일 | 출처 | 크기 |
|---|---|---|
| `GSE164073_RAW.tar` | NCBI GEO / SRA | ~30 GB (FASTQ 6쌍) |
| `GRCh38.primary_assembly.genome.fa` | GENCODE | 3 GB |
| `gencode.v45.primary_assembly.annotation.gtf` | GENCODE | 1.5 GB |
| `samples.csv` (메타데이터, 사람이 작성) | 본인 | 1 KB |

`samples.csv` 예시 — **이 파일은 사람이 직접 작성해야 합니다**. 에이전트가
GEO 페이지에서 추정할 수도 있지만, 검증은 사람의 몫:

```csv
sample_id,condition,replicate,fastq_r1,fastq_r2
SRR12345001,control,1,SRR12345001_1.fastq.gz,SRR12345001_2.fastq.gz
SRR12345002,control,2,SRR12345002_1.fastq.gz,SRR12345002_2.fastq.gz
SRR12345003,control,3,SRR12345003_1.fastq.gz,SRR12345003_2.fastq.gz
SRR12345004,LPS,1,SRR12345004_1.fastq.gz,SRR12345004_2.fastq.gz
SRR12345005,LPS,2,SRR12345005_1.fastq.gz,SRR12345005_2.fastq.gz
SRR12345006,LPS,3,SRR12345006_1.fastq.gz,SRR12345006_2.fastq.gz
```

### 도구 체인

```
SRA Toolkit (prefetch, fasterq-dump)
    ↓
FastQC + MultiQC          (QC 보고서)
    ↓
Trim Galore (cutadapt)    (어댑터 · 저품질 트리밍)
    ↓
STAR 2.7.11b              (스플라이스 인식 정렬)
    ↓
samtools                  (정렬/인덱싱)
    ↓
featureCounts (subread)   (유전자 단위 카운팅)
    ↓
R + DESeq2                (정규화 · DEG)
    ↓
clusterProfiler           (GO / KEGG enrichment)
    ↓
ggplot2 / EnhancedVolcano (시각화)
```

### 출력 구조 (목표)

```
results/
├── qc/
│   ├── multiqc_report.html
│   ├── pca.pdf
│   ├── sample_correlation_heatmap.pdf
│   └── dispersion.pdf
├── deg_table.csv             # padj < 0.05 & |log2FC| > 1
├── deg_table_all.csv         # 전체 (필터 없음)
├── go_bp.csv
├── kegg.csv
├── figures/
│   ├── volcano.pdf
│   ├── ma.pdf
│   ├── top50_heatmap.pdf
│   └── go_dotplot.pdf
└── REPORT.md
```

---

## 3. AI 없이 — 전통 방식 (헤매는 부분 포함)

다음은 셸 능숙도 중하인 박사후 연구원의 실제 일주일 로그를 약간 정돈한 것입니다.

### Day 1 — 환경 구성에서 좌초

```bash
# 시작은 좋아 보였음
conda create -n rnaseq python=3.11 -y
conda activate rnaseq
conda install -c bioconda sra-tools fastqc trim-galore star samtools subread multiqc -y

# 그런데 STAR이 안 깔림 (channel 충돌)
# CondaSolverError: ...

# 구글 검색 30분 → mamba 사용 권장
mamba install -c bioconda star=2.7.11b -y     # 또 다른 에러

# 결국 별도 환경
conda create -n star -c bioconda star=2.7.11b -y
# 환경을 두 개 왔다갔다 해야 함. 슬프지만 작동
```

R 패키지는 별도 지옥:

```r
# R 4.4 깔려 있음
BiocManager::install("DESeq2")
# → Compilation failed for package 'XML' ...
# → libxml2-dev 시스템 패키지 필요. sudo 없는데?
```

연구실 클러스터에 module이 있다는 걸 뒤늦게 알게 됨:

```bash
module load R/4.4
module load star/2.7.11b
```

### Day 2 — SRA 다운로드

```bash
# 처음엔 wget으로 시도
wget https://ftp.sra.ebi.ac.uk/.../SRR12345001/SRR12345001_1.fastq.gz
# 한 파일이 200MB 인데 속도 5MB/s ... 6시간

# prefetch가 빠르다는 걸 알게 됨
prefetch SRR12345001 SRR12345002 SRR12345003 SRR12345004 SRR12345005 SRR12345006
# 30분만에 끝

# fasterq-dump로 FASTQ 풀기
for s in SRR12345001 SRR12345002 SRR12345003 SRR12345004 SRR12345005 SRR12345006; do
  fasterq-dump --split-files $s -O fastq/
done
# 또 1시간
```

### Day 3 — STAR index

```bash
# GENCODE에서 다운로드
wget https://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_45/GRCh38.primary_assembly.genome.fa.gz
wget https://ftp.ebi.ac.uk/pub/databases/gencode/Gencode_human/release_45/gencode.v45.primary_assembly.annotation.gtf.gz
gunzip *.gz

# Index 빌드
mkdir star_idx
STAR --runMode genomeGenerate \
     --runThreadN 16 \
     --genomeDir star_idx \
     --genomeFastaFiles GRCh38.primary_assembly.genome.fa \
     --sjdbGTFfile gencode.v45.primary_assembly.annotation.gtf \
     --sjdbOverhang 100
# 1시간 30분 + 32GB RAM 필요. 첫 시도는 OOM.
# --limitGenomeGenerateRAM 31000000000 추가하고 재시도
```

### Day 4 — Trim + Align

```bash
# 어댑터 트리밍 — 옵션이 너무 많음, 그냥 기본
for s in $(ls fastq/*_1.fastq.gz | sed 's|.*/||;s|_1.fastq.gz||'); do
  trim_galore --paired --output_dir trimmed/ \
    fastq/${s}_1.fastq.gz fastq/${s}_2.fastq.gz
done

# Alignment
mkdir bam
for s in $(ls trimmed/*_1_val_1.fq.gz | sed 's|.*/||;s|_1_val_1.fq.gz||'); do
  STAR --runThreadN 16 \
       --genomeDir star_idx \
       --readFilesIn trimmed/${s}_1_val_1.fq.gz trimmed/${s}_2_val_2.fq.gz \
       --readFilesCommand zcat \
       --outSAMtype BAM SortedByCoordinate \
       --outFileNamePrefix bam/${s}_
done
# 각 샘플 30분 × 6 = 3시간
```

### Day 5 — Count + DESeq2

```bash
# featureCounts
featureCounts -p --countReadPairs -T 16 -s 2 \
              -a gencode.v45.primary_assembly.annotation.gtf \
              -o counts.txt bam/*_Aligned.sortedByCoord.out.bam

# -s 2 가 맞나? RSeQC infer_experiment.py 로 확인 필요한데 못 해봄
# → 일단 진행했다가 다시 돌릴 가능성 큼
```

R 스크립트 (예전 강의 노트를 디스크에서 찾는 데만 1시간):

```r
library(DESeq2)
library(clusterProfiler)
library(org.Hs.eg.db)

counts <- read.table("counts.txt", header=TRUE, skip=1, row.names=1)
counts <- counts[, 6:ncol(counts)]   # 메타데이터 컬럼 제거. 매번 헷갈림
colnames(counts) <- gsub("_Aligned.*", "", colnames(counts))
colnames(counts) <- gsub("bam.", "", colnames(counts))   # 경로 prefix 제거. 또 헷갈림

coldata <- read.csv("samples.csv", row.names=1)
coldata <- coldata[colnames(counts), ]   # 순서 매칭 — 여기서 자주 사고

dds <- DESeqDataSetFromMatrix(counts, coldata, design = ~ condition)
dds <- DESeq(dds)
res <- results(dds, contrast=c("condition","LPS","control"))

# 시각화는 또 다른 한 나절
# ...
```

### Day 6~7 — 그림 만들기 + 보고서

이쯤 되면 분석 자체보다 그림 옵션 찾는 데 시간이 더 들어갑니다.
지도교수는 "왜 이렇게 오래 걸려요?" 하고 묻습니다.

**그래서 일주일.**

---

## 4. AI와 함께 — Claude Code 위임

### 4.1. 환경 세팅 — 한 번만

```bash
mkdir -p ~/projects/rnaseq-lps && cd ~/projects/rnaseq-lps
mkdir -p data results
# samples.csv는 사람이 작성해서 이 폴더에 둠

claude
```

Claude Code가 처음 떴을 때:

> 이 폴더에서 RNA-seq 분석을 할 거야. 우선 `samples.csv` 봐 줘 (control 3 · LPS 3,
> paired-end). 필요한 도구 (sra-tools, fastqc, trim-galore, STAR 2.7.11b,
> samtools, subread, multiqc, R + DESeq2 + clusterProfiler) 가 깔려 있는지 점검하고,
> 빠진 건 conda로 새 환경 `rnaseq` 에 깔아 줘. R 패키지는 `BiocManager::install`
> 사용. 끝나고 `which STAR`, `R -e 'library(DESeq2)'` 결과 보여 줘.

에이전트가 알아서:
1. `conda env list` 로 기존 환경 확인
2. 누락된 도구 식별
3. 충돌 시 mamba 자동 사용 (또는 별도 env)
4. R 패키지 설치 (시스템 의존성 누락 시 conda 대체 패키지 제안)
5. 검증 명령으로 마침

{: .tip }
> 환경 구성은 **autopilot 으로 통째로 위임** 해도 안전한 단계입니다.
> 망가져도 환경을 지우고 다시 만들면 됩니다. `omc autopilot "rnaseq 환경 구성"` 가능.

### 4.2. 데이터 다운로드 — 검증 포인트 한 번

> `samples.csv` 의 6개 SRR accession 다운로드해 줘. `prefetch` → `fasterq-dump`
> 사용, `data/fastq/` 에 paired (`*_1.fastq.gz`, `*_2.fastq.gz`) 형태로. 끝나면
> `ls -lh data/fastq/` 와 각 파일의 첫 4줄 (gzip → head) 보여 줘.

**사람의 검증 포인트**:
- 파일 크기가 비슷한가 (한 샘플만 100MB이고 나머지 1GB면 다운로드 깨짐)
- FASTQ 헤더가 정상인가 (`@SRR...`)
- 6 × 2 = 12개 파일이 다 있는가

### 4.3. 풀 파이프라인 — 한 프롬프트

이게 핵심입니다. 한 줄에 다 위임:

```text
다음을 한번에 실행하는 Bash + R 파이프라인을 짜고 실제로 돌려 줘.

[입력]
- 메타데이터: samples.csv (이미 있음)
- FASTQ: data/fastq/  (paired-end, _1/_2)
- 참조 게놈: GRCh38 primary, Gencode v45
  - 이미 다운로드 된 게 있으면 재사용. 없으면 data/ref/ 에 받아 줘.
- STAR index 캐시: ~/refs/star_idx_grch38_v45/  (있으면 재사용, 없으면 빌드)

[단계]
1. FastQC + MultiQC (results/qc/multiqc_report.html)
2. Trim Galore (paired, q20, --length 36)
3. STAR alignment
   - --outSAMtype BAM SortedByCoordinate
   - --quantMode GeneCounts  (백업용 카운트)
   - --outSJtype Standard
4. samtools index
5. RSeQC infer_experiment.py 로 strandedness 자동 추정.
   결과를 stranded / reverse / unstranded 중 하나로 분류해서 echo,
   featureCounts 의 -s 옵션에 자동 반영.
6. featureCounts -p --countReadPairs -T 16 -a gencode.v45.gtf -o counts.txt
7. R 스크립트 작성·실행 (DESeq2):
   - condition 컬럼 (control / LPS) 모델: ~ condition
   - lfcShrink (apeglm), contrast c("condition","LPS","control")
   - results/deg_table.csv (padj<0.05 & |log2FC|>1)
   - results/deg_table_all.csv (필터 없음, log2FC 포함)
8. 시각화 (R):
   - PCA, sample correlation heatmap, dispersion → results/qc/
   - Volcano (EnhancedVolcano), MA, top50 heatmap (pheatmap) → results/figures/
9. clusterProfiler:
   - enrichGO(BP, org.Hs.eg.db), enrichKEGG
   - results/go_bp.csv, results/kegg.csv, dotplot PDF
10. results/REPORT.md 작성:
    - 각 단계 명령 한 줄 + 결과 파일 + 한 줄 코멘트 ("이 그림은 이렇게 봐야 함")
    - 통과/주의 신호 (PCA에서 condition 분리되는지 등)

[규칙]
- 중간 산출물은 work/ 에, 최종은 results/ 에
- 16 threads, 32GB RAM 가정
- 모든 명령을 실행 전에 echo 로 보여줄 것
- 한 단계 실패하면 멈추고 사람에게 보고 (set -euo pipefail)
- featureCounts strandedness 추정 결과는 반드시 화면에 출력
```

에이전트가 하는 일:
1. 위 텍스트를 분해해서 `run.sh` + `deseq2.R` + `enrichment.R` 작성
2. `set -euo pipefail` 으로 실패 시 멈춤
3. 단계마다 명령 echo 후 실행
4. 중간에 strandedness 추정 결과를 출력 (`Read 1 mapped to + : 0.92` → reverse stranded → `-s 2`)
5. 모든 결과를 `results/` 로 정리
6. `REPORT.md` 자동 작성

**걸리는 시간**: 1시간 (STAR alignment) + 30분 (count + DESeq2 + 그림). 그동안 사람은 다른 일.

### 4.4. 사람의 검증 ─ 한 시간

에이전트가 "끝났습니다" 라고 한 뒤, **사람이 반드시 봐야 하는 것**:

| 파일 | 확인 사항 |
|---|---|
| `samples.csv` ↔ `coldata` | 매핑이 일치하는가 |
| `results/qc/multiqc_report.html` | overall quality, duplication rate |
| `results/qc/pca.pdf` | PC1/PC2에서 condition 으로 분리되는가? batch 흔적? |
| `results/qc/dispersion.pdf` | 정상적인 trumpet 모양인가 |
| `results/deg_table.csv` | 컬럼: `padj` 인가 `pvalue` 인가 — 절대 헷갈리지 말 것 |
| `results/figures/volcano.pdf` | 알려진 LPS 반응 유전자 (`TNF`, `IL6`, `IL1B`, `NFKBIA`) 가 상위에 있나 |
| `results/go_bp.csv` | "immune response", "inflammatory" 가 상위에 있나 (positive control) |

마지막 줄이 핵심입니다 — **알려진 유전자/패스웨이가 잡히지 않으면 메타데이터 매핑이 뒤집혔거나
contrast 방향이 반대일 가능성이 큽니다.**

### 4.5. omc 모드 — 어떤 단계에 무엇을 쓰나

[oh-my-claudecode 가이드](../omc-guide/index.html) 의 모드를 이 케이스에 끼우면:

| omc 모드 | 적합한 단계 | 이유 |
|---|---|---|
| `omc autopilot` | 4.1 (환경 구성) | 망가져도 안전, 길이가 길어 한 번에 위임이 효율적 |
| `omc team` (planner/executor/verifier) | 4.3 (풀 파이프라인) | 큰 작업을 단계별로 plan → exec → verify 자동화 |
| `omc ralph` | strandedness 추정 → featureCounts 까지 | 결과가 기대치와 다르면 끈질기게 재시도 |
| `omc ultrawork` (병렬) | STAR vs Salmon 동시 실행 | 두 정렬 방법의 DEG 일치도 비교 |
| 단일 Claude Code | 4.4 (사람과의 대화) | 사람 판단 비중이 높은 단계는 단일 에이전트가 깔끔 |

병렬 실행 예 — STAR vs Salmon 두 트랙을 워크트리 분리로:

```bash
omc ulw "STAR + featureCounts 트랙" "Salmon + tximport 트랙"
# 두 결과의 DEG 교집합/차집합을 마지막에 비교
```

{: .tip }
> 처음부터 모든 단계를 자율 모드에 맡기지 말고, **4.3 의 한 프롬프트는 일반 Claude Code
> 로 직접** 보내는 것이 안전합니다. 결과를 사람이 본 뒤 다음 분석으로 넘어갈 때 omc 의
> 자동화를 늘려가세요.

### 4.6. 후속 — 같은 데이터로 다른 질문

DEG 표가 나오면 보통 후속 요청이 잇따릅니다:

> top 30 DEG 의 expression heatmap 을 batch (sample replicate) 까지 표시해서 다시 그려 줘.
> 표 컬럼 순서는 control1, control2, control3, LPS1, LPS2, LPS3 으로 고정.

> deg_table.csv 의 상위 유전자 중 transcription factor 만 골라서 (`hTFtarget` DB 기준)
> regulon enrichment 봐 줘. 결과 표로.

> 이 결과를 GSE164073 원논문 Figure 3b 와 비교 가능한 표 만들어 줘. 원논문 표는 PDF 에 있음.

이 반복은 전통 방식에서 가장 시간이 드는 부분 — 매번 R 스크립트 수정 — 인데, 에이전트에게는
1~2분짜리 추가 요청입니다.

---

## 5. 실패 모드와 회복 패턴

### 5.1. 잘못된 reference

**증상**: alignment rate 가 95% 가 아니라 60%, `unmapped` 다수.

**원인 후보**:
- 마우스 데이터를 GRCh38에 정렬
- GRCh38 vs hg19 혼동 (chromosome 표기 `1` vs `chr1`)
- GENCODE primary 가 아닌 full assembly (alt contig 포함) 사용

**회복**:
```text
> alignment rate 60% 인데, FASTQ 의 첫 100,000 reads 를 NCBI BLAST 로 종 추정하고,
> reference 가 맞는지 점검해 줘. 종이 다르면 올바른 reference로 다시 받아.
```

{: .warning }
> 에이전트는 종을 자동 추정하지 않습니다. **반드시 프롬프트에 종·assembly 버전 명시**.

### 5.2. Paired / Single 혼동

**증상**: featureCounts assigned reads 가 예상의 절반.

**원인**: trim_galore 출력 파일명에 `_val_1` 가 붙는데, single 모드로 처리해버림.

**회복 프롬프트**:
```text
> trim 결과가 paired 라는 걸 확실히 인식시키고 STAR/featureCounts 모두 paired 모드
> (`--paired`, `-p --countReadPairs`) 로 다시 돌려 줘.
```

방지: 첫 출력에서 read count 가 예상의 절반이면 의심. STAR `Log.final.out` 의
`Number of input reads` 가 FASTQ 절반이면 paired 가 single 로 들어간 것.

### 5.3. Strandedness 오분류

**증상**: featureCounts assigned 가 30% 미만. counts.txt 의 합이 비정상적으로 작음.

**원인**: `-s 2` (reverse) 가 맞는데 `-s 0` (unstranded) 로 돌렸거나 반대.

**회복**: RSeQC `infer_experiment.py` 로 추정 → 재실행.

```text
> infer_experiment.py 다시 돌려서 fraction of reads on + strand vs - strand
> 보여 주고, 분류 결과로 -s 옵션 다시 정해서 featureCounts 재실행.
```

해석 룰:
- `++,--/+-,-+` 비율 ≈ 0.5 → unstranded (`-s 0`)
- `++,--` 우세 (>0.8) → forward (`-s 1`)
- `+-,-+` 우세 (>0.8) → reverse (`-s 2`)

### 5.4. Batch effect 숨겨짐

**증상**: PCA 의 PC1 이 condition 이 아니라 sequencing batch / sample prep day 에 의해 갈라짐.

**원인**: 메타데이터에 batch 변수가 없거나 design 에 포함 안 됨.

**회복**:
```text
> samples.csv 에 batch 컬럼 추가했으니 (sample1-3=A, sample4-6=B 였는데 실은 1,3,5=A 4,6,2=B),
> DESeq2 design을 ~ batch + condition 으로 바꾸고 재실행. PCA 도 batch 보정 후 (limma::removeBatchEffect)
> 다시 그려 줘.
```

{: .important }
> Batch effect 가 condition 과 **완전 혼동(confounded)** 되어 있으면 통계적으로
> 분리 불가능합니다. 이 경우 에이전트는 "분석 가능합니다" 라고 답하면 안 됩니다 —
> 사람이 실험 설계 단계로 돌아가야 합니다.

### 5.5. Multiple testing 망각

**증상**: `pvalue < 0.05` 로 필터한 DEG 가 5,000 개 (말이 안 됨).

**원인**: `padj` 대신 `pvalue` 로 필터함.

**회복**:
```text
> deg_table.csv 의 필터 컬럼이 padj 인지 확인해 줘. pvalue 였으면 padj 로 바꿔
> 재추출. 또 lfcShrink (apeglm) 가 적용된 log2FoldChange 컬럼을 쓰는지도 확인.
```

방지: REPORT.md 에 **"필터 컬럼: padj"** 를 명시적으로 적게 시키기.

### 5.6. Contrast 방향 반대

**증상**: 알려진 LPS-up 유전자 (`TNF`, `IL6`) 가 log2FC 음수.

**원인**: `contrast=c("condition","control","LPS")` 로 들어가서 reference 가 LPS.

**회복**:
```text
> results(dds, contrast=c("condition","LPS","control")) 로 다시 추출. control 이 reference
> 가 되어야 함. 모든 다운스트림 (volcano, heatmap, enrichment) 도 재생성.
```

방지: 프롬프트에 **"LPS vs control, control 이 reference"** 를 명시.

### 5.7. clusterProfiler 의 universe 누락

**증상**: GO enrichment 결과의 p-value 가 비현실적으로 낮음.

**원인**: `enrichGO(genes, ...)` 호출 시 `universe` 인자를 명시하지 않으면 전체 게놈을 background 로 가정 — DEG 가 이미 발현 유전자만 모은 거라면 false positive 폭증.

**회복**:
```text
> enrichGO 호출 시 universe = rownames(dds)[rowSums(counts(dds)) > 10]
> (즉 발현된 모든 유전자) 로 명시. KEGG 도 동일.
```

### 5.8. 회복 패턴의 일반 원칙

| 상황 | 패턴 |
|---|---|
| "결과가 이상해" | 에이전트에게 결과 진단 부탁 → 가설 → 재실행. **결과를 직접 고치지 말 것** |
| 같은 실수 반복 | `CLAUDE.md` 또는 프로젝트 메모에 규칙 추가 (예: "항상 padj 사용, contrast 는 LPS-control 순서") |
| 에이전트가 자신만만하게 틀림 | 더 작게 쪼개기 — 한 단계만 실행하고 사람이 확인 |
| 환경이 또 깨짐 | `conda env export > env.yml` 로 잠금. 다음에는 그걸로 복구 |

---

## 마치며 — 이 케이스의 본질

**위임 가능한 것** (보일러플레이트):
- 도구 설치, 옵션 검색, 스크립트 작성, 시각화 코드, 보고서 마크다운

**위임 못 하는 것** (생물학):
- "이 유전자가 면역 반응을 설명하는가" — 도메인 지식
- "이 batch 가 condition 과 혼동되었는가" — 실험 설계 이해
- "이 결과를 다음 실험으로 어떻게 잇는가" — 연구 방향

코딩 에이전트는 일주일을 반나절로 줄여 줍니다. 그래서 사람은 **위 세 가지** 에 일주일을 쓸 수 있게 됩니다.

---

## 다음 케이스

→ [Case B. Inverse Folding 설계 루프](case-protein-design) — 반대 방향의 케이스.
AI 엔지니어가 단백질 디자인 루프를 만드는 과정.

## 참고

- [Bio → AI 트랙 / 4. 실전 워크플로우](../for-bio/workflows) — 더 짧은 버전의 같은 예
- [Bio → AI 트랙 / 3. Agent · MCP · Harness](../for-bio/agents-mcp) — omc 모드의 정식 소개
- [GENCODE Human Release 45](https://www.gencodegenes.org/human/release_45.html)
- [STAR manual](https://github.com/alexdobin/STAR)
- [DESeq2 vignette](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html)
- [clusterProfiler book](https://yulab-smu.top/biomedical-knowledge-mining-book/)
