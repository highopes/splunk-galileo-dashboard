[English](README.md) | [简体中文](README.zh-CN.md)

# Splunk Enterprise 银行模型经济性 Dashboard

本仓库包含一个可直接部署的 Splunk App，用于将 `banking_cn_economics.csv` 快照转换为 **Multi-Agent Banking ChatBot Model Economics** Dashboard Studio 看板。该看板可比较模型质量、耗时、工作流成本、Token 使用量、运行复杂度和 Trace 级证据。

本实现基于 Ubuntu 上的 Splunk Enterprise 9.3.1 构建并完成验证。当前 App 版本为 `1.1.0`。

## 目录

- [仓库结构](#仓库结构)
- [Dashboard 的完整制作过程](#dashboard-的完整制作过程)
- [部署到已有 Splunk Enterprise 系统](#部署到已有-splunk-enterprise-系统)
- [部署验证](#部署验证)
- [使用新 CSV 更新 Dashboard 数据](#使用新-csv-更新-dashboard-数据)
- [Dashboard 使用说明](#dashboard-使用说明)
- [指标定义](#指标定义)
- [维护与故障排查](#维护与故障排查)

## 仓库结构

```text
.
├── banking_cn_economics/
│   └── banking_cn_economics.csv              # 原始数据快照
├── banking_model_economics/                   # 可部署的 Splunk App
│   ├── dashboard_source/
│   │   └── banking_multi_agent_model_economics.dashboard.json
│   ├── local/
│   │   ├── app.conf
│   │   ├── transforms.conf
│   │   └── data/ui/
│   │       ├── nav/default.xml
│   │       └── views/banking_multi_agent_model_economics.xml
│   ├── lookups/
│   │   └── banking_cn_economics.csv
│   └── metadata/local.meta
└── README.md / README.zh-CN.md
```

Splunk 从 `banking_model_economics/` 加载 App。`dashboard_source/` 内的文件是便于阅读和维护的 Dashboard Studio 源文件；实际运行的 Dashboard 定义嵌入在 `local/data/ui/views/banking_multi_agent_model_economics.xml` 的 CDATA 中。

## Dashboard 的完整制作过程

### 1. 检查和验证 CSV

CSV 是使用 UTF-8 编码、LF 行尾的快照文件。每个物理行代表一条根 Trace，稳定复合主键为：

```text
experiment_id + trace_id
```

当前快照包含 12 行、2 个 Experiment、2 个 application model、每个 Experiment 6 条 Trace，以及 12 个唯一 Trace ID。长文本字段中的字面量 `\n` 可在不破坏 CSV 行结构的情况下保留换行信息。

首先用以下搜索验证行数和主键控制值：

```spl
| inputlookup banking_cn_economics.csv
| stats count AS rows
        dc(experiment_id) AS experiments
        dc(application_model) AS models
        sum(comparison_ready) AS comparison_ready_rows
        dc(trace_id) AS unique_traces
```

预期结果：

| rows | experiments | models | comparison_ready_rows | unique_traces |
| ---: | ---: | ---: | ---: | ---: |
| 12 | 2 | 2 | 12 | 12 |

然后验证每个 Experiment 内的唯一性和 6 条 Trace 覆盖：

```spl
| inputlookup banking_cn_economics.csv
| stats count AS rows dc(trace_id) AS unique_traces
        max(experiment_trace_count_reported) AS reported_traces
        by experiment_id experiment_name application_model dataset_version
| eval key_status=if(rows=unique_traces,"OK","DUPLICATE")
| eval six_trace_status=if(rows=6 AND reported_traces=6,"OK","WARNING")
| table experiment_name application_model dataset_version rows unique_traces
        reported_traces key_status six_trace_status
```

不能用零替换缺失值。出现重复主键、核心指标不完整、`comparison_ready=0` 或 Trace 数量异常时，必须保留异常并进行调查。

原始控制总数如下：

| Application model | Dataset version | GT 合格 | 总耗时（秒） | 工作流成本（USD） | Supervisor 汇总 output tokens |
| --- | ---: | ---: | ---: | ---: | ---: |
| `qwen3.7-flash` | 1 | 6/6 | 478.763938 | 0.002528790050 | 16,878 |
| `qwen3.8-max` | 6 | 6/6 | 157.938983 | 0.105030042003 | 4,917 |

Dataset version 不同是源数据中的事实。看板会显示可比性警告，而不会隐藏或人为统一版本差异。

### 2. 创建独立的 Splunk App

Dashboard 被隔离在 App ID 为 `banking_model_economics` 的独立 App 中。`local/app.conf` 将 App 设置为可见且启用：

```ini
[install]
is_configured = 1
state = enabled

[ui]
is_visible = 1
label = Banking Model Economics

[package]
id = banking_model_economics
```

将 lookup、视图、导航和权限放在一个 App 中，便于整体移植，也避免修改 `system/default` 或其他无关 App。

### 3. 将 CSV 打包为 lookup

经过验证的 CSV 在不改变内容的情况下复制到：

```text
banking_model_economics/lookups/banking_cn_economics.csv
```

`local/transforms.conf` 定义基于文件的 lookup：

```ini
[banking_cn_economics]
filename = banking_cn_economics.csv
```

Dashboard 搜索使用 `| inputlookup banking_cn_economics.csv`。由于数据是固定快照，Dashboard 特意不提供全局时间选择器。

### 4. 配置导航和权限

`local/data/ui/nav/default.xml` 将 Dashboard 设置为 App 的默认视图：

```xml
<nav search_view="search" color="#0B1220">
  <view name="banking_multi_agent_model_economics" default="true" />
</nav>
```

`metadata/local.meta` 为 lookup、transform、视图和导航对象授予所有角色读取权限，并为 `admin` 和 `power` 授予写入权限。如果目标组织要求更严格的访问控制，应调整这些角色列表。

### 5. 构建 Dashboard Studio 定义

Dashboard 使用基准宽度为 1440 px 的 Absolute 布局，包含：

- 3 个动态下拉输入：Model、Experiment 和 Scenario。
- 15 个可视化面板。
- 17 个 `ds.search` 数据源。
- 首屏为 Executive 模型对比。
- 中部为 6 轮 Trace 对比和证据。
- 末尾为数据完整性与可比性检查。

所有分析搜索都应用相同的三个 token：

```spl
| search application_model="$tok_model$"
         experiment_name="$tok_experiment$"
         scenario_key="$tok_scenario$"
```

三个下拉框默认值均为 `All` / `*`，可选项从 lookup 动态生成。

数值字段在聚合前通过 `tonumber()` 转换。每个 Experiment 都以 `experiment_id` 作为明确的聚合边界，绝不会将不同 Experiment 的结果静默合并成一次运行。

Executive 图表使用 `chart ... over ... by series_label` 将每个 Experiment 的聚合值转换为模型序列。Trace 图表使用 `T01` 到 `T06` 的 `trace_label`，便于在模型之间比较六个轮次。

便于阅读的定义保存在：

```text
banking_model_economics/dashboard_source/banking_multi_agent_model_economics.dashboard.json
```

同一份 JSON 同时嵌入到：

```text
banking_model_economics/local/data/ui/views/banking_multi_agent_model_economics.xml
```

在源代码仓库中修改 Dashboard 时，必须保持这两个定义同步。Splunk 加载的是 XML 视图，而不是独立的 JSON 文件。

### 6. 加入显示和语义保护

- 普通指标显示两位小数，例如 `20.00`。
- 极小金额保留首个有意义的数字，避免显示为零，例如 `0.000013`。
- 工作流成本与 Judge 成本保持独立。Executive 和逐 Trace 的应用工作流成本图均不包含 Judge 成本。
- `supervisor_output_tokens` 是顶层汇总值，已经包含下游 Agent。
- `supervisor_direct_output_tokens` 仅表示 Supervisor 的直接推理，不能再与汇总值相加。
- 未评分的 Ground Truth 行显示为 `Not scored`，不会计为失败。
- Ground Truth 合格率的分母为已评分 Trace 数，而不是总行数。
- 无错误的行保持中性；只有非零错误或 LLM timeout 才产生运行告警。

### 7. 安装、重启和验证

完成后的 App 目录被复制到 Splunk App 目录，默认位置通常为：

```text
/opt/splunk/etc/apps/banking_model_economics
```

由于本实例会缓存 Dashboard Studio 视图 XML，因此部署后重启了 Splunk。重启后在 Splunk Web 中验证了 App 和 Dashboard，并人工验收了布局、筛选器、图表、钻取、控制总数和告警行为。

## 部署到已有 Splunk Enterprise 系统

### 前置条件

- 已测试版本为 Splunk Enterprise 9.3.1。
- 目标 Splunk Web 和搜索服务运行正常。
- 拥有 Splunk 主机的 shell 权限，并可写入 `$SPLUNK_HOME/etc/apps`。
- 已知运行 Splunk 的操作系统账户。
- 仓库位于目标主机，或位于用于构建 App 包的中转主机。

以下示例使用 `/opt/splunk`。如果 Splunk 安装在其他位置，请修改 `SPLUNK_HOME`。

### 单机 Splunk Enterprise 部署

1. 克隆或传输仓库，然后进入仓库根目录。

   ```bash
   git clone <repository-url> galileo-dashboard
   cd galileo-dashboard
   ```

2. 确认 App 内打包的 lookup 与原始 CSV 一致。

   ```bash
   cmp banking_cn_economics/banking_cn_economics.csv \
       banking_model_economics/lookups/banking_cn_economics.csv
   ```

   `cmp` 应当不产生输出并成功退出。

3. 设置目标 Splunk 路径。

   ```bash
   export SPLUNK_HOME=/opt/splunk
   ```

4. 如果存在旧版 App，请先按组织的变更流程备份。然后复制可部署 App 的内容。

   ```bash
   sudo mkdir -p "$SPLUNK_HOME/etc/apps/banking_model_economics"
   sudo cp -a banking_model_economics/. \
       "$SPLUNK_HOME/etc/apps/banking_model_economics/"
   ```

5. 将文件所有者设置为运行 Splunk 的操作系统账户。如有需要，请替换 `splunk:splunk`。

   ```bash
   sudo chown -R splunk:splunk \
       "$SPLUNK_HOME/etc/apps/banking_model_economics"
   ```

6. 检查 App 文件和 Splunk 配置。

   ```bash
   test -r "$SPLUNK_HOME/etc/apps/banking_model_economics/lookups/banking_cn_economics.csv"
   sudo -u splunk "$SPLUNK_HOME/bin/splunk" btool check --debug
   ```

   继续前应检查所有告警。本 App 不会修复其他已有 App 产生的无关告警。

7. 使用平时运行 Splunk 的同一操作系统账户重启服务。

   ```bash
   sudo -u splunk "$SPLUNK_HOME/bin/splunk" restart --answer-yes --no-prompt
   ```

8. 确认服务状态。

   ```bash
   sudo -u splunk "$SPLUNK_HOME/bin/splunk" status --no-prompt
   ```

9. 打开 Splunk Web 并选择 **Apps > Banking Model Economics**，或直接访问：

   ```text
   http://<splunk-host>:8000/en-US/app/banking_model_economics/banking_multi_agent_model_economics
   ```

10. 如果浏览器仍显示旧视图，请使用 `Ctrl+Shift+R` 强制刷新。

App 已包含 lookup 文件和 lookup definition。除非要有意替换打包的数据快照，否则无须再次上传 CSV。

### 通过 Splunk Web 部署 App 包

如果不能直接访问 `$SPLUNK_HOME/etc/apps`，可以在仓库根目录构建标准 Splunk App 归档包：

```bash
tar -czf banking_model_economics-1.1.0.spl banking_model_economics
```

在 Splunk Web 中打开 **Apps > Manage Apps > Install app from file**，选择 `banking_model_economics-1.1.0.spl`；只有在确定要替换早期版本时才允许升级。如果安装程序要求重启 Splunk，请执行重启，然后完成同样的 lookup 和 Dashboard 验证。归档包必须保留 `banking_model_economics` 作为顶层目录。

### Search Head Cluster 或受管部署

不要将文件分别复制到 Search Head Cluster 的各成员。应打包 `banking_model_economics` 目录，并通过组织现有的标准机制分发，例如 Search Head Cluster deployer 或配置管理流水线。必须保留目录名称，因为它就是 App ID。完成所需的集群或 apply-bundle 操作后，再执行下方验证搜索。

## 部署验证

在 Splunk Web 中打开 **Search & Reporting**，将 App 上下文切换为 **Banking Model Economics**，然后运行：

```spl
| inputlookup banking_cn_economics.csv
| stats count AS rows
        dc(experiment_id) AS experiments
        dc(application_model) AS models
        sum(comparison_ready) AS comparison_ready_rows
        dc(trace_id) AS unique_traces
```

依次确认结果为 `12`、`2`、`2`、`12` 和 `12`。然后打开 Dashboard 并验证：

1. Model、Experiment 和 Scenario 筛选器都包含 `All` 和从 lookup 生成的值。
2. 选择全部筛选条件时，Executive 图中显示两个模型。
3. 两个完整快照 Experiment 的 Ground Truth accuracy 都是 `100.00%`。
4. 总耗时与原始控制总数一致。
5. 工作流成本不是零，并且不包含 Judge 成本。
6. Trace 图显示两个模型的 `T01` 到 `T06`。
7. 单击 Trace detail 行后，下方证据面板能够加载内容。
8. 最后的 Data integrity 面板无需滚动即可显示全部六项检查。
9. Dataset version `1, 6` 会产生可比性告警。

## 使用新 CSV 更新 Dashboard 数据

Dashboard 在每次面板搜索执行时通过 `inputlookup` 读取 `banking_cn_economics.csv`。因此，只要新 CSV 保持相同 schema，就可以在不重新设计可视化的情况下更新 Dashboard。该操作应作为受控的快照替换：先验证候选文件，确保仓库源文件与 App 内 lookup 完全相同，部署文件，最后再次验证正式 lookup。

### 1. 判断是否属于兼容的数据更新

满足以下条件时，可以作为仅数据更新处理：

- 文件名仍为 `banking_cn_economics.csv`。
- 表头字段名和字段含义保持不变。
- 每个物理 CSV 行仍代表一条根 Trace。
- `experiment_id + trace_id` 仍然唯一。
- 数值字段仍包含可解析的数值。
- 长文本中的字面换行仍采用编码形式，避免一条 Trace 占用多个物理 CSV 记录。

替换快照会移除新文件中不存在的旧行。如果必须保留历史数据，应有意将新 CSV 构建为累计快照，并确保所有新旧复合主键仍然唯一。Dashboard 没有时间选择器；除非使用筛选器限制，否则它会聚合文件中包含的全部 Experiment。

以下变化不属于仅数据更新，必须同步修改 SPL、Dashboard、验证规则和文档：

- 重命名或删除字段。
- 改变单位，例如将秒改为毫秒，或将 USD 改为其他货币。
- 改变 Supervisor、成本、Judge、合格或 readiness 字段的含义。
- 改变每行对应一条根 Trace 的数据粒度。

新增模型、Experiment、Dataset version 或场景时，下拉搜索会自动识别。没有显式颜色映射的新图表序列会使用 Splunk 默认配色。如果 Experiment 不再恰好包含 6 条 Trace，完整性面板会发出告警，同时应检查面向 Trace 的布局是否仍然易于阅读。

### 2. 保留当前版本并暂存候选文件

候选文件通过验证前，不要覆盖已经验收的快照。将新导出文件暂时保存在两个正式仓库路径之外，例如：

```text
/path/to/export/banking_cn_economics.csv
```

记录当前仓库 revision，或按正常源代码管理流程保存备份，以便恢复先前的数据集和 App 版本。然后比较基本文件属性：

```bash
file /path/to/export/banking_cn_economics.csv
head -n 1 /path/to/export/banking_cn_economics.csv > /tmp/new-banking-header.txt
head -n 1 banking_cn_economics/banking_cn_economics.csv > /tmp/current-banking-header.txt
cmp /tmp/current-banking-header.txt /tmp/new-banking-header.txt
```

表头比较不应产生任何输出。确认候选文件为 UTF-8 文本，并且长文本字段没有引入非预期的物理记录。当带引号字段可能包含真实换行时，简单的行数统计不能替代 CSV 解析。

### 3. 在 Splunk 中使用临时名称验证候选文件

最稳妥的生产流程是在替换正式文件前，将新文件作为临时 lookup 提供。将它复制或上传到同一个 App，并命名为：

```text
banking_cn_economics_candidate.csv
```

使用带 `.csv` 的明确文件名执行 `inputlookup` 时，这项临时测试不需要永久 lookup definition。请在 **Banking Model Economics** App 上下文中运行以下搜索。

检查数据量、维度、唯一性和 readiness：

```spl
| inputlookup banking_cn_economics_candidate.csv
| stats count AS rows
        dc(experiment_id) AS experiments
        dc(application_model) AS models
        dc(dataset_version) AS dataset_versions
        dc(trace_id) AS unique_trace_ids
        dc(eval(experiment_id."::".trace_id)) AS unique_keys
        sum(comparison_ready) AS comparison_ready_rows
| eval key_status=if(rows=unique_keys,"OK","CHECK DUPLICATES")
```

查找重复复合主键；有效结果应返回零行：

```spl
| inputlookup banking_cn_economics_candidate.csv
| stats count AS rows by experiment_id trace_id
| where rows!=1
```

检查核心数值字段和标识字段是否缺失：

```spl
| inputlookup banking_cn_economics_candidate.csv
| where isnull(experiment_id) OR isnull(trace_id)
     OR isnull(application_model) OR isnull(dataset_version)
     OR isnull(comparison_ready)
     OR isnull(trace_duration_seconds) OR isnull(trace_cost_usd)
     OR isnull(supervisor_output_tokens)
| stats count AS incomplete_rows
```

逐个检查所有 Experiment，不能假设原来两个模型的控制总数仍然适用：

```spl
| inputlookup banking_cn_economics_candidate.csv
| stats count AS rows
        dc(trace_id) AS unique_traces
        sum(comparison_ready) AS comparison_ready_rows
        sum(ground_truth_adherence_pass) AS gt_passed
        sum(ground_truth_adherence_scored) AS gt_scored
        sum(trace_duration_seconds) AS total_duration_seconds
        sum(trace_cost_usd) AS total_workflow_cost_usd
        sum(supervisor_output_tokens) AS supervisor_output_tokens
        by experiment_id experiment_name application_model dataset_version
| eval key_status=if(rows=unique_traces,"OK","DUPLICATE"),
       readiness_status=if(rows=comparison_ready_rows,"READY","INCOMPLETE"),
       trace_coverage=if(rows=6,"6 TRACES","REVIEW TRACE COUNT")
| table experiment_name application_model dataset_version rows unique_traces
        comparison_ready_rows gt_passed gt_scored total_duration_seconds
        total_workflow_cost_usd supervisor_output_tokens key_status
        readiness_status trace_coverage
```

必须调查所有重复、数据不完整、单位异常、转换失败以及 Trace 数量变化。不能使用 `fillnull value=0` 让候选文件通过验证。保存预期聚合结果，以便部署后进行对比。

### 4. 更新仓库中的两份 CSV

候选文件通过验证后，先替换原始快照，再将完全相同的文件复制到可部署 App：

```bash
cp /path/to/export/banking_cn_economics.csv \
   banking_cn_economics/banking_cn_economics.csv
cp banking_cn_economics/banking_cn_economics.csv \
   banking_model_economics/lookups/banking_cn_economics.csv
cmp banking_cn_economics/banking_cn_economics.csv \
    banking_model_economics/lookups/banking_cn_economics.csv
```

`cmp` 必须不产生任何输出。检查源代码管理 diff，并将源 CSV、打包 lookup 以及相关文档放在同一次提交中。

对于受控发布，应提升 `banking_model_economics/local/app.conf` 中的版本号。兼容的快照刷新适合使用 patch 版本递增，例如从 `1.1.0` 调整为 `1.1.1`。如果 Dashboard SPL 或布局也发生变化，应按组织的发布策略选择版本号。

### 5. 部署更新后的 lookup

使用与最初安装 App 相同的部署方式。

对于单机环境和仅数据更新，应先将文件暂存到目标目录，再通过重命名替换，避免面板搜索读取到尚未复制完整的 CSV。如果 Splunk 使用其他账户运行，请替换 `splunk:splunk`：

```bash
export SPLUNK_HOME=/opt/splunk
sudo cp banking_model_economics/lookups/banking_cn_economics.csv \
  "$SPLUNK_HOME/etc/apps/banking_model_economics/lookups/banking_cn_economics.csv.new"
sudo chown splunk:splunk \
  "$SPLUNK_HOME/etc/apps/banking_model_economics/lookups/banking_cn_economics.csv.new"
sudo chmod 0644 \
  "$SPLUNK_HOME/etc/apps/banking_model_economics/lookups/banking_cn_economics.csv.new"
sudo mv \
  "$SPLUNK_HOME/etc/apps/banking_model_economics/lookups/banking_cn_economics.csv.new" \
  "$SPLUNK_HOME/etc/apps/banking_model_economics/lookups/banking_cn_economics.csv"
```

如果修改了 `app.conf` 版本，也要使用相同的所有权和权限进行部署。使用 Splunk Web 时，重新构建 `.spl` 归档包并作为升级安装。使用 Search Head Cluster 时，通过 deployer 或组织的配置管理流水线分发更新后的 App，不能逐个修改成员。

### 6. 刷新 Splunk 和浏览器

直接替换基于文件的 lookup 后，新启动的搜索通常可以直接读取新数据，无须重启 Splunk。重新加载或重新打开 Dashboard，使所有面板创建新的搜索作业。将 Model、Experiment 和 Scenario 重置为 `All`，因为之前选择的 token 值可能已经不在新快照中。

出现以下任一情况时应重启 Splunk：

- 安装或 App 升级流程要求重启。
- 除 CSV 外还修改了配置文件，而当前实例没有重新加载这些文件。
- 已确认部署文件和权限正确，但新搜索仍返回旧 lookup。
- 环境变更策略要求 App 部署后重启。

使用平时运行 Splunk 的同一操作系统账户：

```bash
sudo -u splunk "$SPLUNK_HOME/bin/splunk" restart --answer-yes --no-prompt
sudo -u splunk "$SPLUNK_HOME/bin/splunk" status --no-prompt
```

如果搜索结果已经更新，但浏览器仍保留旧画面，请使用 `Ctrl+Shift+R`。

### 7. 验证正式 lookup 和 Dashboard

将候选验证搜索中的临时文件名改为 `banking_cn_economics.csv` 后重新运行。确认正式 lookup 产生之前保存的预期聚合结果。

然后对 Dashboard 进行端到端验证：

1. 新模型、Experiment 和场景出现在下拉框中。
2. Executive accuracy、总耗时和工作流总成本与已验证的聚合结果一致。
3. Trace 表和图表包含预期的 Trace 集合。
4. 工作流成本仍不包含 `judge_cost_usd`。
5. Supervisor aggregate 与 Supervisor direct 仍保持不同含义。
6. 对新加入的 Trace 执行钻取，可以加载 Ground Truth、generated output、Judge rationale 和 tools。
7. Data integrity 面板显示预期的 Dataset version、readiness 和 Trace coverage。
8. 所有面板均不存在 lookup 权限、SPL 或数值转换错误。

验收完成后，通过创建临时 lookup 时所使用的同一受管流程将其移除。在新快照通过运行观察前，应保留上一版本的发布制品。

### 8. 回滚失败的更新

恢复先前的仓库 revision 或已验收 App 包，重新部署其中的 `banking_cn_economics.csv`，然后再次执行正式 lookup 验证。仅数据回滚通常同样无须重启；如果同时回滚了配置，或目标环境策略要求，则应重启。不能根据 Dashboard 输出手工重建旧快照。

## Dashboard 使用说明

### 全局筛选器

所有面板都响应三个筛选器。

| 筛选器 | 用途 |
| --- | --- |
| **Model** | 选择一个 `application_model`，或比较所有模型。 |
| **Experiment** | 将所有指标限制在一个 Experiment name 内。 |
| **Scenario** | 使用稳定的跨 Experiment `scenario_key` 将 Dashboard 限制为一个场景。 |

默认值为 `All`。由于数据是固定 lookup 快照，因此没有时间范围输入。

### Executive 面板

#### Executive · Ground Truth accuracy

比较每个模型或 Experiment 的 Ground Truth adherence 合格百分比。公式为：

```text
100 × sum(ground_truth_adherence_pass) / sum(ground_truth_adherence_scored)
```

分母只包含已评分 Trace。缺少 Judge 分数时显示 `Not scored`，而不是判定失败。

#### Executive · Total duration

比较每个 Experiment 内 `trace_duration_seconds` 的总和。这是根 Trace 的墙钟耗时，单位为秒，不是所有子 Span 耗时的累加。

#### Executive · Total workflow cost

比较每个 Experiment 内 `trace_cost_usd` 的总和，表示应用工作流成本，单位为 USD。`judge_cost_usd` 是独立的评估费用，特意不包含在此指标中。

### 六轮 Trace 对比面板

#### Ground Truth adherence by Trace

每一行代表一个 Trace 和模型，包含：

| 字段 | 含义 |
| --- | --- |
| **Trace** | Experiment 内的 `T01` 到 `T06`。 |
| **Scenario** | 提交给银行应用的用户问题。 |
| **Experiment** | 模型、Dataset version 和缩短的 Experiment ID。 |
| **Result** | `Pass`、`Fail` 或 `Not scored`。 |
| **Score** | 可用时显示 Judge 分数。严格等于 `1` 才算合格。 |
| **Judge status** | Ground Truth 评估器报告的状态。 |

#### Supervisor aggregate output tokens by Trace

比较每条 Trace 的 `supervisor_output_tokens`。该指标来自顶层 Supervisor Span，包含下游 Agent 输出。它是任务要求的主要 Supervisor Token 指标，但不等同于仅由 Supervisor 自身生成的 Token。

#### Supervisor aggregate output tokens · total

在每个 Experiment 内汇总同一 aggregate token 指标。完整快照的控制总数为：`qwen3.7-flash` 16,878，`qwen3.8-max` 4,917。当所选 Experiment 范围不是 6 条 Trace 时，Trace coverage 字段会发出警告。

#### Trace duration

比较 `T01` 到 `T06` 的 `trace_duration_seconds`。可以用它定位较慢场景，并判断某个模型是持续较慢，还是只有个别延迟峰值。

#### Application workflow cost by Trace

在线性 USD 坐标上比较 `trace_cost_usd`。极小数值会保留足够精度，不会显示为零。该图不包含 Judge 成本。线性刻度可以突出最高的 Trace 成本，并避免产生对数距离的误解。

### Token、效率和运行面板

#### Supervisor direct vs downstream output tokens

显示根 Trace 输出的组成：

| 序列 | 含义 |
| --- | --- |
| **Supervisor direct** | `supervisor_direct_output_tokens`，仅由 Supervisor 直接推理轮次生成。 |
| **Downstream agents** | `non_supervisor_output_tokens`，不属于 Supervisor 直接推理的其余输出。 |

这些组成项用于描述根 Trace 输出。不能将任一组成项再次加到 `supervisor_output_tokens` 上，因为汇总值已经包含下游活动。

#### Efficiency

效率指标根据 Experiment 总数重新计算，而不是对逐行比率求平均。

| 指标 | 含义 |
| --- | --- |
| **Output tokens** | `trace_output_tokens` 之和。 |
| **Total tokens** | `trace_total_tokens` 之和，包含输入和输出。 |
| **Output tokens / second** | 总 output tokens 除以根 Trace 总耗时。 |
| **Cost / 1K total tokens** | `1000 × 工作流总成本 / total tokens`。 |

#### Runtime complexity & errors

| 指标 | 含义 |
| --- | --- |
| **LLM calls** | `llm_call_count` 之和。 |
| **Tool calls** | `tool_call_count` 之和。 |
| **Retriever calls** | `retriever_call_count` 之和。 |
| **Spans** | 所有已记录 `span_count` 的总和。 |
| **Error spans** | `error_span_count` 之和。 |
| **Timed-out LLM calls** | `timeout_llm_call_count` 之和。 |
| **Health** | 错误或 LLM timeout 非零时为 `WARNING`，否则为 `OK`。 |

### Trace 证据面板

#### Trace detail · click a row to inspect evidence

该表汇总每条已选 Trace 的关键质量、耗时、成本、Token、调用和错误指标。`Dataset v` 使源数据版本差异保持可见，`trace_id` 则作为稳定的钻取键保留。

| 列 | 含义 |
| --- | --- |
| **Seq** | Experiment 内的整数 Trace 顺序。 |
| **Scenario** | 该 Trace 的用户输入。 |
| **Model** | 该 Trace 中观测到的应用模型。 |
| **Dataset v** | Experiment 使用的 Dataset version。 |
| **GT result** | 派生的 `Pass`、`Fail` 或 `Not scored` 结果。 |
| **GT score** | Trace 已评分时的 Ground Truth Judge 分数。 |
| **Supervisor aggregate** | 顶层 Supervisor output token 汇总，包含下游 Agent。 |
| **Supervisor direct** | 仅由 Supervisor 直接推理产生的 output tokens。 |
| **Duration (s)** | 根 Trace 耗时，单位为秒。 |
| **Cost (USD)** | 根 Trace 应用工作流成本，不包含 Judge 成本。 |
| **LLM calls** | Trace 中的 LLM 调用次数。 |
| **Tool calls** | Trace 中的 Tool 调用次数。 |
| **Errors** | Trace 中的错误 Span 数。 |
| **trace_id** | 用于钻取的稳定根 Trace ID。 |

单击一行会设置 `tok_trace` Dashboard token，并在下方加载所选证据。

#### Selected Trace evidence

将所选 Trace 显示为多个带标签的部分：

- Scenario
- Ground Truth
- Generated Output
- Judge Rationale
- Tools

字面量 `\n` 仅在显示时转换为可读换行，不会修改 lookup 文件。

### Data integrity & comparability

最后一个面板将验证结果展开为六个可见行：

| 检查 | 含义 |
| --- | --- |
| **Dataset version alignment** | 列出已选 Dataset version；存在多个版本时产生告警。 |
| **Experiments** | 不同 `experiment_id` 的数量。 |
| **Models** | 不同 application model 的数量。 |
| **Traces** | 已选根 Trace 行数。 |
| **Comparison ready** | 核心指标完整的行数除以已选行数。 |
| **Six-trace coverage** | 每个 Experiment 的最小和最大已选 Trace 数；完整 Experiment 应均为 6。 |

关于 Dataset version `1` 和 `6` 的告警非常重要：Dashboard 准确报告已观测结果，但两个模型使用了不同 Dataset version，因此质量对比不属于严格的单变量模型实验。

## 指标定义

| 指标或字段 | 定义与解释 |
| --- | --- |
| `experiment_id` | 稳定的聚合边界。所有 Experiment 总数都按该值分组。 |
| `application_model` | 在应用 LLM Span 中观测到的模型。 |
| `dataset_version` | Experiment 使用的数据集修订版本。版本不同时必须保持可见。 |
| `scenario_key` | 用于在 Experiment 之间对齐相同场景的稳定键。 |
| `trace_sequence` | Experiment 内的 Trace 顺序，显示为 `T01` 到 `T06`。 |
| `trace_id` | 稳定的根 Trace 标识符和钻取键。 |
| `comparison_ready` | 只有核心 Trace、Judge、Supervisor 指标完整且 Trace 成功时才为 `1`。 |
| `ground_truth_adherence_score` | Ground Truth Judge 分数。 |
| `ground_truth_adherence_pass` | 有效 Judge 分数严格等于 `1` 时为 `1`；`0` 表示已评分但失败；null 表示未评分。 |
| `ground_truth_adherence_scored` | 存在有效 Judge 分数时为 `1`，用作准确度分母。 |
| `supervisor_output_tokens` | 顶层 Supervisor Span 汇总，包含下游 Agent。 |
| `supervisor_direct_output_tokens` | 仅由 Supervisor 直接 LLM 推理产生的输出。 |
| `non_supervisor_output_tokens` | 根 Trace 中不由 Supervisor 直接推理生成的输出。 |
| `trace_duration_seconds` | 根 Trace 的耗时，单位为秒。 |
| `trace_cost_usd` | 根 Trace 的应用工作流平台成本。 |
| `judge_cost_usd` | 独立 Judge 或评估成本；绝不自动加入工作流成本。 |
| `trace_input_tokens` | 根 Trace input token 汇总。 |
| `trace_output_tokens` | 根 Trace output token 汇总。 |
| `trace_total_tokens` | 根 Trace input 与 output token 汇总。 |
| `span_count` | 一条 Trace 记录的 Span 总数。 |
| `llm_call_count` | LLM 调用次数。 |
| `tool_call_count` | Tool 调用次数。 |
| `retriever_call_count` | Retriever 调用次数。 |
| `error_span_count` | 带错误的 Span 数。 |
| `timeout_llm_call_count` | 超时的 LLM 调用数。 |

## 维护与故障排查

### 替换数据快照

请使用[使用新 CSV 更新 Dashboard 数据](#使用新-csv-更新-dashboard-数据)中的完整流程，其中包括候选文件验证、仓库同步、部署、重启判断、更新后验收和回滚。绝不能只替换仓库中的一份 CSV，也不能用 `fillnull value=0` 隐藏缺失的核心指标。

### Dashboard 修改后没有显示

1. 确认修改后的 XML 位于已部署 App 的 `local/data/ui/views` 目录。
2. 确认 `dashboard_source/*.json` 与 XML 的 CDATA 定义保持同步。
3. 检查 App 是否启用且可见。
4. 如果实例缓存了视图 XML，请重启 Splunk。
5. 使用 `Ctrl+Shift+R` 强制刷新浏览器。

### 找不到 App 或 Dashboard

检查：

```bash
"$SPLUNK_HOME/bin/splunk" status --no-prompt
"$SPLUNK_HOME/bin/splunk" btool check --debug
ls -l "$SPLUNK_HOME/etc/apps/banking_model_economics/local/data/ui/views/"
```

同时检查 Splunk 服务账户是否拥有文件所有权和读取权限。

### 面板没有数据

- 在当前 App 上下文中运行 `| inputlookup banking_cn_economics.csv | head 1`。
- 将三个筛选器全部重置为 `All`。
- 确认 CSV 表头未发生变化。
- 检查搜索作业消息中是否存在无效 SPL 或 lookup 权限错误。
- 确认 `metadata/local.meta` 中的视图和 lookup 权限与查看者角色匹配。

### 成本解释

除非单独且明确标注的分析需要计算应用加评估的总支出，否则绝不能合并 `trace_cost_usd` 和 `judge_cost_usd`。本 Dashboard 专门报告工作流经济性，并将 Judge 费用保持独立。
