# knowledge-base

A centralized, machine-optimized engineering knowledge base designed for **AI coding agents** (Cursor, Claude Code, Google Antigravity, GitHub Copilot), **MCP/RAG pipelines**, and human software engineers.

Every entry in this repository is written in concise, high-signal technical English with explicit `MUST` / `NEVER` invariants, decision matrices, and production verification checklists to maximize retrieval accuracy and minimize token overhead.

## Repository Structure

```text
knowledge-base/
├── README.md                                  # Repository overview & agent integration guide
├── llms.txt                                   # Primary index & trigger map for AI agents
├── _templates/
│   └── kb-entry-template.md                   # Standard schema for authoring new entries
└── knowledge/
    ├── ai-agents-workflows/
    │   └── ai-video-editor-remotion-pipeline.md # Autonomous AI Video Editing (Remotion/FFmpeg/Whisper)
    ├── design-systems/
    │   └── ios-liquid-glass-cross-platform.md   # Cross-platform iOS Liquid Glass (Android & Web)
    ├── graphics-shaders/
    │   └── webgpu-liquid-glass.md               # Production Apple Liquid Glass (WebGPU/WGSL)
    ├── frontend-architecture/                   # (Reserved) React, Next.js, CSS, Design Systems
    └── mobile-expo/                             # (Reserved) Expo, React Native, native builds
```

## Connecting to AI Coding Agents

### 1. Local Workspace / Agent Rules (`.cursorrules`, `CLAUDE.md`, `SKILL.md`)

Instruct your agent to read `llms.txt` first before fetching specific entries:

```markdown
Before implementing complex shaders, architecture patterns, or infrastructure,
consult `knowledge-base/llms.txt` to check if a verified specification exists.
Load only the matching entry and strictly enforce its "Agent Guardrails" section.
```

### 2. Remote Retrieval (MCP / Context7 / RAG)

- Index `llms.txt` as the root routing table.
- Chunk files under `knowledge/` by level-2 (`##`) and level-3 (`###`) headings.
- Preserve YAML frontmatter fields (`id`, `domain`, `stack`, `triggers`) as vector metadata filters.

## Authoring Rules for New Entries

1. **Copy the template:** Start from `_templates/kb-entry-template.md`.
2. **Use technical English:** Avoid narrative filler. Prefer tables, formulas, typed structs, and explicit invariants.
3. **Separate baseline from standard:** Clearly distinguish third-party prototypes or audits from recommended production architecture.
4. **Update `llms.txt`:** Add a single routing line with explicit trigger conditions whenever a new document is committed.
5. **Keep naming consistent:** Use lowercase `kebab-case` for all folders and filenames.

## Quick Push Setup

```bash
git init
git add .
git commit -m "feat(kb): initialize knowledge-base with webgpu liquid glass v2"
git branch -M main
git remote add origin https://github.com/baehaqi-rf/knowledge-base.git
git push -u origin main
```
