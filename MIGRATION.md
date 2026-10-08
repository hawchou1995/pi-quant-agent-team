# 迁移说明与验证记录

## 1. 为什么要做这次迁移（缺口的真实成因）

WorkBuddy 市场插件 `pandaai-ai-quant-research-team` 只把 **6 个 agent 定义**（+ 主理人共 7 个）
迁到了 PI。插件里配套的 **6 个技能整体没落地** —— 它们仍留在两处：

- 原插件包：`<workbuddy market>\plugins\pandaai-ai-quant-research-team\skills\`
- ZCode 技能根：`<user>\.zcode\skills\`

而 PI 只扫描用户级技能根与用户级子代理根，**不扫描 ZCode 技能根**，于是出现：

> 成员定义里宣称能做 DSR / PBO / Haircut / MinTRL，
> 但脚本根本不在 PI 能解析的位置 → 实际跑不了。

最直接的证据是 `overfit-auditor.md` 自带的免责句（迁移时写的）：

> **统计三件套（DSR/PBO/Haircut/MinTRL）PI 无现成技能，由你自带实现** —— 无实现则返回 `BLOCKED_TOOL`

即迁移当时**已经知道**这个缺口，只是没有补。

## 2. 本次动作

```
robocopy <zcode skills>\<6 个技能>  <PI 技能根>\<同名>  /E
```

6 个技能，源/目标文件数逐一对齐：

| 技能 | 文件数 |
|---|---|
| ai-quant-team-runtime | 29 |
| skill-backtest-overfit | 12 |
| skill-factor-mining-pandaai | 20 |
| skill-pandaai-factor-online | 43 |
| skill-report-replication | 21 |
| skill-strategy-tearsheet-report | 11 |

随后修正了唯一的真实缺口引用：`overfit-auditor.md` 第 9 行与"第二段"整段，改为指向
**已实装**的 `skill-backtest-overfit` 脚本，并附上实测输出作为可复跑的自证。

## 3. 验证记录（2026-10-08）

统计类脚本**跑出真实数值**（不是只有 `--help`）：

```
deflated_sharpe.py  → Observed (annualised) Sharpe : 1.649 ／ Deflation benchmark SR0 : 0.0932
pbo_cscv.py         → Pure-noise strategies PBO = 0.584 (expect ~0.5)
                      One real edge PBO = 0.041 (expect low)
haircut.py          → bonferroni : SR 2.00 -> 1.75 (haircut 13%, p 1.8e-08 -> 8.8e-07)
purged_kfold.py     → fold 0: test= 20 train= 75 (purged+embargoed 5 rows)
```

`pbo_cscv.py` 内置零假设校验（纯噪声应 ~0.5、真边缘应低），属**可自证**实现 ——
这是判断"技能是真的能算，还是只是个空壳"的判据。

## 4. 迁移过程中发现并修掉的三个"坑"（留档，避免重复踩）

1. **`v8_factor_cache.pkl` 读不了**：它是 pyarrow 后端 pickle。原生产脚本在本机
   **直接跑不起来**（`ModuleNotFoundError: No module named 'pyarrow'`）。
   解决：`pip install --target D:\Tools\pylibs pyarrow`。
2. **导入顺序**：`D:\Tools\pylibs` 必须在 `import pandas` **之前**入 `sys.path`。
   三序探针实测：`pandas→path→pyarrow` 会抛
   `ImportError: pyarrow>=13.0.0 is required for PyArrow backed StringArray`，
   而 `path→...` 的两种顺序都 OK。
3. **不要用 `try: import pyarrow / except ImportError: pass` 兜底**：静默吞掉后，
   报错会出现在几十行之后的 `read_pickle`，与真因无关，极易误判成"数据坏了"。

## 5. 未完成 / 需注意

- `pandaai-experimenter` 定义里的登录命令指向作者本机路径，换机需自行替换。
- 平台侧（PandaAI CLI / 登录态）不在本仓库范围内，未登录时 `bootstrap.py --status`
  返回 `{"ok": false}` 属预期。
- 技能 frontmatter 的 `name:` 与目录名有三处不一致，**PI 按 `name` 注册**，见 README。
