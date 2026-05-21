---
title: Structure prediction
parent: 작업 레이어
grand_parent: AI → Bio 트랙
nav_order: 4
---

# Structure prediction — 단백질 구조 예측 · 디자인
{: .no_toc }

시퀀스 → 3D, 그리고 그 역방향. AI가 가장 극적으로 생물학을 바꾼 분야이고,
AI 엔지니어가 가장 친숙하게 느끼는 분야이기도 합니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 한 장 요약

```
                       내 입력은?
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
   단량체              복합체               역방향
       │                    │                    │
  ┌────┴────┐         ┌─────┴─────┐         ┌────┴────┐
  │MSA 있음 │         │ 단백질-단백질│         │구조 → 시퀀스│
  │   ↓     │         │ 단백질-DNA  │         │ProteinMPNN │
  │ AF2    │         │ 단백질-RNA  │         │ ESM-IF    │
  │ColabFold│         │ 단백질-리간드│         └────┬────┘
  └─────────┘         │     ↓       │              │
                      │ AF3 / Boltz │         de novo 디자인
                      │ RFAA / Chai │         RFdiffusion
   ┌────────┐         └─────────────┘         ESM-3 / Chroma
   │MSA 없음/│
   │ 빠른   │
   │ 스크리닝│
   │   ↓    │
   │ESMFold │
   │OmegaFold│
   └────────┘
```

모델 카탈로그 전체는 [Bio AI 모델](../../for-bio/bio-ai-models#1-단백질-구조-예측) 참고.

---

## Part 1 — Forward: 시퀀스 → 구조

### 의사결정 트리 (상세)

| 입력 | 추천 모델 | 이유 |
|---|---|---|
| 단량체, 진화 정보 있음 | **AlphaFold 2 (ColabFold)** | 여전히 단량체 SOTA, 안정 |
| 단량체, MSA 만들 시간 없음 | **ESMFold** | single-seq, 100배 빠름 |
| 단량체, orphan / 합성 시퀀스 | **ESMFold · OmegaFold** | MSA 없어도 됨 |
| 호모/헤테로 멀티머 | **AlphaFold 3 · Boltz-1/2** | 복합체 SOTA |
| 단백질-리간드 | **AlphaFold 3 · Boltz-2 · RFAA** | small molecule 지원 |
| 단백질-DNA/RNA | **AlphaFold 3 · RFAA · Chai-1** | 핵산 지원 |
| 항체 (Fv, Fab) | **AF3 · Boltz-2 · IgFold** | CDR 루프 특수 |
| Disordered / IDR | (안 됨) | 어떤 모델도 신뢰 안 됨 |

### MSA: 왜 AF2는 필요하고 ESMFold는 안 필요한가

**핵심 인사이트**:

| 모델 | 진화 정보 어디서? |
|---|---|
| AlphaFold 2 | **MSA** (그 시퀀스의 호몰로그 수백 개를 명시적으로 정렬) |
| ESMFold | **언어모델 가중치** (ESM-2가 250M 시퀀스를 학습하면서 이미 흡수) |
| AlphaFold 3 / Boltz | MSA + small templates + ligand conditioning |

→ ESMFold는 **빠르지만 보존이 약한 단백질, 진화적으로 고립된 시퀀스에서 약함**.
→ AF2는 **MSA 깊이가 충분할 때 매우 강함**. 호몰로그 < 30개면 정확도 떨어짐.

### ColabFold — AF2의 표준 사용법

```bash
# 로컬 설치
pip install "colabfold[alphafold]" -q

# 단일 시퀀스 예측
colabfold_batch \
    input.fasta \
    output_dir/ \
    --model-type alphafold2_ptm \
    --num-recycle 3 \
    --num-models 5 \
    --amber           # OpenMM relaxation
```

또는 [ColabFold notebook](https://colab.research.google.com/github/sokrypton/ColabFold/blob/main/AlphaFold2.ipynb) 으로 GPU 없이도 가능.

### ESMFold — 단일 시퀀스, 즉시

```python
import torch
from transformers import AutoTokenizer, EsmForProteinFolding

tok   = AutoTokenizer.from_pretrained("facebook/esmfold_v1")
model = EsmForProteinFolding.from_pretrained("facebook/esmfold_v1",
                                              low_cpu_mem_usage=True).cuda()

seq = "MVLSPADKTNVKAAWGKVGAHAGEYGAEALERMFLSFPTTKTYFPHF..."
inputs = tok([seq], return_tensors="pt", add_special_tokens=False).to("cuda")
with torch.no_grad():
    out = model(**inputs)

# PDB 문자열로 저장
from transformers.models.esm.openfold_utils.protein import to_pdb, Protein
pdb = model.output_to_pdb(out)[0]
open("pred.pdb","w").write(pdb)
```

A100 1장 / 단백질 1개 / 길이 < 400aa = **수 초**.

### Boltz-1/2 — AF3급 오픈소스

```bash
pip install boltz
boltz predict input.yaml --use_msa_server --output_dir out/
```

`input.yaml` 예 (단백질-리간드 도킹):

```yaml
sequences:
  - protein:
      id: A
      sequence: "MVL...KYR"
  - ligand:
      id: B
      smiles: "CC(=O)Oc1ccccc1C(=O)O"  # 아스피린
```

Boltz-2는 추가로 **친화도 (binding affinity) 예측** 까지 제공.

### AlphaFold 3 — 어디서 어떻게

| 옵션 | 가능 여부 |
|---|---|
| 공식 웹서버 ([AlphaFold Server](https://alphafoldserver.com)) | 비상업, 하루 약 20개 |
| 코드 (deepmind/alphafold3) | 가중치 별도 신청, 비상업 |
| Boltz / Chai-1 | AF3와 거의 동급 결과, 오픈소스 |

실용적으로 **오픈소스가 필요하면 Boltz**, **검증용 SOTA가 필요하면 AF3 서버**.

---

## Part 2 — 신뢰도 지표 (반드시 봐야 함)

예측 구조는 **항상 신뢰도와 함께** 봐야 합니다.

### pLDDT (per-residue confidence)

| pLDDT | 의미 |
|---|---|
| > 90 | 매우 신뢰, side chain까지 정확 가능성 |
| 70 - 90 | 신뢰 (backbone), side chain은 주의 |
| 50 - 70 | 낮음, 도메인 구조만 신뢰 |
| < 50 | 매우 낮음, **거의 무작위** 또는 disordered (IDR) 신호 |

PDB의 B-factor 컬럼에 들어감. PyMOL에서 색칠:
```
spectrum b, blue_white_red, minimum=50, maximum=90
```

### PAE (Predicted Aligned Error)

residue i를 정렬했을 때 residue j의 위치 오차 (Å).
**도메인 간 상대 위치 신뢰도** 를 보여줌.

- 대각선 근처 PAE 낮음 = 도메인 내부 잘 맞음
- 도메인 간 블록 PAE 낮음 = 도메인 간 상대 배치 신뢰
- 도메인 간 PAE 높으면 → flexible linker, 상대 배치 안 믿음

### ipTM (interface pTM) — 복합체

복합체 예측에서 **인터페이스** 정확도.

| ipTM | 해석 |
|---|---|
| > 0.8 | 매우 신뢰 (실험과 일치할 가능성 높음) |
| 0.6 - 0.8 | 그럴듯, 추가 검증 권장 |
| < 0.6 | 인터페이스 신뢰 안 됨 (또는 둘이 정말 안 붙음) |

### pTM, ranking_score

전체 구조 신뢰도. AF3 / Boltz는 `ranking_score` 로 모델들 자동 정렬.

{: .important }
> "예측 = 답" 이 아닙니다. **pLDDT, PAE, ipTM 셋을 항상 같이 보고**, 신뢰도 낮은
> 영역은 disordered 또는 모델 한계로 해석.

---

## Part 3 — Inverse: 구조 → 시퀀스

"이 backbone을 안정하게 folding할 시퀀스는?"

### ProteinMPNN — 사실상 표준

```bash
git clone https://github.com/dauparas/ProteinMPNN
cd ProteinMPNN

python protein_mpnn_run.py \
   --pdb_path my_backbone.pdb \
   --out_folder out/ \
   --num_seq_per_target 8 \
   --sampling_temp "0.1" \
   --seed 37
```

특징:
- 백본 + (선택적) side chain context 입력
- 다양한 sampling temperature로 시퀀스 다양성 조절
- **실험 성공률이 매우 높음** (de novo 단백질 디자인의 표준 컴포넌트)

### ESM-IF (Inverse Folding)

Meta의 inverse folding 모델. ProteinMPNN과 비슷한 성능, transformer 백본.

```python
import esm
model, alphabet = esm.pretrained.esm_if1_gvp4_t16_142M_UR50()
# coords (L,3,3) atom coordinates → sequence prediction
```

---

## Part 4 — 단백질 디자인

### De novo 디자인 파이프라인

```
[1] RFdiffusion: 원하는 모티프 / 결합 부위 → 백본 생성
        ↓
[2] ProteinMPNN: 백본 → 시퀀스 8개
        ↓
[3] AlphaFold 2 / ESMFold: 시퀀스 → 다시 구조 예측
        ↓
[4] 자기일관성 (self-consistency) 체크:
    예측 구조가 [1] 백본과 RMSD < 2Å ?
        ↓
[5] 통과 후보 → 합성 + 실험 검증
```

### RFdiffusion — backbone generation

```bash
# Baker lab RFdiffusion
./scripts/run_inference.py \
   inference.output_prefix=out/binder \
   inference.input_pdb=target.pdb \
   "contigmap.contigs=[A1-100/0 50-100]" \
   ppi.hotspot_res="[A30,A33,A37]" \
   inference.num_designs=50
```

특정 hotspot 잔기에 결합하는 새 단백질 백본 50개 생성.

### ESM-3 — 멀티트랙 디자인

ESM-3는 **시퀀스 + 구조 + 기능 토큰** 을 한 모델에서 다룸. 한 트랙 masking →
다른 트랙 예측 가능.

```python
from esm.sdk import client
from esm.sdk.api import ESMProtein, GenerationConfig

m = client(model="esm3-large-2024-03")
prot = ESMProtein(sequence="_" * 100)         # 100aa de novo
prot = m.generate(prot, GenerationConfig(track="sequence", num_steps=50))
prot = m.generate(prot, GenerationConfig(track="structure", num_steps=50))
```

### Chroma — diffusion on backbone + sequence

Generate Biomedicines의 모델. 조건부 생성이 강함 (대칭, 도메인 수 등).

---

## Part 5 — Docking (단백질-리간드)

### 옵션 비교

| 도구 | 형태 | 강점 | 약점 |
|---|---|---|---|
| **AutoDock Vina** | 전통 search | 무료, 표준 | 도메인 외 약함 |
| **DiffDock** | Diffusion | 빠름, blind docking | 친화도 없음 |
| **Boltz-2** | AF3 변형 | 구조 + 친화도 동시 | 무거움 |
| **AlphaFold 3** | 통합 | SOTA 정확도 | 라이선스 제한 |
| **RFAA** | 통합 | 오픈소스 AF3 대안 | 학습곡선 |

### DiffDock 예

```bash
git clone https://github.com/gcorso/DiffDock
python -m inference \
   --protein_path my_prot.pdb \
   --ligand_description "CC(=O)Oc1ccccc1C(=O)O" \
   --out_dir results/ \
   --inference_steps 20 --samples_per_complex 10
```

10개 pose가 confidence score와 함께 나옴.

### 친화도 예측

도킹 pose만으로는 **얼마나 잘 붙는지** 모름. 친화도가 필요하면:
- **Boltz-2** (구조 + Kd 동시)
- **DeepPurpose** (시퀀스+SMILES → IC50)
- **GNINA** (Vina의 CNN scoring 확장)

---

## Part 6 — 함정

### 1. Disordered region을 예측이라고 믿음
pLDDT < 50 인 긴 영역은 **disordered (IDR)** 신호일 가능성. 그 좌표는 거의 무의미.
[CAID benchmark](https://caid.idpcentral.org) 같은 IDR 예측기로 따로 확인.

### 2. Multimer stoichiometry 모름
"이 단백질은 사실 호모5량체" → 단량체로만 예측하면 활성 자리 없음.
**Uniprot / BioGRID** 에서 quaternary 정보 확인 후 stoichiometry 지정.

### 3. Hallucination on novel folds
AF2/AF3는 학습 분포 안에서 강함. **완전히 새로운 fold** (de novo design)
또는 viral / 합성 시퀀스에서는 자신만만하게 틀린 답을 줄 수 있음. pLDDT가
높아도 실험 검증 전엔 믿지 말기.

### 4. MSA가 얕음 (< 30 sequences)
호몰로그가 거의 없는 시퀀스 (orphan, viral, designed) → AF2는 신뢰 떨어짐.
대안: ESMFold, OmegaFold (single-seq).

### 5. 도메인 간 상대 위치
긴 단백질에서 도메인 사이가 flexible linker로 연결되면 AF2가 **임의의 한 conformation**
을 줌. PAE를 봐서 "도메인 간 상대 위치는 안 믿음" 으로 해석.

### 6. Ligand SMILES 입력 실수
도킹할 때 protonation state, tautomer가 결과 좌우. RDKit으로 표준화:

```python
from rdkit import Chem
m = Chem.MolFromSmiles("CC(=O)Oc1ccccc1C(=O)O")
m = Chem.AddHs(m)
canonical = Chem.MolToSmiles(m)
```

### 7. 인공물 (signal peptide, His-tag)
시그널 펩타이드, 친화도 태그가 시퀀스 앞뒤에 붙어 있으면 모델이 그것까지 fold.
**성숙 단백질 시퀀스** 만 입력하는 것이 일반적.

### 8. 정확도 ≠ 기능
구조가 맞아도 **activity** 와는 다른 문제. 예측 구조로 catalytic mechanism 추정
은 추가 검증 필요.

---

## 평가 / 벤치마크

| 벤치마크 | 대상 |
|---|---|
| **CASP15 / 16** | 격년 단백질 구조 예측 community 평가 |
| **CAMEO** | continuous (매주) 평가 |
| **CAPRI** | 복합체 docking |
| **PoseBusters** | docking 물리적 타당성 |

자기 모델/파이프라인 평가하려면 위 셋 결과와 비교가 표준.

---

## 실용 워크플로우 — "내 단백질의 구조를 보고 싶다"

```
[0] 시퀀스 얻기 (UniProt, NCBI)
   ↓
[1] AlphaFold DB ([AFDB](https://alphafold.ebi.ac.uk)) 조회
   → 이미 있으면 끝. 2.3억+ 단백질 사전 계산됨.
   ↓
[2] 없다? PDB BLAST → 비슷한 실험 구조 확인
   ↓
[3] 직접 예측:
     단량체  → ColabFold (AF2) 또는 ESMFold
     복합체  → Boltz-1 또는 AF3 server
   ↓
[4] pLDDT/PAE/ipTM 평가
   ↓
[5] PyMOL/ChimeraX로 시각화, 활성 부위 / 표면 분석
```

### AlphaFold DB가 1순위인 이유
- 인간 단백질 거의 100% 커버
- 2.3억+ 단백질 (UniProt 거의 전체)
- 무료, 즉시
- 직접 돌리는 거랑 같은 모델

---

## 다음

- 데이터: [Sequence (DNA/RNA/Protein)](../data/sequence)
- 모델 카탈로그: [Bio AI 모델 카탈로그](../../for-bio/bio-ai-models)
- 작업: [Expression analysis](expression-analysis) · [Clinical](clinical)
