# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**占城大师 (Hex Takeover)** — A single-file HTML game (MuleRun format) implementing hex-map tile revealing, resource management, barracks building, auto-combat with 3-unit counter system, and AI opponent. Zero dependencies, all CSS/JS inline in one `index.html`.

Deadline: 2026-06-22. Branch from `doc/PRD.md` and `CONTEXT.md`.

## Key Decisions

- **Delivery:** Single `index.html`, all resources inlined, open in browser to play
- **Matchup:** Single-player vs AI (PvE)
- **Rendering:** Canvas 2D, 10Hz fixed timestep via `requestAnimationFrame`
- **State:** Single `GameState` object, immutable updates
- **Hex layout:** Flat-top hexes, pointy-top arrangement, axial coordinates (q, r)
- **Map:** 5 columns × 7 rows (~35 cells), bases at both ends
- **Testing:** No test framework — use `console.assert` for embedded logic checks; manual browser verification

## Module Architecture

| Module | Responsibility |
|--------|---------------|
| HexGrid | Hex rendering, coordinate math, click picking |
| TileReveal | Reveal logic, tile content generation |
| ResourceSystem | Gold income/expense, mine production |
| BarracksSystem | Build barracks, unit spawning |
| CombatSystem | Unit movement, engagement, counter system, death |
| TileOwnership | Cell ownership, faction colors |
| AIBehavior | AI decision-making (reveal/build/spawn) |
| GameLoop | 10Hz main loop, win/lose conditions |

## Core Numbers

Refer to `doc/PRD.md` → "核心数值" section for all balance values (gold costs, unit stats, counter multipliers, timings).

## Out of Scope

Card collection/upgrades, multi-faction, unit merge/star, real-time PvP, audio/BGM, mobile adaptation, animation effects. See `doc/PRD.md` → "Out of Scope".

## Working Conventions

- No build tooling — edit `index.html` directly, refresh browser to test
- No linting or formatting tools — keep code clean and readable
- Keep all code in one file; no module bundling
- When adding features, check `doc/PRD.md` first for defined behavior and constraints

## Known Pitfalls

<!-- 由 /work-issues command 自动维护，最多 8 条 -->
- **favicon.ico 404:** 浏览器自动请求 `/favicon.ico` 导致控制台 404。修复：`<link rel="icon" href="data:,">` 内联空图标
- **GameLoop 断言时机:** `console.assert(tickCount > 0)` 在 loop 启动前同步执行，tickCount 尚未更新。修复：用 `setTimeout(..., 200)` 延迟验证
- **Playwright file:// 拦截:** Playwright MCP 阻止 `file://` 协议访问。修复：用 `python3 -m http.server` 提供 HTTP 服务
- **初始化顺序冲突:** `initGameState()` 在 `TileReveal` 定义之前调用，导致 `Cannot access before initialization`。修复：将依赖模块（HexGrid、TileReveal）放在 GameState 初始化之前
