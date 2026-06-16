# AI-Infra — Agent Guide

This is a **pure markdown documentation** repository (no source code). It provides a curated AI infrastructure landscape and learning path, focused on Kubernetes + cloud-native AI workloads.

Maintainer: pacoxu (personal learning repo, some AI-generated content — verify before use).

## 中文文档 (`zh/`)

`zh/` 目录是完整的英文文档镜像，所有 `.md` 文件已翻译为中文。目录结构与英文完全一致（相同深度），因此相对链接无需修改。根级文件（`zh/README.md` 等）的链接使用 `../` 前缀指向英文原版。

已有中文翻译的博客文章保留在英文目录的 `_zh.md` 文件中，未复制到 `zh/` 目录。

## Repo structure

- `docs/kubernetes/` — Scheduling, GPU, DRA, pod lifecycle
- `docs/inference/` — LLM serving engines, orchestration platforms
- `docs/training/` — Distributed training, ML pipelines
- `docs/observability/` — GPU/inference monitoring
- `agent-infra/` — Agent platforms, runtime benchmarks
- `diagrams/` — All images (SVG/PNG); referenced via relative paths
- `scripts/generate-landscape-svg.js` — Node.js script to generate the landscape SVG
- `project_watchlist.json` — Curated project tracking (included + todo)

## Content rules

- Every `.md` file **must** have YAML front matter: `status`, `maintainer`, `last_updated`, `tags`
- Single source of truth per topic — link to canonical file, do not duplicate content
- External projects: use `<a href="...">`ProjectName`</a>` format with CNCF status
- Images: place in `diagrams/`, reference with relative paths (e.g. `../../diagrams/foo.svg`)
- Cross-reference with relative links (e.g. `./topic.md` or `../other/topic.md`)

## Commands

```bash
# Setup
npm install -g markdownlint-cli markdown-link-check

# Lint all markdown
markdownlint **/*.md

# Check external links (3-5 min for full repo)
find . -name "*.md" -exec markdown-link-check {} \;

# Regenerate landscape SVG
node scripts/generate-landscape-svg.js
# Input: diagrams/ai-infra-landscape.data.json → Output: diagrams/ai-infra-landscape.svg
```

## Validation workflow

1. `markdownlint **/*.md` — fix line length (120 char), trailing spaces, inline HTML
2. `markdown-link-check` on modified files — ignore Status:0 for conference/asset URLs
3. Verify internal links resolve and referenced images exist

**No CI/CD** — all validation is manual.

## Key facts

- Two entrypoints: `README.md` (EN) and `README.zh-CN.md` (ZH)
- `RoadMap.md` and `STRUCTURE.md` describe future work and directory conventions
- `.github/copilot-instructions.md` has the older, detailed Copilot instructions (this file supersedes for OpenCode)
- `.markdownlint.json` configures custom rules (MD013 line_length=120, MD024 siblings_only, MD025/MD029/MD033/MD034/MD036 disabled)
- Blog posts in `docs/blog/` use date-prefixed filenames (`YYYY-MM-DD-topic.md`)
