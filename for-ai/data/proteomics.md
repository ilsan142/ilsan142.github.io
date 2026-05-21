---
title: Proteomics
parent: 데이터 레이어
grand_parent: AI → Bio 트랙
nav_order: 4
---

# Proteomics — 단백질 데이터
{: .no_toc }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 두 갈래

"Proteomics" 라고 하면 두 가지가 섞입니다:

1. **시퀀스 기반** — UniProt / FASTA. ESM-2/3 같은 protein LM이 다루는 영역.
2. **질량분석 기반** — 실험으로 어떤 단백질이 얼마나 있는지 측정. Mass spec.

AI 엔지니어가 처음 진입하면 **시퀀스 기반** 이 훨씬 친숙합니다.

---

## 시퀀스 기반

### 데이터 형식 — FASTA

```
>sp|P68871|HBB_HUMAN Hemoglobin subunit beta OS=Homo sapiens GN=HBB
MVHLTPEEKSAVTALWGKVNVDEVGGEALGRLLVVYPWTQRFFESFGDLSTPDAVMGNPK
VKAHGKKVLGAFSDGLAHLDNLKGTFATLSELHCDKLHVDPENFRLLGNVLVCVLAHHFG
KEFTPPVQAAYQKVVAGVANALAHKYH
```

20종 아미노산:
```
A  Ala (Alanine)        F  Phe        K  Lys        P  Pro        U  Sec (selenocysteine, rare)
C  Cys                  G  Gly        L  Leu        Q  Gln        V  Val
D  Asp                  H  His        M  Met        R  Arg        W  Trp
E  Glu                  I  Ile        N  Asn        S  Ser        Y  Tyr
                                                      T  Thr
```

### 주요 데이터베이스

- **UniProt** — 모든 단백질의 표준 DB. SwissProt (curated) + TrEMBL (자동).
- **InterPro / Pfam** — 도메인 / family
- **PDB** — 3D 구조 (시퀀스도 함께)

### 단백질 시퀀스 분석

```python
from Bio import SeqIO
for rec in SeqIO.parse("proteins.fasta", "fasta"):
    print(rec.id, len(rec.seq))
```

#### 상동성 검색

| 도구 | 속도 | 용도 |
|---|---|---|
| `BLAST` (psiblast, blastp) | 느림 | 표준 |
| `DIAMOND` | 100x BLAST | 대규모 |
| `MMseqs2` | 매우 빠름 | 클러스터링 + 검색 |
| `Foldseek` | 구조 기반 | 시퀀스 너무 멀 때 |

#### 정렬

`MAFFT`, `MUSCLE`, `Clustal Omega` — 다중 시퀀스 정렬 (MSA).
AlphaFold 2의 입력으로 필수.

---

## AI 모델 — Protein LM

### ESM 시리즈 (Meta)

| 모델 | 파라미터 | 컨텍스트 |
|---|---|---|
| ESM-2 (t6_8M) | 8M | 1024 |
| ESM-2 (t12_35M) | 35M | 1024 |
| ESM-2 (t33_650M) | 650M | 1024 |
| ESM-2 (t36_3B) | 3B | 1024 |
| ESM-2 (t48_15B) | 15B | 1024 |
| ESM-3 | 1.4B / 7B / 98B | 더 길음 |

```python
from transformers import EsmTokenizer, EsmModel
import torch

tok = EsmTokenizer.from_pretrained("facebook/esm2_t33_650M_UR50D")
model = EsmModel.from_pretrained("facebook/esm2_t33_650M_UR50D")

seq = "MVHLTPEEKSAVTALWGKVNVDEVGG"
inputs = tok(seq, return_tensors="pt")
with torch.no_grad():
    out = model(**inputs)
    # out.last_hidden_state : (1, L+2, 1280)
    # CLS / EOS 제거하고 사용
```

### 용도별

| 용도 | 모델 |
|---|---|
| 임베딩 → classifier (기능 예측) | ESM-2, ProtT5 |
| Zero-shot 변이 효과 (ΔΔG) | ESM-2 (likelihood) |
| 시퀀스 → 구조 | ESMFold (ESM-2 + structure head) |
| 구조 → 시퀀스 (inverse folding) | ESM-IF, ProteinMPNN |
| 시퀀스 생성 / 디자인 | ESM-3, RFdiffusion + ProteinMPNN |

### ProteinMPNN

구조 → 시퀀스 회귀의 SOTA. 단백질 디자인의 핵심 도구.

```bash
git clone https://github.com/dauparas/ProteinMPNN
python ProteinMPNN/protein_mpnn_run.py \
    --pdb_path target.pdb --num_seq_per_target 8 \
    --out_folder out/
```

---

## 질량분석 (Mass Spectrometry) 기반

이쪽은 AI 엔지니어에게 익숙하지 않은 영역.

### 원리

1. 세포 추출액의 단백질을 효소 (trypsin) 로 잘게 자름 → peptide
2. LC로 분리, MS로 m/z 측정
3. Spectrum → peptide 시퀀스 → 단백질로 환원

### 데이터 형식

| 확장자 | 무엇 |
|---|---|
| `.raw` (Thermo) / `.d` (Bruker) / `.wiff` (Sciex) | 벤더 raw |
| `.mzML` / `.mzXML` | 오픈 표준 |
| `.mgf` | peak list |
| `.mzTab` / `.tsv` | 정량 결과 |

### 분석 도구

| 도구 | 형태 | 특징 |
|---|---|---|
| **MaxQuant** | GUI / CLI | 표준, free |
| **DIA-NN** | CLI | DIA 방식 표준 |
| **FragPipe / MSFragger** | GUI | 매우 빠름 |
| **Proteome Discoverer** | 상용 | Thermo 공식 |
| **Skyline** | GUI | targeted |

### AI in mass spec

- **DeepNovo / Casanovo** — de novo peptide sequencing
- **Prosit** — fragmentation spectrum 예측
- **MS²Rescore** — DL 기반 PSM rescoring

---

## 자주 마주치는 함정

### 1. **UniProt accession vs gene symbol**
같은 단백질의 다른 ID 체계. `P53` (gene) ≠ `P04637` (UniProt). 변환 도구 `mygene`, `gprofiler`.

### 2. **Isoform**
같은 유전자에서 alternative splicing으로 여러 단백질. UniProt에서 canonical만 vs all isoform.

### 3. **PTM (Post-translational modification)**
인산화, 메틸화 등 modification은 시퀀스만으로 표현 안 됨. PSI-MOD 어휘 사용.

### 4. **시퀀스 길이 한계**
ESM-2는 1024 aa 컨텍스트. 큰 단백질(>1024)은 sliding window 필요.

### 5. **Mass spec FDR**
peptide identification은 false discovery rate 1% 가 표준. PSM (peptide spectrum match)
수준에서 FDR 계산.

---

## 데이터 받을 수 있는 곳

- **UniProt** — 모든 단백질 시퀀스
- **PDB** — 3D 구조 (~22만 구조)
- **AlphaFold DB** (EBI) — 2억+ 예측 구조
- **PRIDE** (EBI) — mass spec 데이터 저장소
- **MassIVE** — UCSD mass spec 저장소
- **CPTAC** — 임상 proteogenomics

---

## 다음

- 관련 데이터: [Structural](structural)
- 작업: [Structure prediction](../tasks/structure-prediction)
