<!-- Copilot / AI agent 指南：旨在帮助自动化编码代理快速理解仓库约定与可执行路径 -->

# 快速目标

- 目的：让 AI 代理能在不问太多上下文的情况下，理解本仓库的核心数据口径、关键文件位置、常用脚本与可运行示例，以及与此代码库交互时应遵循的项目特定约定。

## 关键上下文（大局）

- 本仓库为“城市创新网络”数据与分析管线。流程为：抓取 → 解析/标准化 → 地理匹配 → 生成边表/节点表 → 网络与空间计量分析。
- 重要目录（请优先查看）：`data/`、`scripts/`、`notebooks/`、`data/processed/city_lookup.csv`（城市口径主表）。主要说明见根目录 `README.md`。

## 项目级别约定（必须遵守）

- 数据字段与口径严格：多值字段用分号（;）分隔（例如 `ipc_all = "H01L;G06N"`）。日期使用 `YYYY-MM-DD`，抓取时间 `YYYY-MM-DD HH:MM:SS`。
- 城市主键：统一使用 `city_code`（字符串）作为地级市的唯一主键，所有合并/聚合操作要以此为主键。
- 去重键：仓库约定使用基于 `{type|relation_subtype|source_entity|target_entity|date}` 的哈希（MD5）作为 `dup_group_key`。修改去重逻辑时请保持兼容字段名。
- 权重字段：事件权重 `weight` 默认 `1`；分摊计数的实现需在记录中保留 `fraction_scheme` 或在脚本注释中说明算式（例如 `1/(n_src * n_tgt)`）。

## 常见文件与可执行路径（示例）

- 快速复现（Python venv）：

  1. 创建并激活虚拟环境，安装依赖（见 `README.md` 的建议）。
  2. 主要脚本路径示例：`scripts/python/01_collect.py`、`scripts/python/02_parse.py`、`scripts/python/03_label_geo.py`、`scripts/python/04_make_edges_raw.py`。
  3. R 分析脚本示例：`scripts/R/10_network_metrics.R`、`scripts/R/20_spatial_econometrics.R`。

## 代理工作注意点（对 AI 的具体指导）

- 修改数据结构或字段名前，先搜索仓库中对该字段的所有引用（包括 README、脚本、notebooks）。保持向后兼容：尽量保留旧字段并新增字段，或同时写转换脚本。
- 若需更改地理映射规则，请优先修改并运行 `data/processed/city_lookup.csv` 的生成或映射脚本，验证 sample 行是否与 README 中示例一致。
- 在处理多值字段时，务必使用分号作为分隔符；写读取/写出代码时保持此约定并在文件头注释中说明。
- 生成或修改 `dup_group_key` 时，请示例化哈希计算并对比现有数据以确认行为一致。

## 编辑/检查清单（提交变更前）

1. 在更改脚本或字段前搜索仓库以定位所有使用点（README + scripts + notebooks）。
2. 运行受影响的脚本的最小复现（例如解析脚本处理一小批 `data/raw/` 示例）并检查输出列头是否包含/保留原字段。  
3. 若更改 data/processed 中的主表（如 city_lookup），提供一个小型转换脚本并在 README 中更新示例。  

## 合并策略（当仓库已有 `.github/copilot-instructions.md` 时）

- 如果检测到已有文件：保留其中“项目级别约定”和“关键路径”段落，合并任何 README 中最新的字段规范更新；不要简单覆盖 README 中列出的口径细节。

---
如果本文件有不清晰之处，请指出具体要点（例如“去重键如何包含金额”或“city_lookup 的编码规则”），我会按需补充示例与小型验证脚本。
