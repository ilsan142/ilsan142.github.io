# Bioinformatics × AI Engineers

AI를 잘 쓰고 싶은 **생명정보학 전공자** 와, 생명정보학에 발을 들이려는 **AI 엔지니어**
양쪽을 위한 한국어 양방향 가이드.

## 사이트 구조

```
/
├── index.md                 # 랜딩 (두 트랙 진입점)
├── for-bio/                 # Bio 전공자 → AI 트랙
│   ├── coding-agents.md     # Claude Code · Cursor · Codex 입문
│   ├── bio-ai-models.md     # AlphaFold · ESM · Boltz · Evo · scGPT ...
│   ├── agents-mcp.md        # Agent · MCP · Harness 아키텍처
│   └── workflows.md         # 실전 사용 패턴
├── for-ai/                  # AI 엔지니어 → Bio 트랙
│   ├── mental-model.md      # 생물학의 사고 구조 (DNA → RNA → 단백질 → 표현형)
│   ├── data/                # 데이터 종류별 (omics 트리)
│   │   ├── genomics.md
│   │   ├── transcriptomics.md
│   │   ├── proteomics.md
│   │   ├── single-cell.md
│   │   ├── structural.md
│   │   └── others.md
│   └── tasks/               # 분석 작업별
│       ├── sequence-analysis.md
│       ├── variant-calling.md
│       ├── expression-analysis.md
│       ├── structure-prediction.md
│       └── clinical.md
├── bridge/                  # 두 분야 통합 케이스 스터디
│   ├── case-rnaseq.md       # Claude Code로 RNA-seq 파이프라인
│   └── case-protein-design.md
└── omc-guide/               # 별도 가이드 (oh-my-claudecode)
    └── index.html
```

## 로컬 미리보기

```bash
# Ruby + bundler 필요
bundle install
bundle exec jekyll serve
# → http://localhost:4000
```

## 배포

GitHub User Page (`ilsan142.github.io`) 로 배포 — push 하면 자동 빌드.

- URL: <https://ilsan142.github.io>
- Source: **Settings → Pages → Source: Deploy from a branch (main / root)**
  또는 Actions 빌드 — `remote_theme` 사용 중이므로 둘 다 가능.

## 기여

콘텐츠 제안이나 오류 제보는 Issues/PR 환영.
콘텐츠 라이선스는 CC BY-SA 4.0, 사이트 빌드 설정은 MIT.
