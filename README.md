# 城市创新网络数据与分析脚本（R / Python / Julia / Stata）

含“跨城创新联系事件”（edge-level）与“城市—年份指标”（node-level），提供从 抓取 → 清洗 → 标准化 → 网络构建 → 空间计量分析 的可复现实验管线。

## 目录
```
data/
  raw/
  interim/
  processed/
scripts/
  python/
  R/
  julia/
  stata/
notebooks/
outputs/
docs/
```

## 事件数据最小字段（edge-level）

id：事件唯一 ID（来源缩写 + 原始 ID）

type：事件类型（patent_transfer | co_author | firm_tie）

relation_subtype：关系子类（如 assignment / license / joint_venture / equity_invest / M&A / strategic_alliance / supplier_link / board_interlock）

date（YYYY-MM-DD），year（整数）

source_entity，target_entity

source_entity_type，target_entity_type（firm | university | institute | fund | hospital | gov）

raw_address_source，raw_address_target

source_postcode，target_postcode（可缺）

source_admin_code，target_admin_code（可缺）

source_city_code，target_city_code（地级市代码，统一口径）

地理字段统一聚合到地级市；映射表见 data/processed/city_lookup.csv。

## 采集字段规范（v1.0）

为唯一定位“跨城创新联系事件”，并完成到地级市的精确映射、去重与后续加权，抓取与入库需满足以下字段与口径。

### 一、通用必备字段（所有来源共用）

| 列名                      | 类型       | 说明 / 取值                                                                                                                                  |
| ----------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| id                      | string   | 事件唯一ID（建议=来源系统缩写+原始ID），如 `CNIPA-2020-000123`                                                                                             |
| type                    | enum     | `patent_transfer` \| `co_author` \| `firm_tie`                                                                                           |
| relation\_subtype       | enum     | `assignment` \| `license` \| `joint_venture` \| `equity_invest` \| `M&A` \| `strategic_alliance` \| `supplier_link` \| `board_interlock` |
| date                    | date     | 发生日期，`YYYY-MM-DD`                                                                                                                        |
| year                    | int      | 年份                                                                                                                                       |
| source\_entity          | string   | 源主体法定全称                                                                                                                                  |
| target\_entity          | string   | 目标主体法定全称                                                                                                                                 |
| source\_entity\_type    | enum     | `firm` \| `university` \| `institute` \| `fund` \| `hospital` \| `gov`                                                                   |
| target\_entity\_type    | enum     | 同上                                                                                                                                       |
| raw\_address\_source    | string   | 源地址原文                                                                                                                                    |
| raw\_address\_target    | string   | 目标地址原文                                                                                                                                   |
| source\_postcode        | string?  | 邮编（可缺）                                                                                                                                   |
| target\_postcode        | string?  | 同上                                                                                                                                       |
| source\_admin\_code     | string?  | 县/区级行政代码（可缺）                                                                                                                             |
| target\_admin\_code     | string?  | 同上                                                                                                                                       |
| source\_city\_code      | string?  | 地级市代码（匹配到就填；不确定可留空，保留原文以便后续匹配）                                                                                                           |
| target\_city\_code      | string?  | 同上                                                                                                                                       |
| industry\_code\_system  | enum?    | `CPC` \| `IPC` \| `SIC` \| `NAICS` \| `CSRC` \| `GICS` \| `JCR` \| `CCS` …                                                               |
| industry\_code          | string?  | 对应分类编码，**多值分号分隔**（如 `H01L;G06N`）                                                                                                         |
| amount                  | number?  | 金额（若适用；VC/并购/合同等），**单位= CNY**                                                                                                            |
| currency\_code          | enum?    | 原币种（如 `CNY`/`USD`/`EUR`）                                                                                                                 |
| fx\_rate\_to\_cny       | number?  | 折算汇率（原币→CNY）                                                                                                                             |
| weight                  | number   | 事件权重（默认 `1`；如按作者/发明人/多主体分摊可填小数）                                                                                                          |
| source\_system          | enum     | 数据来源（如 `CNIPA`/`WOS`/`CSCD`/`CNKI`/`EXCHANGE`/`QIXINKU` 等）                                                                               |
| evidence                | string   | 证据定位（URL/公告号/公报编号/DOI/招股书页码等）                                                                                                            |
| evidence\_type          | enum     | `regulatory_filing` \| `exchange_announcement` \| `patent_gazette` \| `journal` \| `news` \| `website`                                   |
| crawl\_time             | datetime | 抓取/录入时间，`YYYY-MM-DD HH:MM:SS`                                                                                                            |
| city\_match\_method     | enum     | `exact_name` \| `postcode` \| `admin_code` \| `fuzzy` \| `manual`                                                                        |
| city\_match\_confidence | number   | 0–1 置信度                                                                                                                                  |
| geocode\_source\_lon    | number?  | 源经度（WGS84）                                                                                                                               |
| geocode\_source\_lat    | number?  | 源纬度（WGS84）                                                                                                                               |
| geocode\_target\_lon    | number?  | 目标经度（WGS84）                                                                                                                              |
| geocode\_target\_lat    | number?  | 目标纬度（WGS84）                                                                                                                              |
| geocode\_method         | enum?    | `text_rule` \| `poi_db` \| `api` \| `manual`                                                                                             |
| dup\_group\_key         | string   | 去重键（如 `{type | relation_subtype | source_entity | target_entity | date}` 的 hash） |
| version                 | int      | 记录版本号（来源变更时递增）                                                                                                                           |
| note                    | string?  | 备注                                                                                                                                       |

## 口径建议

金额统一折算为 CNY；另存原币与汇率（currency_code、fx_rate_to_cny）。

城市口径优先“总部地”（备注可存“注册地”作稳健性）。

weight 若采用“分摊计数”，请在文档写明方案（如 1/(n_src * n_tgt)；合作作者/单位采用等额或调和分摊需注明）。

地理字段统一到地级市；映射表 data/processed/city_lookup.csv 为主键来源。

缺失值一律留空（CSV）或 null（JSON/Parquet），不填 NA/-/0 代替。

### 二、按来源的扩展字段

#### A. 专利转让 / 许可（type=patent_transfer）

| 列名                            | 类型      | 说明                                                   |
| ----------------------------- | ------- | ---------------------------------------------------- |
| patent\_id                    | string  | 专利标识（建议存 `publication_no` 或 `application_no`）        |
| application\_no               | string | 申请号                                                  |
| publication\_no               | string | 公开号                                                  |
| grant\_no                     | string | 授权号                                                  |
| patent\_title                 | string | 专利标题                                                 |
| application\_date             | date   | 申请日                                                  |
| publication\_date             | date   | 公布日                                                  |
| grant\_date                   | date   | 授权日                                                  |
| ipc\_main / ipc\_all          | string | 主/全量 IPC（分号分隔）                                       |
| cpc\_main / cpc\_all          | string | 主/全量 CPC（分号分隔）                                       |
| transfer\_mode                | enum   | `assignment` \| `license` \| `pledge` \| `others`    |
| consideration\_amount         | number | 对价金额（CNY）                                            |
| share\_percent                | number | 许可/股权份额（%）                                           |
| assignee\_from / assignee\_to | string | 转出/转入受让人                                             |
| inventor\_names               | string | 发明人姓名（分号分隔）                                          |
| agent                         | string | 代理机构                                                 |
| legal\_status                 | enum   | `pending` \| `granted` \| `invalidated` \| `expired` |

#### B. 联合论文 / 科研合作（type=co_author）

| 列名                                                    | 类型      | 说明                                                             |
| ----------------------------------------------------- | ------- | -------------------------------------------------------------- |
| paper\_id                                             | string  | `doi` \| `wos_uid` \| `pmid` \| `cnki_id`                      |
| title                                                 | string  | 论文标题                                                           |
| journal                                               | string  | 期刊名                                                            |
| volume / issue / pages                                | string  | 卷/期/页                                                          |
| doc\_type                                             | string  | Article, Review, …                                             |
| publish\_date                                         | date    | 出版日期                                                           |
| affiliation\_source\_full / affiliation\_target\_full | string  | 源/目标机构地址原文                                                     |
| num\_authors / num\_affiliations                      | int     | 作者数/机构数                                                        |
| subject\_category                                     | string  | 学科分类（JCR/CCS/CSCD 等）                                           |
| fraction\_scheme                                      | enum    | `equal` \| `harmonic` \| `first_last_priority` \| `full_count` |
| citation\_count                                       | int?    | 被引次数（可后补）                                                      |

#### C. 企业关系（type=firm_tie）

（覆盖并购、股权投资、合资、战略合作、供需链、董事交叉等）

| 列名                                        | 类型      | 说明                                                                                                                          |
| ----------------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------- |
| deal\_id / agreement\_id                  | string  | 交易/协议编号                                                                                                                     |
| deal\_status                              | enum    | `announced` \| `signed` \| `closed` \| `terminated`                                                                         |
| relation\_subtype                         | enum    | `M&A` \| `equity_invest` \| `joint_venture` \| `strategic_alliance` \| `supplier_link` \| `board_interlock` \| `co_project` |
| invest\_round                             | enum    | `seed` \| `A` \| `B` … \| `pre-IPO`                                                                                         |
| stake\_percent                            | number  | 持股比例（%）                                                                                                                     |
| valuation\_pre / valuation\_post          | number  | 事前/事后估值（CNY）                                                                                                                |
| buyer\_or\_acquirer / seller\_or\_target  | string  | 收购方/被收购方                                                                                                                    |
| product\_or\_service                      | string  | 合作标的/产品                                                                                                                     |
| hs\_code / cicc\_code / gics              | string  | 供需/行业编码                                                                                                                     |
| contract\_amount                          | number  | 合同额（CNY）                                                                                                                    |
| start\_date / end\_date                   | date    | 生效区间                                                                                                                        |
| is\_listed\_source / is\_listed\_target   | int     | 是否上市（0/1）                                                                                                                   |
| stock\_code\_source / stock\_code\_target | string  | 证券代码，如 `600519.SH`                                                                                                          |
| exchange\_source / exchange\_target       | enum    | `SSE` \| `SZSE` \| `HKEX` \| `NASDAQ` \| `NYSE`                                                                             |
| director\_name                            | string  | （若 `board_interlock`）                                                                                                       |
| lead\_investor\_flag                      | int     | 是否领投（0/1）                                                                                                                   |
| syndicate\_size                           | int     | 联合投资家数                                                                                                                      |

### 三、实现与校验建议（精简）

去重键（dup_group_key）：以 {type|relation_subtype|source_entity|target_entity|date} 连接后做 md5；如含金额/证据编号等可加入以增强区分度。

权重（weight）：默认 1；若多源/多目标分摊，按 1/(n_src * n_tgt) 或作者等额分摊；论文也可用调和分摊（按序权重）并记录到 fraction_scheme。

地理匹配：优先 admin_code > postcode > exact_name > fuzzy；置信度（0–1）按规则递减；人工校正记为 manual=1。

金额口径：amount/valuation_* 统一以 CNY 存储；如非 CNY，必须同时给出 currency_code 与 fx_rate_to_cny。

多值字段：用分号 ; 分隔字符串（如 industry_code、ipc_all 等）；库表规范化时建议拆为一对多明细表。

时间格式：date=YYYY-MM-DD；crawl_time=YYYY-MM-DD HH:MM:SS（本地时区或 UTC 需在文档说明）。

## 快速开始

### Python

```bash
# 建议使用虚拟环境
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate
```
R
```r
install.packages(c("tidyverse","data.table","readxl","sf","spdep","igraph","arrow"))
```

Julia

```julia
using Pkg
Pkg.activate(".")
Pkg.instantiate()
```

复现流程（建议）
scripts/python/01_collect.py：抓取/下载 → data/raw/
scripts/python/02_parse.py：解析与标准化 → data/interim/
scripts/python/03_label_geo.py：地址清洗与行政区匹配 → data/processed/
scripts/python/04_make_edges_raw.py：生成 edges_raw.csv
scripts/R/10_network_metrics.R：构图、中心度/社区识别
scripts/R/20_spatial_econometrics.R：空间权重矩阵与空间计量
scripts/julia/abm_demo.jl：主体迁移与网络演化仿真
环境要求
Python 3.10+
R 4.2+
Julia 1.10+
操作系统：Windows / macOS / Linux（UTF-8 编码）
可选：GDAL/GEOS/PROJ（用于 geopandas/pyproj/shapely 等地理依赖）
data/processed/city_lookup.csv 期望列（最小）
用于把原始地址映射到地级市统一口径，供事件边表与节点指标共享主键。
必需列：
city_code：地级市代码（推荐 GB/T 2260 衍生的地级口径或自定义稳定编码，字符串）
city_name：地级市中文名（UTF-8）
province_name：省级名称
可选列（推荐）：
province_code：省级代码（字符串）
admin_code_sample：代表性的区县行政代码示例
lon,lat：地级市质心经纬度（WGS84）
alias：历史/俗名及英文别名（分号分隔）
notes：备注
最小示例（CSV）：

```csv
city_code,city_name,province_name,province_code,lon,lat
110100,北京市,北京市,110000,116.405,39.904
310100,上海市,上海市,310000,121.473,31.230
440100,广州市,广东省,440000,113.264,23.129
440300,深圳市,广东省,440000,114.059,22.543
```

注意：city_code 作为仓库内唯一主键，请在所有表（如 edges_raw.csv、节点年度指标）中统一采用该键进行合并。
许可证
代码：MIT（见 LICENSE）
数据：CC BY 4.0（见 LICENSE-DATA）
