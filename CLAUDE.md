# CLAUDE.md

This file is a pointer. The authoritative project guide is `AGENTS.md` (single source of truth).
The domain glossary lives in `CONTEXT.md`.

**Always read `AGENTS.md` in full before making any change to this repo.**

## Where to look

- **AGENTS.md → Project Overview / Key Decisions** — 项目定位、v2 重写状态、交付约束
- **AGENTS.md → v2 Rewrite Strategy** — v1→v2 模块对应、垂直切片路径
- **AGENTS.md → Module Architecture** — 模块顺序、依赖关系、插入规则
- **AGENTS.md → v2 Core Numbers** — 地图尺寸、HQ、价格档位、混合池权重等关键数值
- **AGENTS.md → Module Insertion Order — CRITICAL** — 新模块必须按依赖插入，否则 `Cannot access before initialization`
- **AGENTS.md → Code Style** — 无分号、2 空格、中文注释、模块模式
- **AGENTS.md → Git Workflow** — `main` 直推、`#N` 前缀 commit message
- **AGENTS.md → Testing Workflow** — 嵌入式 `console.assert` IIFE + Playwright
- **AGENTS.md → Known Pitfalls** — 已知坑（每次动手前先扫一遍）

## Files to read before making changes

- `AGENTS.md` — 必读
- `CONTEXT.md` — 涉及领域词汇时必读
- `doc/design/v2-prd.md` — 涉及新模块设计时按需读
- `doc/research/gameplay-v2.md` — 涉及数值与机制调研时按需读
- `docs/adr/0001-v2-architecture-pivot.md` — 涉及 v2 架构决策时按需读

## Do NOT

- Do **not** treat this file as the source of truth. If it conflicts with `AGENTS.md`, `AGENTS.md` wins.
- Do **not** duplicate content from `AGENTS.md` into this file — update `AGENTS.md` instead.
