---
title: 2. 전문 Bio AI 모델 카탈로그
parent: Bio → AI 트랙
nav_order: 2
---

# 전문 Bio AI 모델 카탈로그
{: .no_toc }

LLM이 모든 걸 해주지는 않습니다. 특정 생물학 문제에 특화된 **foundation model**
들을 한 표로 정리.
{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>이 페이지의 목차</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 이 페이지의 목적

생물학을 위한 AI 모델은 **5개 큰 묶음** 으로 나뉩니다:

1. **단백질 구조** — 시퀀스 → 3D
2. **단백질 시퀀스** — 시퀀스 임베딩 · 기능 예측
3. **유전체 (DNA)** — 시퀀스 → regulatory / variant effect
4. **단일세포 (transcriptome)** — 세포 단위 발현 표현
5. **분자/약물** — 작은 분자 SMILES → property

각각 어떤 모델이 있고, **언제 어떤 걸 쓰는지** 한눈에 보이는 의사결정표를 제공합니다.

{: .note }
> 모델은 빠르게 갱신됩니다. 이 페이지는 **2026년 5월 기준** 입니다.
> 가장 최신은 각 모델의 GitHub / arXiv 를 직접 확인하세요.

---

## 1. 단백질 구조 예측

시퀀스 → 3D 좌표.

| 모델 | 출시 | 핵심 | 라이선스 | 추천 사용처 |
|---|---|---|---|---|
| **AlphaFold 2** | 2021 | 단량체·복합체 (제한적) | Apache 2.0 | 단량체 기본 |
| **AlphaFold 3** | 2024 | 단백질·DNA·RNA·리간드 복합체 | 비상업적 (서버) | 복합체·도킹 |
| **Boltz-1 / Boltz-2** | 2024 / 2025 | AF3 급 오픈소스 | MIT | 자유로운 로컬 사용 |
| **ESMFold** | 2022 (Meta) | 단일 시퀀스 (MSA 불필요) | MIT | 빠른 스크리닝 |
| **RoseTTAFold All-Atom** | 2024 | 단백질·리간드·핵산 | BSD | AF3 대안 |
| **OmegaFold** | 2022 | 단일 시퀀스 | Apache 2.0 | 고아 단백질 |
| **Chai-1** | 2024 | AF3 오픈 재구현 | 비상업 | 학술 |

### 의사결정 흐름

```
당신의 입력은?
├─ 단량체 단백질, MSA 만들 시간 있음     → AlphaFold 2 (ColabFold)
├─ 단량체, 빠른 결과 필요               → ESMFold
├─ 복합체 (단백질-단백질)                → AlphaFold 3 또는 Boltz-1
├─ 단백질-리간드 도킹                    → AlphaFold 3 · Boltz-2 · RFAA
├─ 단백질-핵산                          → AlphaFold 3 · RFAA
└─ MSA 만들 정보 없는 신규 시퀀스        → ESMFold · OmegaFold
```

### 실용 팁

- **ColabFold** — AlphaFold 2를 무료 Colab에서 돌리는 가장 쉬운 방법.
  생물학자 첫 시도라면 여기서.
- **Boltz-1 로컬** — A100 한 장이면 충분. `pip install boltz`.
- **AlphaFold 3 웹서버** — 하루 ~20개 무료, 비상업적.
- **신뢰도 지표**: **pLDDT > 70** = 신뢰, **PAE 낮음** = 도메인 간 위치 정확.

---

## 2. 단백질 시퀀스 — Language Models

시퀀스 → 임베딩, 기능 예측, 변이 영향.

| 모델 | 파라미터 | 토큰 | 라이선스 | 강점 |
|---|---|---|---|---|
| **ESM-2** | 8M ~ 15B | 아미노산 | MIT | 가장 널리 쓰임 |
| **ESM-3** | 1.4B ~ 98B | multi-track (시퀀스·구조·기능) | 비상업 | 디자인 작업 |
| **ProtT5** | 3B | 아미노산 | Apache | T5 백본, 임베딩 품질 |
| **ProstT5** | 3B | 시퀀스 ↔ 구조 토큰 | Apache | Foldseek 통합 |
| **SaProt** | 35M~650M | 시퀀스 + 구조 토큰 | MIT | 구조 인지 임베딩 |
| **AMPLIFY** | 120M~350M | 효율적 ESM 대안 | MIT | 적은 데이터 |
| **xTrimoPGLM** | 100B | 시퀀스 | 비상업 | 최대 규모 |

### 어디에 쓰나

- **변이 효과 예측 (ΔΔG, fitness)** — ESM-2 zero-shot, ESM-IF, [AlphaMissense](https://github.com/google-deepmind/alphamissense)
- **단백질 디자인** — ESM-3, [ProteinMPNN](https://github.com/dauparas/ProteinMPNN) (구조 → 시퀀스)
- **기능 어노테이션** — ESM-2 임베딩 + classifier
- **상동성 검색 (sequence 너무 멀어 BLAST 실패)** — ProstT5 + Foldseek

### 코드 예 (ESM-2 임베딩)

```python
from transformers import EsmTokenizer, EsmModel
import torch

tok = EsmTokenizer.from_pretrained("facebook/esm2_t33_650M_UR50D")
model = EsmModel.from_pretrained("facebook/esm2_t33_650M_UR50D")

seq = "MVLSPADKTNVKAAWGKVGAHAGEYGAEALERMFLSFPTTKTYFPHFDLSHGSAQVKGHGKKVADALTNAVAHVDDMPNALSALSDLHAHKLRVDPVNFKLLSHCLLVTLAAHLPAEFTPAVHASLDKFLASVSTVLTSKYR"
inputs = tok(seq, return_tensors="pt")
with torch.no_grad():
    emb = model(**inputs).last_hidden_state.mean(dim=1)  # (1, 1280)
```

이 임베딩 위에 분류기를 얹으면 **단백질 위치/기능 분류** 가 거의 즉시 됩니다.

---

## 3. 유전체 (DNA) Foundation Models

DNA 시퀀스 → regulatory effect, variant impact, gene prediction.

| 모델 | 파라미터 | context | 라이선스 | 강점 |
|---|---|---|---|---|
| **Evo** | 7B | 131k bp | Apache 2.0 | 유전체 + RNA + 단백질 통합 |
| **Evo-2** | 7B / 40B | 1M bp | Apache 2.0 | 가장 긴 컨텍스트 |
| **Nucleotide Transformer** | 50M~2.5B | 6k bp | CC BY-NC | InstaDeep, 다종 |
| **DNABERT-2** | 117M | 변동 (BPE) | MIT | 가벼움, 미세조정 쉬움 |
| **HyenaDNA** | 6.5M~6.5M | 1M bp | Apache | long context, 작은 모델 |
| **Caduceus** | 1.9M~7M | 131k bp | Apache | bi-directional, 유전체 SOTA |
| **GENA-LM** | varies | 36k bp | MIT | BigBird 백본 |

### 어디에 쓰나

- **Promoter / enhancer 분류** — DNABERT-2, Nucleotide Transformer
- **Variant pathogenicity** — Evo, [Enformer](https://github.com/google-deepmind/deepmind-research/tree/master/enformer)
- **CRISPR guide off-target** — DNABERT-2 임베딩 + 분류기
- **합성 DNA 디자인** — Evo (장거리 종속성 학습)

{: .warning }
> 모델 출력은 **인간 유전체 (hg38)** 학습 비율이 압도적입니다. 다른 종이면
> "**transfer learning**" 미세조정 필수. 종 정보 없이 zero-shot 신뢰 위험.

---

## 4. 단일세포 (Single-cell) Foundation Models

세포 단위 transcriptome → 세포 타입, 상태, perturbation 응답.

| 모델 | 학습 셀 수 | 핵심 | 라이선스 | 강점 |
|---|---|---|---|---|
| **scGPT** | 33M | Transformer, generative | MIT | 다용도, 가장 널리 인용 |
| **Geneformer** | 30M | BERT 형 | Apache 2.0 | gene ranking 입력 |
| **UCE** (Universal Cell Embedding) | 36M | species-aware | MIT | 종 간 비교 |
| **scFoundation** | 50M | 100M 파라미터 | 비상업 | 큰 규모 |
| **CellPLM** | 9M | cell-level pretraining | MIT | 빠른 미세조정 |

### 어디에 쓰나

- **Cell type annotation** — Geneformer, scGPT (zero-shot or fine-tune)
- **Batch correction** — scGPT 임베딩 + Harmony 대체
- **Perturbation 예측** (어떤 유전자 KO 시 발현 변화) — Geneformer in silico
- **Cross-species transfer** — UCE

### 현장 분위기

이 분야는 **2023~2024년 이후 폭발** 했고, **모델 간 벤치마크 합의가 아직 약합니다**.
실제로 일부 비교 논문은 "단순 PCA가 foundation model 보다 안 떨어진다" 라고 보고했습니다
([Kedzierska et al. 2023](https://doi.org/10.1101/2023.10.16.561085)). 자기 데이터에서
직접 검증하세요.

---

## 5. 분자 / 약물 모델

작은 분자 SMILES → property, generation.

| 모델 | 형태 | 라이선스 | 강점 |
|---|---|---|---|
| **ChemBERTa-2** | RoBERTa | MIT | SMILES 임베딩 |
| **MolFormer** | Transformer | Apache | IBM, large-scale |
| **Uni-Mol** | 3D + transformer | MIT | 분자 conformation |
| **DiffDock** | Diffusion | MIT | 단백질-리간드 도킹 |
| **Boltz-2** | AF3 변형 | MIT | 도킹 + 친화도 |
| **REINVENT 4** | RL | Apache | de novo 분자 디자인 |
| **MOSES / GuacaMol** | 벤치마크 | - | 비교 평가 |

### 어디에 쓰나

- **Docking** — DiffDock, Boltz-2, AlphaFold 3
- **ADMET 예측** — ChemBERTa-2 + classifier, [DeepPurpose](https://github.com/kexinhuang12345/DeepPurpose)
- **De novo 디자인** — REINVENT, GENTRL
- **Retrosynthesis** — [AiZynthFinder](https://github.com/MolecularAI/aizynthfinder)

---

## 의사결정 — "내 문제에 뭐가 맞나"

```
당신의 입력은 무엇?
├─ 단백질 시퀀스
│   ├─ 3D 구조가 필요   → AlphaFold/ESMFold/Boltz
│   ├─ 기능/변이 영향   → ESM-2 임베딩 + 분류기
│   └─ 새 시퀀스 디자인 → ESM-3, ProteinMPNN
├─ DNA 시퀀스
│   ├─ 짧은 (~kb) regulatory → DNABERT-2, Nucleotide Transformer
│   ├─ 긴 (~100kb~Mb)        → Evo / Evo-2, HyenaDNA
│   └─ Variant pathogenicity → AlphaMissense (단백질측), Evo (DNA측)
├─ Count matrix (scRNA-seq)
│   ├─ Cell type 분류        → Geneformer, scGPT
│   ├─ Cross-species          → UCE
│   └─ Perturbation 예측      → Geneformer
├─ 단백질-리간드 복합체
│   └─ → AlphaFold 3, Boltz-2, DiffDock
└─ 작은 분자 SMILES
    ├─ Property              → ChemBERTa-2, MolFormer
    └─ Generation            → REINVENT 4
```

---

## 어디서 모델을 받나

- **Hugging Face** — ESM, DNABERT, ChemBERTa, scGPT 등 대부분
- **GitHub** — AlphaFold (deepmind), Boltz, ProteinMPNN
- **NVIDIA BioNeMo** — GPU 최적화된 통합 플랫폼 (상업)
- **InstaDeep TRIDENT / Nucleotide Transformer** — Hugging Face

---

## 다음

→ **[3. Agent · MCP · Harness 아키텍처](agents-mcp)** — 이 모델들을 **한 명령으로
연쇄 호출** 하는 에이전트 시스템.
