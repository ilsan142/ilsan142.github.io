---
title: Structural
parent: 데이터 레이어
grand_parent: AI → Bio 트랙
nav_order: 5
---

# Structural — 단백질 3D 구조 데이터
{: .no_toc }

AlphaFold 이후 가장 빠르게 변하고 있는 분야. AI 엔지니어가 들어가기에 가장
"드라이"하고, 모델 카드만 봐도 절반은 따라갈 수 있습니다.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 핵심 개념

**Structural biology** = 단백질 / 핵산 / 복합체의 3차원 좌표를 다루는 분야.

| 표현 수준 | 무엇 | 예 |
|---|---|---|
| **Atom** | 모든 원자 좌표 (x, y, z) | C-alpha, N, C, O, side chain heavy atoms |
| **Residue** | 잔기 단위 (alpha-carbon 만 쓰는 경우 多) | LYS-42 의 CA |
| **Chain** | 폴리펩티드 한 가닥 | chain A, chain B |
| **Assembly** | 생리학적 복합체 (homo/hetero-mer) | hemoglobin = 2α + 2β |

좌표는 옹스트롬 (Å) 단위. 1 Å = 0.1 nm. 단백질 평균 결합 길이 ~1.5 Å.

---

## 데이터 형식

### PDB (`.pdb`) — 옛 표준

고정폭 텍스트. 사람이 직접 읽고 편집할 수 있어 여전히 도구들이 이 형식을 받습니다.

```
HEADER    OXYGEN TRANSPORT                        25-FEB-98   1A3N
ATOM      1  N   VAL A   1      11.751  29.143   7.103  1.00 52.71           N
ATOM      2  CA  VAL A   1      11.875  29.484   8.530  1.00 50.79           C
ATOM      3  C   VAL A   1      12.881  28.541   9.196  1.00 47.81           C
ATOM      4  O   VAL A   1      13.640  27.794   8.547  1.00 47.71           O
ATOM      5  CB  VAL A   1      10.523  29.355   9.252  1.00 53.13           C
...
HETATM 1226  FE  HEM A 142      28.435  23.811   8.420  1.00 31.74          FE
TER
END
```

컬럼 의미 (1-based, 고정폭):

| 컬럼 | 의미 |
|---|---|
| 1-6 | record type (ATOM / HETATM / TER / END) |
| 7-11 | atom serial |
| 13-16 | atom name (`CA`, `N`, `CB`, ...) |
| 17 | alternate location indicator (`A` / `B` / 공백) |
| 18-20 | residue name (3-letter: `VAL`, `LYS`) |
| 22 | chain ID |
| 23-26 | residue sequence number |
| 31-38 | x (Å) |
| 39-46 | y |
| 47-54 | z |
| 55-60 | occupancy (0~1) |
| 61-66 | **B-factor** (temperature factor, Å²) — 결정학적 흔들림. AlphaFold 결과에선 pLDDT (0~100) 가 들어감 |
| 77-78 | element |

`ATOM` = 표준 아미노산 / 핵산. `HETATM` = ligand, ion, water, modified residue.

### mmCIF / PDBx (`.cif`) — 새 표준

PDB는 80컬럼 한계 / 100,000 atom 한계 / chain ID 1글자 한계 등 옛 결정학용 제약이 많아,
2014년부터 **mmCIF** 가 정식 표준. AlphaFold 3 / Boltz / 큰 ribosome 등은 mmCIF만 받음.

```cif
data_1A3N
loop_
_atom_site.group_PDB
_atom_site.id
_atom_site.label_atom_id
_atom_site.label_comp_id
_atom_site.label_asym_id
_atom_site.label_seq_id
_atom_site.Cartn_x
_atom_site.Cartn_y
_atom_site.Cartn_z
_atom_site.occupancy
_atom_site.B_iso_or_equiv
ATOM 1 N   VAL A 1 11.751 29.143  7.103 1.00 52.71
ATOM 2 CA  VAL A 1 11.875 29.484  8.530 1.00 50.79
...
```

key-value 형식이라 파싱이 더 안전합니다. 대규모 복합체 (chain ID 2글자, residue 100k+) 도 표현 가능.

{: .tip }
> 새 코드를 짠다면 **mmCIF 우선**. `gemmi`, `biotite`, `Bio.PDB.MMCIFParser` 모두 잘 지원.

### 기타

| 확장자 | 무엇 |
|---|---|
| `.bcif` | binary CIF, 빠른 로드 (Mol* 웹) |
| `.pdbqt` | docking용 (AutoDock 계열) — partial charge + atom type |
| `.sdf` / `.mol2` | 작은 분자 (ligand) |
| `.npz` (custom) | 모델 학습용 텐서 dump |

---

## 신뢰도 / 품질 지표

AI 예측 구조를 다룰 때 가장 자주 봐야 할 숫자들.

### 실험 구조

| 지표 | 의미 | 좋은 값 |
|---|---|---|
| **Resolution** | X-ray 회절 분해능 (Å) | < 2.0 Å good, < 1.5 Å excellent |
| **R-factor / R-free** | model-data 일치도 | < 0.25 |
| **B-factor** | 원자별 흔들림 (Å²) | < 30 stable, > 50 floppy |

### AI 예측 구조

| 지표 | 의미 | 범위 | 해석 |
|---|---|---|---|
| **pLDDT** | per-residue confidence (AlphaFold) | 0~100 | > 90 매우 정확, 70~90 backbone OK, 50~70 모호, < 50 disordered 가능성 |
| **PAE** (Predicted Aligned Error) | 잔기 i–j 간 상대 위치 오차 (Å) | 0~30 | 낮을수록 도메인 간 배치 신뢰 ↑ |
| **pTM / ipTM** | 전체 / 인터페이스 TM-score 예측 | 0~1 | > 0.8 강함. 복합체는 ipTM > 0.75 보면 갈만함 |
| **ranking_score** | AlphaFold 3 자체 점수 (0.8·ipTM + 0.2·pTM − ...) | 다양 | 모델 5개 중 best 고를 때 |

### 구조 비교

| 지표 | 무엇 | 좋은 값 |
|---|---|---|
| **RMSD** | 원자 좌표 평균 제곱근 차 (Å) | < 2 Å similar fold |
| **TM-score** | 길이 정규화 fold similarity | > 0.5 = same fold, > 0.8 = same family |
| **GDT-TS / LDDT** | local distance test | > 70 good |

{: .warning }
> RMSD는 길이가 다르거나 도메인이 회전한 경우 큰 값으로 보입니다. 두 구조의 fold가 같은지
> 보려면 **TM-score** 또는 **lDDT** 가 더 안전.

---

## 어디서 받나

| 출처 | 무엇 | 규모 (2026) | 접근 |
|---|---|---|---|
| **RCSB PDB** | 실험 결정 구조 (X-ray, NMR, cryo-EM) | ~22만 | https://www.rcsb.org, REST API |
| **PDBe / PDBj** | PDB 미러 (Europe / Japan) | 동일 | EBI |
| **AlphaFold DB** | AlphaFold 2 예측 — UniProt 전체 | ~2억 | https://alphafold.ebi.ac.uk |
| **ESM Atlas** | ESMFold 예측 — metagenome | ~6.17억 | https://esmatlas.com |
| **BFVD** | Big Fantastic Virus Database (viral fold) | ~35만 | |
| **ModelArchive** | 일반 예측 구조 deposit | 수만 | |

```bash
# PDB 다운로드 - REST
curl -O https://files.rcsb.org/download/1A3N.cif

# AlphaFold DB - UniProt accession으로
curl -O https://alphafold.ebi.ac.uk/files/AF-P68871-F1-model_v4.cif

# 대량 - rsync
rsync -av rsync.rcsb.org::ftp_data/structures/divided/mmCIF/ ./pdb_mmcif/
```

---

## AI 모델 — Structure Prediction

2024-2026년의 핵심 모델들. **모두 monomer + complex + ligand 어느 정도 지원** 으로 수렴 중.

| 모델 | 출시 | 입력 | 출력 | 라이선스 | 강점 |
|---|---|---|---|---|---|
| **AlphaFold 2** | 2021 | sequence + MSA | monomer 3D | Apache 2.0 (코드) | 표준, MSA-기반 정확도 |
| **AlphaFold-Multimer** | 2022 | sequences | complex | Apache 2.0 | 복합체 표준 |
| **AlphaFold 3** | 2024 | seq + ligand + nucleic acid + ion | 모든 biomolecule | non-commercial (server) → 2024.11 weights 공개 (CC-BY-NC) | SOTA, ligand 직접 |
| **ESMFold** | 2022 | sequence only | monomer 3D | MIT | MSA 불필요, 10x 빠름 |
| **OmegaFold** | 2022 | sequence only | monomer 3D | Apache 2.0 | MSA 없이 orphan protein |
| **RoseTTAFold All-Atom** | 2024 | seq + ligand + 핵산 | 복합체 | BSD | David Baker lab, ligand 강함 |
| **Boltz-1** | 2024.11 | AF3-호환 입력 | 모든 biomolecule | **MIT** | AF3 수준 + 완전 open |
| **Boltz-2** | 2025 | + affinity | 구조 + 결합 친화도 | MIT | 친화도 동시 예측 |
| **Chai-1** | 2024.09 | AF3-호환 | 모든 biomolecule | Apache 2.0 (가중치 non-commercial) | 빠름, ESM 임베딩 옵션 |
| **Protenix** | 2025 | AF3-호환 | 모든 biomolecule | Apache 2.0 | ByteDance, 학습 코드 공개 |

### 의사결정 트리 — 어떤 걸 쓸까

```
입력이 단일 단백질 시퀀스인가?
├─ YES, MSA 있어/만들 수 있어
│   └─ AlphaFold 2 (vanilla / ColabFold)        ← 가장 검증됨
├─ YES, MSA 만들기 부담 (대규모 / 빠르게)
│   └─ ESMFold                                   ← 한 시퀀스 ~10초
└─ NO, 복합체 / ligand / 핵산 포함
    ├─ 상업 OK 라이선스 필요
    │   ├─ Boltz-1/2          ← MIT, AF3 수준, 권장
    │   └─ Protenix           ← Apache 2.0, 재학습 가능
    ├─ AF3 자체 결과가 필요
    │   └─ AlphaFold Server (web) 또는 AF3 weights (CC-BY-NC)
    └─ ligand 정확도가 최우선
        └─ RoseTTAFold All-Atom 또는 AlphaFold 3
```

{: .note }
> **2026년 실용 default**: 상업 환경 = **Boltz-2**, 학술 비상업 = **AlphaFold 3**, 빠른
> 단일 단백질 = **ESMFold**. ColabFold (AF2-기반) 도 여전히 유효한 baseline.

---

## 코드 예시

### Biopython 으로 PDB / mmCIF 읽기

```python
from Bio.PDB import PDBParser, MMCIFParser

parser = MMCIFParser(QUIET=True)
structure = parser.get_structure("1A3N", "1A3N.cif")

# 계층 순회: Structure → Model → Chain → Residue → Atom
for model in structure:
    for chain in model:
        for residue in chain:
            if residue.id[0] != " ":      # HETATM (water, ligand) 스킵
                continue
            for atom in residue:
                if atom.get_name() == "CA":
                    print(chain.id, residue.id[1], residue.resname,
                          atom.coord, atom.bfactor)
                    break
```

### gemmi — 더 빠르고 표준

```python
import gemmi

st = gemmi.read_structure("1A3N.cif")
model = st[0]
for chain in model:
    polymer = chain.get_polymer()
    seq = gemmi.one_letter_code([r.name for r in polymer])
    print(chain.name, len(seq), seq[:50])
```

### ESMFold inference (한 시퀀스)

```python
import torch
from transformers import AutoTokenizer, EsmForProteinFolding

tok   = AutoTokenizer.from_pretrained("facebook/esmfold_v1")
model = EsmForProteinFolding.from_pretrained("facebook/esmfold_v1",
                                             low_cpu_mem_usage=True)
model = model.cuda().eval()

seq = "MVHLTPEEKSAVTALWGKVNVDEVGGEALGRLLVVYPWTQRFFESFGDLSTPDAVMGNPK"
inputs = tok([seq], return_tensors="pt", add_special_tokens=False).to("cuda")

with torch.no_grad():
    out = model(inputs["input_ids"])

# out.positions       : (1, L, 14, 3)   heavy atom 좌표
# out.plddt           : (1, L, 14)      per-atom pLDDT, residue로 평균
# out.predicted_aligned_error : (1, L, L)

# PDB 문자열로
from transformers.models.esm.openfold_utils.protein import to_pdb, Protein as OFProtein
from transformers.models.esm.openfold_utils.feats import atom14_to_atom37
final_atom_positions = atom14_to_atom37(out.positions[-1], out)
# ... PDB writer
```

### Boltz-1 / Boltz-2 (CLI)

```bash
pip install boltz

# YAML 입력
cat > target.yaml <<'EOF'
version: 1
sequences:
  - protein:
      id: A
      sequence: MVHLTPEEKSAVTALWGK...
  - ligand:
      id: L
      smiles: "CC(=O)OC1=CC=CC=C1C(=O)O"
EOF

boltz predict target.yaml --use_msa_server --recycling_steps 3 --diffusion_samples 5
```

### 좌표 텐서 추출 (학습용)

```python
import gemmi, numpy as np

def coords_ca(cif):
    st = gemmi.read_structure(cif)
    coords, seq = [], []
    for chain in st[0]:
        for res in chain.get_polymer():
            ca = res.find_atom("CA", "*", gemmi.El("C"))
            if ca is None: continue
            coords.append([ca.pos.x, ca.pos.y, ca.pos.z])
            seq.append(gemmi.find_tabulated_residue(res.name).one_letter_code.upper())
    return np.array(coords), "".join(seq)

X, s = coords_ca("AF-P68871-F1-model_v4.cif")
print(X.shape, s)        # (L, 3), "MVHLTPEE..."
```

---

## 시각화 도구

| 도구 | 용도 |
|---|---|
| **PyMOL** | 데스크탑 표준, 논문 그림 |
| **ChimeraX** | UCSF, cryo-EM 강함 |
| **Mol\*** | 웹 표준 (PDB 사이트) |
| **py3Dmol / nglview** | Jupyter 안에서 |
| **ProteinViewer (HuggingFace)** | 가벼운 데모 |

```python
# Jupyter에서 빠르게 보기
import py3Dmol
view = py3Dmol.view(width=500, height=400)
view.addModel(open("1A3N.pdb").read(), "pdb")
view.setStyle({"cartoon": {"color": "spectrum"}})
view.zoomTo(); view.show()
```

---

## 구조 검색 / 비교

| 도구 | 용도 |
|---|---|
| **Foldseek** | 구조 기반 fast search. BLAST의 구조판 |
| **DALI** | 클래식 구조 정렬, 정확 but 느림 |
| **TM-align** | pairwise, TM-score 표준 |
| **US-align** | TM-align 후속, 복합체 / RNA |
| **MASTER** | sub-structure motif 검색 |

```bash
foldseek easy-search query.pdb afdb_proteome result.m8 tmpDir
# --format-output "query,target,pident,alntmscore,evalue"
```

Foldseek 는 **3Di alphabet** (구조를 글자로) 으로 인코딩 → BLAST 속도. 시퀀스 동질성이
사라진 먼 homolog 를 찾을 때 결정적.

---

## 자주 마주치는 함정

### 1. **Missing residues**
실험 구조는 disordered 부분이 흔히 누락. residue number는 1, 2, 5, 6, ... 처럼 점프함.
`structure.get_residues()` 순회 시 sequence index를 그대로 믿지 말고 `residue.id[1]` 확인.

### 2. **Alternate locations (altloc)**
같은 원자가 두 상태 (A/B) 로 모델링된 경우. 기본은 A만 사용:
```python
# Biopython: occupancy 1.0 또는 altloc " "/A 만 keep
for atom in residue.get_atoms():
    if atom.get_altloc() not in (" ", "A"):
        continue
```

### 3. **HETATM 의 정체**
- `HOH` = water (보통 제거)
- `MSE` = selenomethionine (= MET, X-ray phasing용)
- `MG`, `ZN`, `FE` = ion
- 3-letter code 가 처음 보는 거면 **CCD** (Chemical Component Dictionary) 검색

### 4. **Multi-chain / Assembly**
`pdb_id.pdb` 의 chain은 **asymmetric unit** (결정학적 단위). 생물학적 복합체
(biological assembly) 는 다를 수 있음 — `pdb_id.pdb1` 또는 mmCIF의 `_pdbx_struct_assembly`
참조. Hemoglobin (1A3N) 은 asymmetric unit = 한 αβ pair, biological unit = α₂β₂.

### 5. **Chain ID 충돌**
PDB는 1글자, mmCIF는 다글자. `label_asym_id` (mmCIF 내부) vs `auth_asym_id` (논문의 chain
이름) 가 다를 수 있음. 대부분 도구는 `auth_*` 표시.

### 6. **AlphaFold 좌표의 의미**
AF 예측은 단일 conformation의 best guess. **앙상블 / 동적 거동** 정보 아님.
pLDDT < 70 영역은 잘못된 게 아니라 "여기는 본래 disordered" 일 수도.

### 7. **Numbering 차이**
UniProt residue number ≠ PDB residue number 인 경우 흔함 (signal peptide 잘려서 등).
SIFTS 매핑 또는 mmCIF의 `_pdbx_poly_seq_scheme` 참조.

### 8. **Hydrogen**
대부분의 PDB 구조에 H 없음 (분해능 부족). Docking / MD 전에 `reduce`, `pdb2pqr`,
또는 OpenMM 으로 H 추가 필요.

---

## 데이터 / 모델 라이선스 요약

| 자원 | 라이선스 |
|---|---|
| PDB 구조 자체 | CC0 (public domain) |
| AlphaFold DB | CC-BY 4.0 |
| AlphaFold 2 코드 | Apache 2.0 |
| AlphaFold 3 코드 + weights | **CC-BY-NC** (학술만, 2024.11~) |
| ESMFold | MIT |
| **Boltz-1/2** | MIT (가장 자유) |
| Chai-1 | Apache 2.0 코드 / non-commercial weights |
| RoseTTAFold All-Atom | BSD |

{: .warning }
> 상업 프로젝트라면 **Boltz / Protenix / ESMFold / RoseTTAFold** 중에서 고르세요.
> AlphaFold 3 weights 는 학술 전용입니다.

---

## 다음

- 작업: [Structure prediction](../tasks/structure-prediction), [Docking](../tasks/docking)
- 관련 데이터: [Proteomics](proteomics) (시퀀스 측), [기타](others#imaging) (cryo-EM)
