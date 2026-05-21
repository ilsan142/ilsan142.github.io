---
title: Case B. Inverse Folding 설계 루프
parent: Bridge 케이스 스터디
nav_order: 2
---

# Case B. Inverse Folding 단백질 설계 루프
{: .no_toc }

AI 엔지니어가 RFdiffusion · ProteinMPNN · AlphaFold 3 를 한 자동 루프로 묶기.
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
| 대상 독자 | AI/ML 엔지니어 · PyTorch 능숙 · 단백질 입문 |
| 목표 | PD-L1 (PDB `5C3T`) 의 PD-1 결합 부위에 결합하는 새 mini-protein (~70 aa) 설계 |
| 입력 | PD-L1 구조 + hot-spot residue 리스트 |
| 출력 | 합성 후보 시퀀스 10~50개 + 각 시퀀스의 in-silico 신뢰 지표 |
| 전통 방식 소요 | 수개월 (Rosetta hand-tuning) |
| 에이전트 방식 (in-silico) 소요 | GPU 24~72시간 (배치 사이즈에 의존) |
| 핵심 도구 체인 | `RFdiffusion` → `ProteinMPNN` → `ESMFold` → `AlphaFold 3` |
| Wet-lab 사이클 | 2~4주 (gene synthesis + expression + binding assay) |

{: .important }
> 이 케이스는 [for-bio/bio-ai-models](../for-bio/bio-ai-models) 에서 카탈로그로
> 다룬 모델들을 **하나의 자동 루프로 묶는 코드 수준의 케이스 스터디** 입니다.

---

## 1. 문제 정의

### 시나리오

당신은 ML 엔지니어로 신생 biotech 회사에 입사했습니다. 첫 프로젝트:

> "PD-L1 의 PD-1 결합 부위에 결합하는 mini-binder (50~80 aa) 를 in silico 로 100개
> 디자인해서, 합성 가치가 높은 후보 20개를 골라 wet-lab 팀에 넘겨 줘.
> 4주 안에. wet-lab 데이터가 돌아오면 다음 라운드에 반영."

당신이 마주한 현실:
- 단백질 디자인은 전통적으로 Rosetta + 수작업 + 박사 4년
- 2023년 이후 RFdiffusion + ProteinMPNN + AlphaFold 3 조합이 **사람보다 잘함**
  ([Watson et al. 2023](https://www.nature.com/articles/s41586-023-06415-8))
- 도구는 다 있지만 **한 루프로 묶어 자동 평가** 하는 코드는 본인이 짜야 함
- wet-lab 결과는 batch 로 돌아옴 — 반응형 active learning 루프 설계 필요

이 케이스는 **그 루프의 실제 구현** 입니다.

### 디자인 목표 (스펙)

```python
spec = {
    "target_pdb": "5C3T",                # PD-L1 / PD-1 복합체
    "target_chain": "A",                 # PD-L1
    "binder_chain": "B",                 # 우리가 디자인할 사슬 (PD-1 대체)
    "hotspot_residues": [54, 56, 66, 68, 121, 123, 124, 126],  # PD-L1 측 hot-spot
    "binder_length_range": (50, 80),
    "n_backbones": 200,                  # RFdiffusion 으로 생성할 백본 개수
    "n_seqs_per_backbone": 8,            # ProteinMPNN 시퀀스 다양성
    "filters": {
        "plddt_min": 80,                 # ESMFold/AF3 신뢰도
        "pae_interaction_max": 10,       # 인터페이스 PAE
        "iptm_min": 0.7,                 # AF3 인터페이스 신뢰도
        "rmsd_to_design_max": 2.0,       # self-consistency
    },
    "compute": {
        "rfdiffusion_gpu_hours": 24,
        "esmfold_gpu_hours": 4,
        "alphafold3_gpu_hours": 24,
    },
}
```

---

## 2. 데이터 · 도구

### 입력

| 자료 | 출처 | 용도 |
|---|---|---|
| PDB `5C3T` | RCSB | 타겟 + 결합 부위 정의 |
| Hot-spot residue 리스트 | 도메인 전문가 / 문헌 | RFdiffusion conditioning |
| (선택) BindingDB, SAbDab | DB | 음성 대조군 시퀀스 |

### 도구 체인

| 모델 | 역할 | 라이선스 | GPU |
|---|---|---|---|
| **RFdiffusion** | 백본 (3D coord) de novo 생성, hot-spot conditioning | BSD | A100 1장 |
| **ProteinMPNN** | 백본 → 아미노산 시퀀스 (inverse folding) | MIT | A100 1장 |
| **ESMFold** | 빠른 1차 fold 검증 (MSA 불필요) | MIT | A100 1장 |
| **AlphaFold 3** | 복합체 fold + iPTM 검증 (final pass) | 비상업 / Boltz-1 대체 | A100/H100 |
| **Boltz-1 / Boltz-2** | AF3 오픈 대체, 친화도 추정 | MIT | A100 1장 |
| **PyRosetta** | 인터페이스 에너지 (`InterfaceAnalyzer`), 자동 mutation | 학술 무료 | CPU |

코드 흐름:

```
RFdiffusion              (target+hotspot → backbone.pdb)
        ↓
ProteinMPNN              (backbone → 8 sequences/backbone)
        ↓
ESMFold (fast filter)    (sequence → predicted structure, RMSD vs design)
        ↓                ── 통과한 후보만
AlphaFold 3 (complex)    (target+binder → iPTM, PAE_interaction)
        ↓
PyRosetta (refine)       (interface energy, hbond, sap_score)
        ↓
Ranking + selection      (top-20 → wet-lab)
        ↓
Wet-lab (BLI/SPR)        ── 결과를 다시 학습 데이터로
        ↓
Active learning round 2
```

### 출력 구조

```
runs/round1/
├── backbones/                 # RFdiffusion 출력 200 PDB
├── sequences/                 # ProteinMPNN .fasta + jsonl (score 포함)
├── esmfold/                   # 1차 fold 결과 PDB + plddt
├── af3/                       # AF3 complex 결과 + 신뢰 지표
├── pyrosetta/                 # 인터페이스 에너지 표
├── candidates.csv             # 모든 후보 + 모든 점수
├── top20.csv                  # 합성용 최종 후보
├── synthesis_order.tsv        # vendor 에 보낼 시퀀스 표
└── REPORT.md
```

---

## 3. AI 없이 — 전통적 합리적 디자인

2020년 이전의 정통 방식. 박사후 연구원 한 명이 1년에 binder 1~3개를 만들었습니다.

### 단계

1. **타겟 분석** — PyMOL 로 PD-L1 결합 부위를 봄. PD-1 의 결합 표면 잔기를 손으로
   추출. 잔기마다 sidechain orientation, hydrogen bond, hydrophobic patch 를 표시.

2. **Scaffold 라이브러리** — Baker lab 의 `top7`, `EHEE`, `4HB` 등의 작은 스캐폴드
   라이브러리를 PyMOL/Rosetta 로 타겟 위에 docking. RosettaDock 으로 수천 회 시도.

3. **인터페이스 디자인** — 가장 가까운 스캐폴드 위에서 잔기들을 Rosetta `fixbb`,
   `RosettaScripts` 로 swap. 매번 에너지 함수 가중치를 tuning. xml 파일 한 줄 바꿔서
   3일짜리 job 다시.

4. **수작업 검증** — Ramachandran plot, clash 점검, hydrogen bond 거리 수동 측정.
   PyMOL 에서 한 잔기씩 보면서 "이 sidechain 이 가능한 rotamer 가 있나?".

5. **선별** — 50~100 개 후보를 만들면 박사후가 직감으로 5~10개를 골라 합성.

6. **합성·실험** — 6주~3개월. 대부분 expression 안 됨 / soluble 안 됨 / binding 안 됨.

### 한 줄짜리 Rosetta 명령의 모습

```bash
# fixbb design — 한 백본 위에서 amino acid 재배치
rosetta_scripts.linuxgccrelease \
    -s scaffold_docked.pdb \
    -parser:protocol design.xml \
    -nstruct 1000 \
    -ex1 -ex2 \
    -extra_res_fa ligand.params \
    -corrections::beta_nov16 true \
    -out:prefix design_run1_
```

`design.xml` 은 보통 200~500 줄. 잔기마다 어떤 amino acid 가 허용되는지, 어떤
scorefunction 을 쓰는지, 어떤 mover 가 어떤 순서로 도는지를 명시. 이 XML 을
짜는 것 자체가 박사 수준 기술이었습니다.

### 왜 느렸나

- **백본이 고정** — 스캐폴드 라이브러리 의존
- **시퀀스 탐색 공간이 작음** — Rosetta energy 가 SOTA 아님
- **검증이 사람 눈** — Ramachandran, clash 를 PyMOL 에서
- **한 라운드가 길음** — 합성 6주 + 분석 2주 → 피드백 루프 폐쇄가 안 됨

---

## 4. AI와 함께 — 자동 디자인 루프

2024~2026년 표준이 된 조합. **백본부터 새로 만듭니다** (RFdiffusion). 시퀀스는
ProteinMPNN 이 1~5초만에 8개씩 뱉습니다. 검증은 AF3/Boltz/ESMFold 가 자동.

### 4.1. 환경 — Claude Code 에게 한 번에

```text
A100 한 장 있는 워크스테이션에 단백질 디자인 루프 환경 구성해 줘.

- RFdiffusion (https://github.com/RosettaCommons/RFdiffusion) — conda env `rfdiff`, SE(3)-transformer 빌드까지
- ProteinMPNN (https://github.com/dauparas/ProteinMPNN) — 같은 env 안에서 사용 가능하게
- ESMFold (facebook/esmfold_v1, transformers) — env `esmfold`, fp16 활성화
- AlphaFold 3 (DeepMind 공개 가중치) 또는 Boltz-1 (오픈) — env `af3` / `boltz`
- PyRosetta — 별도 env `rosetta`

Python 3.11. CUDA 12.4 가정. 각 env 의 sanity check (소형 PDB 1개로 끝-단부터 끝까지)
하나씩 돌려서 결과 보고. 의존성 충돌은 conda env 분리로 해결.
```

{: .tip }
> 이 단계는 **omc autopilot** 또는 **omc ralph** 가 적합. 환경 빌드는 retry 가 잦은
> 단계이고 망가져도 env 만 다시 만들면 됨.

### 4.2. RFdiffusion — 백본 생성

핵심 명령 (에이전트에게 위임할 때 같은 코드를 짜게 함):

```bash
# RFdiffusion: PD-L1 (chain A) 의 hotspot 에 결합하는 70 residue binder 생성
cd RFdiffusion
./scripts/run_inference.py \
    inference.output_prefix=runs/round1/backbones/design \
    inference.input_pdb=5C3T_clean.pdb \
    contigmap.contigs=['A1-150/0 70-70'] \
    ppi.hotspot_res=[A54,A56,A66,A68,A121,A123,A124,A126] \
    inference.num_designs=200 \
    denoiser.noise_scale_ca=0 \
    denoiser.noise_scale_frame=0
```

해석:
- `contigmap.contigs=['A1-150/0 70-70']` — chain A (PD-L1) 1-150 잔기 고정, 그 뒤
  새 사슬 (binder) 70 잔기 de novo 생성
- `ppi.hotspot_res=[...]` — 백본이 이 잔기들 근처에 위치하도록 guidance
- `noise_scale=0` — 결정론적 (실험적; 기본값 1 도 가능)

### 4.3. ProteinMPNN — 시퀀스 회귀

```bash
# 각 backbone PDB 에 대해 8개 시퀀스
for pdb in runs/round1/backbones/*.pdb; do
  python ProteinMPNN/protein_mpnn_run.py \
    --pdb_path $pdb \
    --pdb_path_chains "B" \
    --fixed_chains "A" \
    --num_seq_per_target 8 \
    --sampling_temp "0.1" \
    --batch_size 1 \
    --out_folder runs/round1/sequences/$(basename $pdb .pdb)/
done
```

- `--fixed_chains "A"` — PD-L1 시퀀스는 그대로
- `--sampling_temp 0.1` — 낮을수록 보수적 (recovery 높지만 다양성 낮음)
- 같은 백본에 다른 temp 를 섞어 다양성 확보 (0.1, 0.2, 0.5)

### 4.4. ESMFold — 빠른 1차 필터

ESMFold 는 MSA 가 필요 없어 **5~30초 / 시퀀스** 로 빠릅니다. 200 backbone × 8 seq = 1,600
시퀀스를 6~10시간에 검증. AF3 의 1/20 시간.

```python
# scripts/esmfold_screen.py
import torch
from transformers import EsmForProteinFolding, AutoTokenizer
import json, pathlib

tok = AutoTokenizer.from_pretrained("facebook/esmfold_v1")
model = EsmForProteinFolding.from_pretrained("facebook/esmfold_v1").cuda().half()
model.esm = model.esm.half()

records = []
for fasta in pathlib.Path("runs/round1/sequences").rglob("*.fa"):
    for header, seq in read_fasta(fasta):
        toks = tok([seq], return_tensors="pt", add_special_tokens=False).to("cuda")
        with torch.no_grad():
            out = model(toks["input_ids"])
        plddt = out["plddt"].mean().item()
        pdb_str = model.output_to_pdb(out)[0]
        # 디자인 백본과의 RMSD (DockQ 또는 TMalign)
        rmsd = compute_rmsd(pdb_str, fasta.parent.parent /
                            "backbones" / f"{fasta.stem}.pdb")
        records.append({"seq_id": header, "plddt": plddt,
                        "rmsd_to_design": rmsd, "seq": seq})
        pathlib.Path(f"runs/round1/esmfold/{header}.pdb").write_text(pdb_str)

json.dump(records, open("runs/round1/esmfold/summary.json", "w"))
```

**필터**: `plddt > 80 AND rmsd_to_design < 2.0` 통과만 다음 단계로.
일반적으로 1,600 → 200~400 개로 줄어듭니다.

### 4.5. AlphaFold 3 — 복합체 검증

AF3 (또는 Boltz-1) 는 **타겟 + 디자인 binder 를 함께 fold** 해서 인터페이스 신뢰도를
줍니다. ESMFold 의 단일 사슬 검증과 다름 — 실제로 결합 형태로 잡히는지 확인.

```python
# scripts/af3_complex.py — 의사 코드
import boltz   # 오픈소스 AF3 대체

for cand in load_filtered_candidates("runs/round1/esmfold/summary.json"):
    complex_input = {
        "target_chain": load_seq("5C3T", "A"),
        "binder_chain": cand["seq"],
    }
    pred = boltz.predict_complex(complex_input,
                                 num_recycles=10,
                                 num_samples=5)
    # 핵심 지표:
    metrics = {
        "iptm": pred.iptm,                          # 인터페이스 신뢰도
        "ptm": pred.ptm,
        "pae_interaction": pred.pae_interface,      # 낮을수록 좋음
        "plddt_binder": pred.plddt[binder_residues].mean(),
        "rmsd_to_designed_backbone": align_rmsd(pred.binder_pdb,
                                                cand["design_pdb"]),
    }
    save_metrics(cand["seq_id"], metrics)
```

**필터**: `iptm > 0.7 AND pae_interaction < 10 AND rmsd_to_designed_backbone < 2.0`.
이게 **self-consistency** 의 핵심 — 디자인했던 백본을 AF3 가 독립적으로 재현하면
신뢰 가능.

### 4.6. 자동화 루프 — Claude Code 한 프롬프트

위 모든 걸 한 줄로 시키는 프롬프트:

```text
디자인 루프 실행 스크립트 `scripts/run_round.py` 작성하고 round1 돌려 줘.

[입력]
- target: 5C3T (PD-L1, chain A)
- hotspot: [A54, A56, A66, A68, A121, A123, A124, A126]
- binder length: 70
- N_backbones: 200, seqs/backbone: 8

[단계]
1. RFdiffusion (env `rfdiff`) — 200 backbone → runs/round1/backbones/
2. ProteinMPNN (env `rfdiff`) — 각 backbone × 8 seq, temp [0.1, 0.2, 0.5] 섞기
3. ESMFold (env `esmfold`) — 모든 시퀀스 fold, plddt + rmsd_to_design
   * filter1: plddt > 80 AND rmsd_to_design < 2.0
4. AlphaFold 3 (또는 Boltz-1, env `af3`) — filter1 통과 시퀀스만 complex fold
   * filter2: iptm > 0.7 AND pae_interaction < 10 AND rmsd_to_designed_backbone < 2.0
5. PyRosetta (env `rosetta`) — filter2 통과 시퀀스의 인터페이스 에너지
   (`InterfaceAnalyzerMover`), sap_score 계산
6. candidates.csv (모든 후보 + 모든 점수) 작성
7. 다음 기준으로 top20.csv 추출:
   - filter2 통과
   - dG_separated < -25 (Rosetta REU)
   - sap_score < 0
   - 시퀀스 다양성: 두 후보 간 sequence identity < 70%
     (top20 안에서 클러스터링 후 cluster당 1개)
8. synthesis_order.tsv 작성 (vendor 가 합성할 수 있게 — His-tag 추가, codon optimization 제안)
9. REPORT.md — 각 필터에서 몇 개가 살아남았는지, 그림 (plddt 분포, iptm 분포, top20 logo plot)

[규칙]
- 각 단계 결과를 disk 에 저장 (재시도 시 캐시 사용)
- A100 한 장 가정, RFdiffusion 24h / ESMFold 6h / AF3 24h 예상
- 단계 실패 시 멈추고 보고
- 모든 시퀀스에 random ID (`r1_b042_s3`) 부여, wet-lab 추적용
```

### 4.7. omc 모드 매핑

| omc 모드 | 이 케이스에서의 용도 |
|---|---|
| `omc autopilot` | 환경 구성 + 첫 라운드 풀 실행 |
| `omc team` (planner/executor/verifier/fixer) | 각 단계가 독립 + 검증 필요한 다단 파이프라인에 적합 |
| `omc ralph` | AF3 가 OOM 으로 중단되는 경우 배치 사이즈를 줄여가며 재시도 |
| `omc ultrawork` | 같은 spec 으로 hotspot 만 다른 5 variant 를 병렬 라운드 |
| 단일 Claude Code | top20 선정 시 인간이 보면서 대화하는 단계 |

병렬 실행 예 — hotspot 조합을 4가지로 변주:

```bash
omc ulw \
  "hotspot v1: [A54,A56,A66,A68]" \
  "hotspot v2: [A54,A56,A121,A123]" \
  "hotspot v3: [A66,A68,A124,A126]" \
  "hotspot v4: [A54,A56,A66,A68,A121,A123,A124,A126]  (현재 기본)"
```

각 변종이 별도 워크트리에서 실행. 24~72시간 후 4개 round1 결과를 비교.

### 4.8. Wet-lab 피드백을 코드에 다시 끼우기

Round 1 의 top20 을 vendor 에 합성 의뢰. 2~4주 뒤 wet-lab 데이터가 돌아옵니다.

전형적으로 다음 형태:

```csv
seq_id,expressed,soluble,bli_kd_nm,sec_monomer_pct
r1_b042_s3,yes,yes,12.4,92
r1_b042_s5,yes,no,,
r1_b078_s2,no,,,
r1_b105_s1,yes,yes,>1000,45
...
```

이 표를 Round 2 에 어떻게 끼우나?

**전략 1 — 분류기 학습**:

```python
# 어떤 in silico 지표가 wet-lab 성공을 가장 잘 예측하는가?
import pandas as pd, xgboost as xgb
from sklearn.model_selection import cross_val_score

df = pd.read_csv("runs/round1/candidates_with_wetlab.csv")
features = ["plddt", "iptm", "pae_interaction", "dG_separated",
            "sap_score", "rmsd_to_designed_backbone"]
y = (df["bli_kd_nm"].fillna(1e6) < 100).astype(int)   # 100 nM 이하면 성공
X = df[features]

clf = xgb.XGBClassifier(n_estimators=200, max_depth=4)
print(cross_val_score(clf, X, y, cv=5).mean())
clf.fit(X, y)
# 다음 라운드는 이 분류기 점수로도 후보 순위 매김
```

**전략 2 — Filter threshold 보정**:

```text
> round1 의 wet-lab 결과 보니까 iptm > 0.7 통과한 후보 중 절반이 expression 실패였어.
> 실패한 후보들의 공통점을 분석해서 round2 의 필터에 추가 조건을 제안해 줘.
> (특히 sap_score, charge distribution, transmembrane prediction)
```

에이전트 분석 결과 예:
- 실패한 후보의 80% 가 sap_score > 5 (표면 hydrophobic 잔기 과다)
- 실패한 후보 평균 isoelectric point > 10
- → round2 필터: `sap_score < 3 AND 6 < pI < 9`

**전략 3 — Active learning**:

Round 1 의 successful binder 시퀀스를 ProteinMPNN 의 `score_only` 모드로 점수화 →
높은 점수 영역의 시퀀스를 round 2 에서 더 sampling. 또는 ESM-2 임베딩 공간에서
nearest neighbor 로 round 2 후보 추천.

### 4.9. Round 2 프롬프트

```text
Round 1 결과 분석해서 round2 spec 짜 줘.

[입력]
- runs/round1/candidates.csv (전체)
- runs/round1/wetlab_results.csv (20개 결과)

[해야 할 일]
1. wet-lab 결과로 분류기 학습 (성공 = kd < 100nM AND expressed AND soluble)
2. round1 의 필터 임계치를 wet-lab 통계로 재조정. 새 임계치 제안.
3. 성공한 binder 들의 시퀀스 motif 분석 (MEME 또는 logo).
   - round2 의 ProteinMPNN sampling 에 partial sequence design 으로 끼우는 게 가능한지 검토.
4. round2 spec.yaml 작성 (round1 의 변경 사항을 diff 형태로)
5. round2 실행 — 같은 파이프라인 + 새 필터 + (가능하면) 성공 motif 시드

먼저 분석 결과를 보여주고, round2 실행은 내 확인 받은 다음에.
```

이렇게 **사람의 확인 게이트** 를 한 곳에 둡니다 — round 간 spec 변경. 그 외 모든
실행은 에이전트가.

---

## 5. 실패 모드와 회복 패턴

### 5.1. Self-consistency failure

**증상**: ESMFold 의 plddt 는 90+ 인데, AF3 complex 에서 iptm < 0.3 (인터페이스 잘못).

**원인 후보**:
- ESMFold 가 단일 사슬은 잘 fold 하지만 **target 과의 상호작용** 은 보지 않음
- RFdiffusion 백본이 물리적으로는 fold 가능하지만 hot-spot 과 정렬 안 됨
- ProteinMPNN 시퀀스가 백본 fold 는 잘 잡지만 **인터페이스 sidechain** 이 약함

**회복**:
```text
> Self-consistency 실패 후보들 (esm_plddt > 90 AND af3_iptm < 0.3) 의 패턴을 분석해 줘.
> 1) RFdiffusion 백본의 hot-spot residue 와의 평균 거리
> 2) ProteinMPNN 의 시퀀스가 인터페이스 위치에 hydrophilic 인지 hydrophobic 인지
> 3) 디자인 백본 vs AF3 complex 의 binder 위치 RMSD
> 결과에 따라 RFdiffusion 의 hotspot guidance scale 을 올리거나, ProteinMPNN 의
> interface residue 에 partial design 제약 추가 제안.
```

방지: round 1 에서 **무조건 AF3 까지 통과한 것만 합성**. ESMFold 만 보고 합성하지 말 것.

### 5.2. Hallucinated folds

**증상**: 디자인 시퀀스를 BLAST 하면 알려진 단백질과 매우 유사 (>90% identity).

**원인**: ProteinMPNN 의 학습 데이터에 가까운 영역으로 회귀했고, 사실은 자연
단백질을 약간 변형한 것에 불과.

**회복**:
```text
> top20 후보의 시퀀스를 BLAST nr (또는 UniRef90) 로 검색해서, e-value < 1e-10 인
> hit 있는 후보는 제외하고 top20 재선정. 또는 새로 sampling 할 때 ProteinMPNN
> temperature 를 0.5 이상으로 올리고 partial design 비율을 낮춰.
```

방지: **합성 전 BLAST 단계를 파이프라인에 필수로** 끼움.

{: .warning }
> 단순 유사 단백질은 IP/특허 문제를 일으킬 수 있습니다. **de novo 인지 확인** 은
> 단순 검증이 아니라 비즈니스 요구사항.

### 5.3. ProteinMPNN sequence diversity 부족

**증상**: 200 backbone × 8 seq = 1,600 시퀀스 중 의미 있는 diversity 가 없음
(top 20 의 평균 pairwise identity > 85%).

**원인**: temperature 0.1 만 사용 → 매번 같은 잔기 선택.

**회복**:
```python
# 다양성 sampling
for temp in [0.1, 0.2, 0.5, 1.0]:
    proteinmpnn_run(backbone, num_seq=2, sampling_temp=temp)
# 또는 결과 시퀀스를 ESM-2 임베딩 공간에서 클러스터링 후 cluster당 1개
```

방지: round 시작 전에 **diversity 메트릭 (mean pairwise identity, embedding spread)**
을 spec 에 명시 — 미달 시 자동 재실행.

### 5.4. Wet-lab expression failure

**증상**: top20 중 50% 가 E. coli 에서 발현 안 됨 / 응집.

**원인 후보**:
- Surface hydrophobic patch (sap_score 높음)
- Disordered region (RFdiffusion 이 만든 loop 가 너무 김)
- Rare codon 다수 (sequence 자체는 좋지만 합성 후 문제)
- Disulfide bond 비대칭

**회복**:
```text
> 발현 실패 후보들의 공통 특징을 분석해 줘:
>  - SAP score (PyRosetta SapScore)
>  - Disorder prediction (IUPred3)
>  - Codon adaptation index (CAI) for E. coli
>  - Cysteine count (홀수면 의심)
>  - Aggregation prediction (AGGRESCAN3D)
> 통계 비교 (실패 vs 성공) 표 + 다음 라운드 필터에 추가할 임계치 제안.
```

방지: 합성 의뢰 전에 **codon optimization 도구 (예: IDT codon optimizer API)** 자동 호출,
**solubility predictor** (Protein-Sol, CamSol) 점수를 필터에 포함.

### 5.5. AlphaFold 3 의 confident-but-wrong

**증상**: AF3 가 iptm 0.85 를 주는 후보가 wet-lab 에서 결합 안 함.

**원인**:
- AF3 는 **결합 가능성** 을 예측하지만 **친화도** 는 약하게만 추정 — kd 100 pM 과 100 nM 을 구별 못 함
- 학습 데이터에 있는 인터페이스와 유사한 백본을 confident 하게 평가

**회복**:
- Boltz-2 의 **친화도 헤드** 를 추가로 사용
- 여러 AF3 sample (num_samples=5~10) 의 분산을 신뢰도 지표로 사용 — 분산 큰 후보는 제외
- 최종 단계에 **MD simulation** (예: GROMACS short equilibration) 으로 인터페이스 안정성 검증

```text
> top50 후보에 대해 Boltz-2 친화도 점수 추가하고, AF3 num_samples=10 으로 다시 돌려서
> iptm 의 standard deviation 도 점수에 반영해 줘 (std < 0.05 일 때만 신뢰).
```

### 5.6. Hotspot 잘못 정의

**증상**: round1 결과의 binder 들이 PD-L1 의 PD-1 결합 부위가 아니라 dimerization 부위에 붙음.

**원인**: hotspot residue 번호를 5C3T 의 chain A 잔기 번호로 줬는데, 실제 PD-1 결합 표면이
다른 영역.

**회복**:
```text
> 5C3T 의 PD-L1 (chain A) 에서 PD-1 (chain B) 가 실제로 접촉하는 잔기들을 다시 추출해 줘
> (PyMOL: `select interface, (chain A within 4.5 of chain B) and polymer`).
> 결과를 RFdiffusion hotspot 인자로 변환하고 round을 다시 시작.
```

방지: 첫 라운드 전에 **타겟 구조의 interface analysis** 단계를 spec 에 명시 — hot-spot 을
손으로 박지 말고 알고리즘적으로 추출.

### 5.7. 일반 회복 원칙

| 원칙 | 적용 |
|---|---|
| **빠른 모델로 거른다** | ESMFold 로 1차, AF3 는 통과한 것만 — 비용 1/20 |
| **여러 신호의 합의** | iptm, plddt, Rosetta dG, sap_score 의 **합** 으로 ranking |
| **합성 전 BLAST** | 자연 단백질 표절 방지 |
| **여러 sample, 분산 활용** | confident-but-wrong 방지 |
| **사람 게이트는 적게, 그러나 결정적인 곳에** | round 간 spec 변경, 합성 의뢰 직전 |

---

## 마치며 — 이 케이스의 본질

**위임 가능한 것** (글루 코드):
- 도구 호출 순서, 환경 설정, 결과 표 정리, 시각화, 분류기 학습, 보고서

**위임 못 하는 것** (단백질 디자인):
- "이 hotspot 이 생물학적으로 의미 있는가" — 문헌 + 도메인
- "이 후보 20개 중 합성 우선순위" — 위험 분산 + 직관
- "wet-lab 실패의 진짜 원인이 in silico 단계에 있나, 발현 단계에 있나" — 실험 디자인
- "이 binder 를 어떤 적응증에 적용할 것인가" — 비즈니스 + 임상

전통 디자인은 박사 4년이 60개 후보를 만들었습니다. AI 루프는 같은 시간에 6,000개를
만들고 60개로 거릅니다. **그 60개를 어디에 쓸지는 여전히 사람이 정합니다.**

---

## 다음

→ [Case A. RNA-seq을 Claude Code로](case-rnaseq) — 반대 방향. 셸 비숙련자가 분석을 위임하는 케이스.

## 참고

- [Bio → AI 트랙 / 2. Bio AI 모델 카탈로그](../for-bio/bio-ai-models) — 여기 등장한 모든 모델의 상세
- [Bio → AI 트랙 / 3. Agent · MCP · Harness](../for-bio/agents-mcp) — omc 모드 정식 소개
- **RFdiffusion** — [Watson et al. 2023, Nature](https://www.nature.com/articles/s41586-023-06415-8) ·
  [GitHub](https://github.com/RosettaCommons/RFdiffusion)
- **ProteinMPNN** — [Dauparas et al. 2022, Science](https://www.science.org/doi/10.1126/science.add2187) ·
  [GitHub](https://github.com/dauparas/ProteinMPNN)
- **AlphaFold 3** — [Abramson et al. 2024, Nature](https://www.nature.com/articles/s41586-024-07487-w) ·
  [server](https://alphafoldserver.com/)
- **Boltz-1 / Boltz-2** — [GitHub](https://github.com/jwohlwend/boltz) (오픈 AF3 대체)
- **ESMFold** — [Lin et al. 2023, Science](https://www.science.org/doi/10.1126/science.ade2574)
- **PyRosetta** — [pyrosetta.org](https://www.pyrosetta.org/)
- [RFdiffusion + ProteinMPNN tutorial (Baker lab)](https://github.com/RosettaCommons/RFdiffusion/blob/main/examples)
