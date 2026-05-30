# AGENTS.md

This file provides guidance to the AI agent when working with code in this repository.

## Project Overview

**占城大师 (Hex Takeover)** — Single-file HTML game (MuleRun format) with hex-map tile revealing, resource management, barracks, auto-combat, and AI opponent. Zero dependencies, all CSS/JS inline in one `index.html`.

Deadline: 2026-06-22. Branch from `docs/PRD.md` and `CONTEXT.md`.

**Current status: v2 rewrite in progress.** v2 完全取代 v1（不保留 reveal 翻格子机制和三角克制）。完整 v2 领域词汇见 `CONTEXT.md`，机制调研见 `docs/research/gameplay-v2.md`。

## Key Decisions

- **Delivery:** Single `index.html`, all resources inlined, open in browser to play
- **Target form factor:** 竖屏移动端，固定逻辑分辨率 540×960 + letterbox 等比缩放；全 HUD 画进 canvas；单击输入。详见 `docs/adr/0002-portrait-mobile-ui.md`
- **Matchup:** Single-player vs AI (PvE)
- **Rendering:** Canvas 2D, 10Hz fixed timestep via `requestAnimationFrame` + accumulator
- **State:** Single `GameState` object, immutable updates
- **Hex layout:** Pointy-top hexes, odd-r offset convention (odd rows shifted right by half a hex, matches RedBlob Games)。`axialToPixel`/`pixelToAxial` 用真正的 odd-r offset 像素映射（矩形网格不剪切）；`drawHex` 用 pointy-top 形状（`60*i-30`，平边在左右）
- **Map:** 9 columns × 12 rows = 108 cells, no lake row, seed=42 (deterministic). HQ: AI r=1, player r=10（均在含 6 邻居的行，开局必出 6 建筑）
- **Testing:** Embedded `console.assert` IIFE tests at bottom of index.html (count evolves with v2 slices) + manual browser verification

## v2 Rewrite Strategy

v2 是"保留骨架、换内脏"的中等规模重构：Canvas 渲染、HexGrid 坐标数学、10Hz 游戏循环、GameState 单例保留；其余模块基本重写。

| v1 模块 | v2 目标 | 工作量 |
|---------|---------|-------|
| `HexGrid` | 9×12 odd-r offset 网格（无湖水） | 小 |
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

**开发方式：垂直切片迭代**，5 个切片，每个切片保持"游戏能跑起来"的状态（详见 `docs/design/`）。

## Module Architecture (v1 历史快照 · 已废弃)

> ⚠️ **下表是 v1 的模块/行号/机制，已不反映 v2 代码**（v2 无 `TileReveal`、无 `MAP_COLS=5`、无步骑弓克制链）。
> v2 模块对应见上方「v2 Rewrite Strategy」表，v2 关键数值见下方「v2 Core Numbers」表。保留此表仅供对照 v1→v2 演进，**勿据此定位 v2 代码**。

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

完整数值见 `docs/research/gameplay-v2.md` → "数值建议"。以下是开发期最常查的常量：

| 类别 | 值 |
|------|---|
| 地图 | 9 列 × 12 行 = 108 格（无湖水，odd-r offset 矩形网格） |
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
| 人口上限 | 每方 30（开发期可调，参考图为 100；只数移动单位，建筑不计入） |
| 满人口行为 | 所有产兵源 CD 暂停，名额释放后恢复；双方独立，AI 同样受限 |

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
- **HUD 必须在 renderGrid 之后画：** `renderGrid` 用 `ctx.fillRect(0,0,canvas.width,canvas.height)` 清整块画布。`renderTopBar`/`renderBottomBar`/`renderNotification`/覆盖层若在 renderGrid 之前画会被清掉。**render() 顺序固定为：grid → units → wave → death → topBar → bottomBar → notification → 覆盖层**。
- **TerritorySystem.owners 开局必须登记已归属格：** `initGameState` 设 `tile.owner` 但不自动写 `TerritorySystem.owners`，导致 `count(owner)` 开局返回 0（地块数 HUD 显示 0/0）。**修复：generateOpening 后遍历 tiles，把有 owner 的格子写进 `TerritorySystem.owners`**。注意 `TerritorySystem.init()` 仍清空 owners（测试依赖），登记发生在 initGameState 而非 init。
- **canvas 点击坐标必须做 letterbox 缩放补偿：** 画布逻辑尺寸 540×960 但 CSS 缩放后显示尺寸不同。点击事件给的是 CSS 像素，必须 `(clientX-rect.left) * canvas.width/rect.width` 换算到逻辑坐标，否则命中全偏。见 `toCanvasCoords`。
- **v2 TerritorySystem.claim 现在同时设置 tile.owner + flashTicks：** slice 9 起 `claim(hexId, owner, tiles, rng)` 在 `tiles` 存在时会设 `tile.owner = owner` 和 `tile.flashTicks = 3`（领地变更闪白过渡）。旧代码假设 claim 不改 tile.owner 已失效。**修复：若测试只传空对象 `{}` 作为 tiles，`tiles[hexId]` 是 undefined，安全跳过；但生产代码总会传完整 tiles 字典**。
- **v2 resolveUnitAttack 金阶 mode 必须直接用 unit.mode：** 若先按距离算 `mode = dist > 1.2 ? 'ranged' : 'melee'` 再用 `if (gold) mode = unit.mode...` 覆盖，AOE 判断（`mode === 'ranged'`）会读错值。金阶分支**必须直接** `mode = unit.mode === 'melee' ? 'melee' : 'ranged'`，非金阶才走距离判断。
- **v2 测试 IIFE 污染全局 gameState：** 每个测试的 `resetAll()` 调用各系统 `.init()` 清空全局状态（如 `BarracksSystem.init()` 清空 timers，把 `initGameState()` 注册的兵营全删掉，游戏循环永不吐兵）。**修复：所有测试 IIFE 执行完后 `gameState = initGameState()` 重新初始化**。
- **v2 HQ 摧毁必须绕过 RuinSystem + 触发 VictorySystem：** `CombatSystem._destroyBuilding` 对所有 hp≤0 的建筑调用 `RuinSystem.createRuin`，但 HQ 摧毁意味着游戏立即结束，不需要废墟生命周期（且废墟格会破坏波浪翻色逻辑）。**修复：在 `_destroyBuilding` 顶部加 `if (tile.building.type === 'hq') { tile.building = null; return }` 早返回；在 `_cleanup` 中识别 `isHq` 并调用 `VictorySystem.startWave(attacker, hexId, tiles)` 而非 `TerritorySystem.claim`**。
- **v2 AIBehavior.tick 必须传 inner action 而非 wrapper 给 executeAction：** `pendingAction = { action, executeAt }` 是 wrapper，`executeAction` 期望的是 inner action `{ type, tile, cost }`。若误写 `this.executeAction(this._pendingAction, ...)`，内部 `const { tile } = action` 解构出 undefined，所有后续 tile 操作静默失败（early-return），AI 永远不动。**修复：`this.executeAction(this._pendingAction.action, tiles, rng)`**。
- **v2 SeededRNG.init(seed) 必须返回 `this` 支持链式：** 测试/模块里常用 `rng = SeededRNG.init(seed)` 捕获实例。若 init 不返回 this，`rng` 是 undefined，噪声分支 `rng.nextFloat()` 崩溃。**修复：init 末尾 `return this`**。注意：SeededRNG 是单例，多 `init(seed)` 调用都返回同一对象——若需并行多 RNG 流（如 AI 独立噪声），必须自己构造独立实例而非复用 SeededRNG。
- **v2 AI 打分必须难度感知（normal vs hard 用不同公式）：** 升级打分若用单一公式（如金矿 base 30 / 箭塔 base 15+5·enemy），普通档在前线压力下仍会算出箭塔 > 金矿，破坏"普通档只升金矿"的 AC。**修复：普通档箭塔用低权重（8+2·enemy），困难档用高权重（15+5·enemy）；金矿 base 足够高（40 cap 20）使回本期分数稳定**。打分函数里 `if (this._difficulty === 'hard')` 分支决定公式。
- **v2 VictorySystem.startWave 立即设 gameOver，但波浪动画仍能播放：** `startWave` 内 `this.gameOver = { winner, reason }` 立即执行，但 `tick()` 先检查 `if (this.wave)` 再检查 `if (this.gameOver)`，波浪存在时跳过 gameOver 短路。**注意：不要在 wave 活跃期额外检查 gameOver 来阻止波浪推进**。波浪在 WAVE_TICKS 后自清，此后 gameOver 才真正阻止后续 tick。
- **v2 Node 提取 IIFE 测试必须用括号深度计数器定位结尾：** 从 HTML 提取 `(function testXxx(){...})()` 时，字符串搜索 `})()` 不可靠（注释或字符串里可能出现同样字符）。**修复：从左括号开始计数深度，depth=0 且下一字符是 `)` 时定位结尾；写临时 .js 文件执行，避免模板字符串与测试代码反引号冲突**。

## Core Numbers

Refer to `docs/PRD.md` → "核心数值" section for all balance values. Quick reference in `CONTEXT.md`.

## Out of Scope

Card collection/upgrades, multi-faction, unit merge/star, real-time PvP, audio/BGM. See `docs/PRD.md` → "Out of Scope".

> ⚠️ **"mobile adaptation" 已移出 Out of Scope（2026-05-29）**：竖屏移动端现为主交付形态，见 Key Decisions 与 `docs/adr/0002-portrait-mobile-ui.md`。
