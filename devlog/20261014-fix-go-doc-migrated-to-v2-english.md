# 20261014 修复 OpenCode Go / Go Plus 图表全空（官方文档迁仓 + 改英文单语 + 模式清单换格式）

## 原始需求

"这个项目今天又出bug了，OpenCode Go和Plus列表加载不出来，排查并修复。"

## 查因结论

这是官方**第四次**把解析打挂，但这次不是改文案，而是**数据源本身迁走了**，三处同时变化，
实测 `parse()` 返回 `items: 0 / patterns: 0`，`index.html` 与 `go-plus-limits.html` 两页图表全空：

| # | 位置 | 旧代码依赖 | 官方改成 | 后果 |
| --- | --- | --- | --- | --- |
| 1 | 数据源路径 | `packages/web/src/content/docs/zh-cn/go.mdx` | `services/www/src/docs/content/console/go.mdx` | 两个 URL 均 **HTTP 404** |
| 2 | 文档语言 | 中文表头 `模型/输入/…/每月限制` | 英文表头 `Model/Input/…/Monthly limit` | `prices=[]` → **整图空白** |
| 3 | token 模式清单 | `- X — 870 个输入 token，…` | `- X — 870 input, 55,000 cached, 200 output tokens per request` | `patterns=0` → 全部无法估算 |

额外发现（**不需要改代码**，但影响后续排查）：

- 仓库目录重组：`packages/*` → `services/*`；文档站点同期切 V2，线上地址由
  `opencode.ai/docs/go/` 变为 `https://opencode.ai/v2/docs/console/go/`（锚点 `#usage-limits` 仍在）。
- 文档改为**英文单语**：仓库里 `zh-cn/go.mdx` 已删除，全仓库只剩一份 `console/go.mdx`（英文）。
- 两张表现在嵌在 `<PlanTabs>` 的 `<div slot="go">` / `<div slot="go-plus">` 缩进块里。
  `parseTableByHeader` 用 `trim()` 后比较，缩进不影响命中；两套表顺序仍是 Go 在前，`occurrence` 参数继续有效。

## 实现小结

只动"数据契约"三处，解析与渲染逻辑未改：

- `SOURCES`：两个 URL 换到 `services/www/src/docs/content/console/go.mdx`（raw + jsdelivr 兜底各一份）。
- `parse()` 表头常量改英文：
  - 价格表 `['Model','Input','Output','Cached Read','Cached Write','Monthly limit']`
  - 官方对照 `['Model','Requests per 5 hours','Requests per week','Requests per month']`
- `parsePatterns()` 正则改为 `- <表达式> — N input, N cached, N output tokens per request`。
- 页脚"数据源"链接（zh/en 各一处）与 `README.md` 指向 `opencode.ai/v2/docs/console/go/#usage-limits`。

`index.html` 与 `go-plus-limits.html` 的上述四处**逐字一致**（后者仅 `PLAN = 1` 不同），保持两文件便于对照同步。
`goat-limits.html` 抓 commandcode.ai，本次不受影响。

## 验证

沿用「从 HTML 里切出真实代码在 node 下执行」的探针 + 真实浏览器端到端：

1. **改前复现**：旧代码跑当日上游 mdx → `items 0 / patterns 0`，中文表头命中 0、英文表头命中 43，确认根因。
2. **改后解析**（两页各跑一遍，结果一致）：43 个价格档位、40 个可估算、patterns 27 条、
   官方对照命中 **43/43**、`monthly` NaN 0 个。隐藏 3 个：
   `LongCat 2.5 Preview Free` / `Step 5 Preview Free` / `Cat Jam Free`（价格 Free、额度 Unlimited）。
3. **取表正确性**：两页逐档比对，43 行模型名与顺序完全一致；40 档 quota 倍数落在
   `2x:6 / 3x:7 / 4x:26 / 8x:1`，**异常 0 条**。抽样 `GLM-5.3` `$15→$120`（989 → 7,916，×8）、
   `GLM-5.3-Flash` 31,579 → 94,737（×3）。
4. **端到端（真实浏览器）**：本地 `http.server` + Chrome headless（带代理）加载两页并 dump DOM —
   均 **40 根柱 / 明细表 43 行 / `#error` 隐藏 / meta 显示"共 43 个价格档位"**，
   全文 NaN 仅出现在注释里（实际数据 0 个 NaN）。
5. **数据源可用性**：raw 与 jsdelivr 两个 URL 均 HTTP 200、大小 43,070 字节、内容一致。

## 与官方的口径差异（非回归，页面 ▲ 提示）

- 官方对照表这次**覆盖了全部 43 档**，且与工具公式高度吻合（`GLM-5.3-Flash` 工具 31,579 vs 官方 31,580）。
- 但官方对照表对多档位模型（`(≤ 200K tokens)` / `(Peak)` 等）只给**汇总值**，工具仍按价格档位拆分估算，
  高档位单价成倍 → 估算约为官方值的 1/2 或 1/3，此类行仍由 ▲ 标记提示。
- `GLM-5.3` 工具 989 vs 官方 1,080 之类的固有差异（官方表与自陈参数不一致）保持原状。

## 使用方式

双击 `index.html` / `go-plus-limits.html`，或线上 `mousebomb.org/opencode-go-limits/`。
本地改完 `git push` 会经 `.githooks/pre-push` 自动 SFTP 部署（`mbtools/deploy.cjs`）。

## TODO（已知边界）

- **脆弱面仍是表头列名 + 模式清单格式**。本次已是官方第四次变更数据契约，且这次连"语言"都变了
  （中文 → 英文单语）。如需彻底免疫，可在 `parseTableByHeader` 失配时加兜底：
  "6 列 且首列含 model/模型"即认定为价格表；模式清单则可在正则尾部放宽为可选语言分组。
  当前按最小改动原则未做，仅记录方案。
- 两页解析代码仍为复制关系（`index.html` / `go-plus-limits.html`），官方再改版需同步改两处。
- 上游若再迁仓（`services/www` 已是从 `packages/web` 迁来），`SOURCES` 里的路径会再次 404；
  上线前可顺手 `curl -I` 两个 URL 自检。
