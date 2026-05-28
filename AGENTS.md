# AGENTS.md

This file provides guidance to the AI agent when working with code in this repository.

## Project Overview

**占城大师 (Hex Takeover)** — Single-file HTML game (MuleRun format) with hex-map tile revealing, resource management, barracks, auto-combat, and AI opponent. Zero dependencies, all CSS/JS inline in one `index.html`.

Deadline: 2026-06-22. Branch from `doc/PRD.md` and `CONTEXT.md`.

**Current status: v2 rewrite in progress.** v2 完全取代 v1（不保留 reveal 翻格子机制和三角克制）。完整 v2 领域词汇见 `CONTEXT.md`，机制调研见 `doc/research/gameplay-v2.md`。

## Key Decisions

- **Delivery:** Single `index.html`, all resources inlined, open in browser to play
- **Matchup:** Single-player vs AI (PvE)
- **Rendering:** Canvas 2D, 10Hz fixed timestep via `requestAnimationFrame` + accumulator
- **State:** Single `GameState` object, immutable updates
- **Hex layout:** Flat-top hexes, axial coordinates (q, r), odd rows shifted right by half a hex (odd-r offset convention, matches RedBlob Games)
- **Map:** 9 columns × 13 rows (incl. 1 central lake row at r=6), ~108 playable cells, seed=42 (deterministic)
- **Testing:** Embedded `console.assert` IIFE tests at bottom of index.html (count evolves with v2 slices) + manual browser verification

## v2 Rewrite Strategy

v2 是"保留骨架、换内脏"的中等规模重构：Canvas 渲染、HexGrid 坐标数学、10Hz 游戏循环、GameState 单例保留；其余模块基本重写。

| v1 模块 | v2 目标 | 工作量 |
|---------|---------|-------|
| `HexGrid` | 扩展到 9×12 + 湖水行 | 小 |
| `TileReveal` | 重写为 `TileSystem` (未激活/可开/已开状态机，混合池+二次概率) | 大 |
| `ResourceSystem` | 扩展：多产金源 (HQ/金矿)、多花费 (开地块/升级) | 中 |
| `moveUnits` | 重写：品阶差异化寻敌 (防守反击/越格/AOE) | 大 |
| `checkVictory` | 扩展：HQ 血量 + 3 分钟倒计时 | 中 |
| Combat | 删三角克制 → 改品质压制+射程+AOE | 大 |
| `BarracksSystem` | 重写：每兵营槽绑定兵种按 CD 吐兵 | 大 |
| `AIBehavior` | 重写：评分-选择算法，三档难度 | 大 |
| UI / render | 加倒计时、HQ 血条、价格标签、建筑图标 | 中 |
| Click handling | 加"可开格子高亮 + 点击开地块" | 中 |
| **新增** `OpeningLayout` | 开局 HQ 周围 6 格随机填入建筑 | 小 |
| **新增** `UpgradeSystem` | 金矿/箭塔三档升级 | 中 |
| **新增** `RuinSystem` | 建筑被摧毁 → 废墟 | 小 |
| **新增** `DeathEffect` | 单位死亡粒子特效 | 中 |

**开发方式：垂直切片迭代**，5 个切片，每个切片保持"游戏能跑起来"的状态（详见 `doc/design/`）。

## Module Architecture

| Module | Line | Responsibility |
|--------|------|----------------|
| `createGameLoop()` | 96 | 10Hz fixed timestep |
| `HexGrid` | 126 | Coordinate math, pixel conversion |
| `TileReveal` | 153 | Map generation (seeded RNG), tile types (40/30/20/10%), reveal |
| `ResourceSystem` | 228 | Gold management, reveal cost 10g, mine income 5g/s |
| `moveUnits()` | 267 | Unit movement, tile claiming |
| `checkVictory()` | 328 | Win/lose detection |
| Combat functions | 348 | Counter system (infantry>cavalry>archer), damage calc |
| `initGameState()` + globals | 465 | MAP_COLS=5, MAP_ROWS=7, MAP_SEED=42 |
| `showGameOver`, `restartGame` | 491 | Game over overlay, restart |
| `updateUI()` | 527 | Gold display |
| Rendering | 535 | `drawHex`, `renderGrid`, `renderUnits`, `render` |
| Click handling | 658 | Canvas click, `findHexAt()` |
| Notification | 734 | `showNotification` |
| **Main loop** | 757 | Income → barracks spawn → AI → movement → combat → victory |
| `BarracksSystem` | 808 | Build 40g, spawn 3s, 3 unit types |
| `AIBehavior` | 878 | Reveal 2s, build check 5s, frontline threshold 3 |

Tests are embedded at the bottom (lines ~960-1365): `testHexGrid`, `testTileReveal`, `testResourceSystem`, `testBarracksSystem`, `testAIBehavior`, `testUnitMovement`, `testCombatDamage`, `testCheckVictory`, `testGameLoop`.

> ⚠️ **v1 — 整表已失效（2026-05-28 起）**：上面这张 Module Architecture 表是 v1 的行号与模块名参考，**不要按它定位代码**。v2 slice 1 已删除 TileReveal / ResourceSystem / BarracksSystem / CombatSystem / AIBehavior 等全部 v1 模块。当前 index.html 仅含 `createGameLoop` + `HexGrid` (v2) + 最小 GameState 骨架 + render + 单个 testHexGrid IIFE。新模块命名与职责见上文 "v2 Rewrite Strategy" 表；v2 推荐插入顺序见下文 "Module Insertion Order"。本表将在 v2 全部切片完成后整表重写。

## v2 Core Numbers (Quick Reference)

完整数值见 `doc/research/gameplay-v2.md` → "数值建议"。以下是开发期最常查的常量：

| 类别 | 值 |
|------|---|
| 地图 | 9 列 × 12 行 + 1 行湖水 (~100 格) |
| 开局 | 60 金币，HQ 周围 6 格随机填入建筑 (≥1 金矿概率 70%) |
| 对局时长 | 3:00 倒计时 |
| HQ 血量 | 1000 HP |
| HQ 攻击 | ATK ≈ 100 金箭塔，射程 6.0-7.0，产金 12g/6s |
| 金矿产金 | 10g/6s |
| 价格档位 | 25 / 50 / 100 / 250 金币 |
| 混合池权重 | ? 55% / 头盔 25% / 盾牌 10% / 金属 10% |
| 25 金档 ? 概率 | 空 50% / 绿 25% / 蓝 15% / 紫 7.5% / 金 2.5% |
| 25 金档头盔概率 | 绿 50% / 蓝 35% / 紫 12% / 金 3% |
| 盾牌 | 50 金档 / 100 金档 各 50% |
| 金属 | 无二次随机，直接出金矿 |
| AI 决策周期 | 1.5s |
| AI 升级评估周期 | 5s |
| 单位攻击频率 | 1 次/秒 |

## Module Insertion Order — CRITICAL

This is a single file with no module system. **Definition order matters.** When adding a new module:

1. Insert it **after** `TileSystem` (v2 重写后位置) and **before** `initGameState()`
2. If it depends on other modules, place it after its dependencies
3. Register it in `initGameState()` and the main game loop as needed
4. Never place a module definition after `initGameState()` or the main loop — this causes `Cannot access before initialization`

**v2 推荐模块顺序**（按依赖关系）：
```
HexGrid → TileSystem → ResourceSystem → UnitTypes → BarracksSystem →
CombatSystem → MoveUnits → VictorySystem → AIBehavior →
OpeningLayout → UpgradeSystem → RuinSystem → DeathEffect →
initGameState() → main loop → UI/render → click handling → tests
```

## Code Style

- **No semicolons** — relies on ASI (Automatic Semicolon Insertion)
- **2-space indentation**
- **Chinese comments and UI text** throughout
- **Module pattern:** `const Foo = {}` for stateless modules; bare functions for stateful logic
- **New functions:** always check complete bracket matching — missing closing brace is a known pitfall
- **Tests:** use `(function testXxx() { ... })()` IIFE pattern with `console.assert`, placed at the bottom of the file

## Git Workflow

- **Branch:** develop directly on `main`, no feature branches
- **Commit message:** `#N` prefix to link issues, e.g. `feat: #7 胜负判定 + 游戏流程闭环`
- **PRs:** Not used — push directly to main

## Testing Workflow

- **Quick test:** Open `index.html` in browser, check console for `[test] ... tests passed` output — 每个 IIFE 末尾打印一行
- **Playwright E2E:** Must use `python3 -m http.server` — Playwright MCP blocks `file://` protocol
- **Node-only 跑测试：** 如果只想验证测试断言（不开浏览器），可以从 index.html 抽出 `<script>` 中 HexGrid + testHexGrid IIFE 两段，用 `node -e` eval；需 mock `document` / `canvas` 等浏览器 API 或仅抽纯逻辑模块
- **新测试插入位置：** 所有 `testXxx` IIFE 放在 index.html 末尾 `</script>` 之前，用 `;(function testXxx() { ... })()` 形式（前置 `;` 防御 ASI 拼接）
- **`console.assert` 不抛异常：** Node 与浏览器都是"打印但不中断"。如果想在 CI 里 fail-fast，需临时用 `console.assert = (c, m) => { if (!c) throw new Error(m) }` 覆写

## Known Pitfalls

<!-- Keep updated, max ~8-9 entries -->
- **v2 HexGrid 实际是 pointy-top even-r 偏移：** `axialToPixel` 用 `(1.5 * r)` 公式是 pointy-top；cube distance 必须用 **even-r 偏移 `ax = q - floor(r/2)`**（不是 odd-r 的 `q + floor(r/2)`，AGENTS 曾误写为 odd-r）。`getNeighbors` 偏移表必须匹配 even-r 约定，否则邻居关系不对称。改偏移前**先用 `Math.hypot` 验证像素距离 = √3·size**。
- **v2 湖水格必须可通行（视觉装饰而非移动屏障）：** 六边形邻居只覆盖 ±1 行，若 `getNeighbors` 和 BFS 过滤 r=LAKE_ROW，地图上下两半完全断开，单位无法跨越。**修复：`getNeighbors` 不再过滤湖水格，BFS 不再跳过湖水，渲染保留蓝色视觉带**。
- **v2 computeBFS 返回的 path 包含目标格本身：** path[0] 是第一步，path[last] 是目标格。相邻目标返回 length=1（而非 0）。tick() 中 `path[0]` 是单位的下一步目标格。单位移动到敌方格子时触发 TerritorySystem.claim。
- **v2 测试手动构造 unit 必须带 speed 字段：** MoveUnits.tick 读 `unit.speed` 计算 `moveProgress += speed/10`。遗漏 speed → `undefined/10 = NaN` → moveProgress 永远 NaN → 单位不动。**修复：测试 unit 字面量必须含 `speed: UnitTypes.get(key).speed`**。
- **v2 cube distance vs 像素距离：** `axialToPixel` 返回的坐标经 `Math.hypot` 算出的距离比格子数大 √3 倍（如相邻格像素距离 ≈ 52px，但 cube distance = 1）。射程比较（`range: 3.5`）必须用 cube distance，**不是**像素距离。CombatSystem._hexDist 和 MoveUnits.canAttackFromHere 都用 cube distance。
- **v2 building.attackCd NaN 陷阱：** 建筑 attackCd 初始是 `undefined`，`Math.max(0, undefined - 1) = NaN`，之后 `NaN > 0` 为 false，导致箭塔/HQ 永远不攻击。**修复：递减前加双条件守卫 `if (tile.building.attackCd !== undefined && tile.building.attackCd > 0)`**；新建筑也可主动初始化 `attackCd: 0`。
- **v2 resolveUnitAttack 金阶 mode 必须直接用 unit.mode：** 若先按距离算 `mode = dist > 1.2 ? 'ranged' : 'melee'` 再用 `if (gold) mode = unit.mode...` 覆盖，AOE 判断（`mode === 'ranged'`）会读错值。金阶分支**必须直接** `mode = unit.mode === 'melee' ? 'melee' : 'ranged'`，非金阶才走距离判断。
- **v2 测试 IIFE 污染全局 gameState：** 每个测试的 `resetAll()` 调用各系统 `.init()` 清空全局状态（如 `BarracksSystem.init()` 清空 timers，把 `initGameState()` 注册的兵营全删掉，游戏循环永不吐兵）。**修复：所有测试 IIFE 执行完后 `gameState = initGameState()` 重新初始化**。
- **v2 死亡单位 splice 后数组索引失效：** `CombatSystem._cleanup` 用 `splice` 移除 hp≤0 的单位，导致测试里 `u[2]` 可能指向错误对象或 undefined。**修复：测试中用 `.find(u => u.hexId === '...')` 按 hexId 查找存活单位，不用硬编码下标**。

## Core Numbers

Refer to `doc/PRD.md` → "核心数值" section for all balance values. Quick reference in `CONTEXT.md`.

## Out of Scope

Card collection/upgrades, multi-faction, unit merge/star, real-time PvP, audio/BGM, mobile adaptation. See `doc/PRD.md` → "Out of Scope".
