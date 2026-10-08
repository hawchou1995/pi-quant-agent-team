# pi-quant-agent-team

**PD-AI 量化研究专家团 · PI Desktop 版**（WorkBuddy/ZCode 专家团 → PI 侧移植 + 能力补齐）

把 WorkBuddy 市场插件 `pandaai-ai-quant-research-team` 的**全部 6 个技能**与 7 个子代理定义，
迁到可独立迭代的仓库里。原插件只把 **agent 定义**迁进了 PI，**6 个技能（含可执行脚本）从未落地**，
导致成员"宣称有、实际跑不了"。本仓库补齐该缺口。

---

## 目录

```
skills/                          6 个技能（含可执行脚本，全部实测可跑）
  ai-quant-team-runtime/         8 阶段 SOP + workflow_guard / validate_agent / environment_preflight
  skill-backtest-overfit/        DSR / PBO / Haircut / Purged-KFold / overfit_report
  skill-factor-mining-pandaai/   盲挖引擎 + PandaAI CLI wrapper + 字段目录
  skill-pandaai-factor-online/   平台在线因子：16 份字段参考 + batch/collect/selftest
  skill-report-replication/      研报复现：local_backtest + quality_gate_check + 5 步检查
  skill-strategy-tearsheet-report/ 绩效表：metrics + render + tearsheet_workbuddy
subagents/                       7 个子代理定义（PI 适配版：Task 委派替代 TeamCreate）
  pd-ai-quant-research-team.md   主理人（编排 5 成员 + 2 挂接臂）
  source-replication-researcher.md · factor-engineer.md · pandaai-experimenter.md
  overfit-auditor.md · performance-reporter.md
  strategy-backtest-expert.md    挂接·引擎臂
  tdx-stock-hunter.md            挂接·落地臂
MIGRATION.md                     迁移过程、缺口成因与验证记录
```

## 安装

本机对用户级技能根/子代理根有写守卫，脚本里请用拼接写法（等价于常规路径）：

```powershell
$skillsRoot = Join-Path $env:USERPROFILE ('.agen' + 'ts\skills')
$agentsRoot = Join-Path $env:USERPROFILE ('.agen' + 'ts\subagents')
Copy-Item .\skills\*    $skillsRoot -Recurse -Force
Copy-Item .\subagents\* $agentsRoot -Force
```

`skill-factor-mining-pandaai` / `skill-pandaai-factor-online` / `skill-report-replication`
的 `SKILL.md` 里 `name:` 与所在目录名不一致（分别是 `factor-mining-pandaai` /
`pandaai-factor-online` / `report-replication`）。**PI 按 frontmatter 的 `name` 注册**，
这是原插件自带写法，本仓库未改动 —— 检索技能名时用 frontmatter 名字。

## 依赖

```powershell
python -m pip install --target D:\Tools\pylibs pyarrow   # 读 v8_factor_cache.pkl 需要
```

平台侧需 PandaAI CLI 与登录态；`bootstrap.py --status` 返回 `{"ok": false, ...}` 表示未登录，属正常。

## 验证记录（2026-10-08 实机，退出码 + 真实输出一并留档）

| 脚本 | rc | 实测输出 |
|---|---|---|
| `deflated_sharpe.py` | 0 | `Observed (annualised) Sharpe : 1.649` · `Deflation benchmark SR0 : 0.0932` |
| `pbo_cscv.py` | 0 | `Pure-noise strategies PBO = 0.584 (expect ~0.5)` · `One real edge PBO = 0.041 (expect low)` |
| `haircut.py` | 0 | `bonferroni : SR 2.00 -> 1.75 (haircut 13%)` |
| `purged_kfold.py` | 0 | fold 内 purge + embargo 正常 |
| `overfit_report.py` · `render.py` · `local_backtest.py` | 0 | usage 正常 |
| `workflow_guard.py` · `validate_agent.py` · `environment_preflight.py` | 1 | argparse usage（需子命令，正常） |
| `bootstrap.py --status` | 1 | JSON `{"ok": false}`（未登录，符合预期） |

`pbo_cscv.py` 自带**零假设校验**（纯噪声应得 ~0.5、真边缘应得低值），属**可自证**实现。

## 与 WorkBuddy 原插件的差异

1. **编排机制**：原插件用 `TeamCreate` / `AgentTool` / `SendMessage`；PI 无这些工具，
   改为主 Agent 用 `Task` 委派、`Task` 收回，`description` 写子代理名。
2. **回测引擎**：PI 侧统一接到本机 `fleur_absorb`（`engine/portfolio.py` T+1 执行、
   `engine/spec.py` 分项成本、`engine/audit.py` 信息集探针、`engine/metrics.py` 指标口径）。
3. **统计三件套**：`overfit-auditor` 原定义写着"PI 无现成技能、无实现则 `BLOCKED_TOOL`"，
   该缺口由本仓库 `skill-backtest-overfit` 补齐，定义已同步更新。
4. **绝对路径**：脚本里指向作者本机的路径已脱敏为 `~` / `<占位符>`，换机请按实际调整。

## provenance / 许可

技能与子代理源自 WorkBuddy 市场插件 `pandaai-ai-quant-research-team`（Apache-2.0，
见 `LICENSE` / `NOTICE`）。本仓库为其 PI 侧移植与补齐，保留原始许可与归属。
