[English](README.md) | [简体中文](README.zh-CN.md)

# Banking Model Economics Dashboard for Splunk Enterprise

This repository contains a deployable Splunk app that turns the `banking_cn_economics.csv` snapshot into the **Multi-Agent Banking ChatBot Model Economics** Dashboard Studio dashboard. It compares model quality, duration, workflow cost, token usage, runtime complexity, and trace-level evidence.

The implementation was built and validated on Splunk Enterprise 9.3.1 on Ubuntu. The current app version is `1.1.0`.

## Contents

- [Repository layout](#repository-layout)
- [How the dashboard was built](#how-the-dashboard-was-built)
- [Deploy to an existing Splunk Enterprise system](#deploy-to-an-existing-splunk-enterprise-system)
- [Validate the deployment](#validate-the-deployment)
- [Update dashboard data from a new CSV](#update-dashboard-data-from-a-new-csv)
- [Dashboard user guide](#dashboard-user-guide)
- [Metric definitions](#metric-definitions)
- [Maintenance and troubleshooting](#maintenance-and-troubleshooting)

## Repository layout

```text
.
├── banking_cn_economics/
│   └── banking_cn_economics.csv              # Original source snapshot
├── banking_model_economics/                   # Deployable Splunk app
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

Splunk loads the app from `banking_model_economics/`. The file in `dashboard_source/` is the readable Dashboard Studio source. The operational dashboard is embedded in the CDATA definition in `local/data/ui/views/banking_multi_agent_model_economics.xml`.

## How the dashboard was built

### 1. Inspect and validate the CSV

The CSV is a UTF-8, LF-terminated snapshot. One physical row represents one root Trace. The stable composite key is:

```text
experiment_id + trace_id
```

The current snapshot contains 12 rows, two Experiments, two application models, six Traces per Experiment, and 12 unique Trace IDs. Literal `\n` sequences inside long text fields preserve the CSV row structure.

The first validation search establishes the row and key controls:

```spl
| inputlookup banking_cn_economics.csv
| stats count AS rows
        dc(experiment_id) AS experiments
        dc(application_model) AS models
        sum(comparison_ready) AS comparison_ready_rows
        dc(trace_id) AS unique_traces
```

Expected result:

| rows | experiments | models | comparison_ready_rows | unique_traces |
| ---: | ---: | ---: | ---: | ---: |
| 12 | 2 | 2 | 12 | 12 |

The second search validates uniqueness and six-Trace coverage within every Experiment:

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

No missing value is replaced with zero. A duplicate key, an incomplete core metric, `comparison_ready=0`, or an unexpected Trace count must remain visible and be investigated.

The raw control totals are:

| Application model | Dataset version | GT passed | Total duration (s) | Workflow cost (USD) | Supervisor aggregate output tokens |
| --- | ---: | ---: | ---: | ---: | ---: |
| `qwen3.7-flash` | 1 | 6/6 | 478.763938 | 0.002528790050 | 16,878 |
| `qwen3.8-max` | 6 | 6/6 | 157.938983 | 0.105030042003 | 4,917 |

The differing Dataset versions are intentional source facts. They are shown as a comparability warning rather than hidden or normalized away.

### 2. Create an isolated Splunk app

The dashboard is isolated in the app ID `banking_model_economics`. The app is visible and enabled through `local/app.conf`:

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

Keeping the lookup, view, navigation, and permissions inside one app makes the solution portable and prevents changes to `system/default` or unrelated apps.

### 3. Package the CSV as a lookup

The validated CSV was copied without content changes to:

```text
banking_model_economics/lookups/banking_cn_economics.csv
```

`local/transforms.conf` defines the file-based lookup:

```ini
[banking_cn_economics]
filename = banking_cn_economics.csv
```

Dashboard searches use `| inputlookup banking_cn_economics.csv`. The data is a snapshot, so the dashboard deliberately has no global time picker.

### 4. Configure navigation and permissions

`local/data/ui/nav/default.xml` makes the dashboard the app's default view:

```xml
<nav search_view="search" color="#0B1220">
  <view name="banking_multi_agent_model_economics" default="true" />
</nav>
```

`metadata/local.meta` grants read access to all roles and write access to `admin` and `power` for the lookup, transform, view, and navigation objects. Adapt those role lists if the destination organization requires tighter access.

### 5. Build the Dashboard Studio definition

The dashboard uses an Absolute layout with a 1440 px design width. It contains:

- Three dynamic dropdown inputs: Model, Experiment, and Scenario.
- 15 visualizations.
- 17 `ds.search` data sources.
- Executive model comparisons first.
- Six-Trace comparisons and evidence in the middle.
- Data integrity and comparability checks at the end.

Every analytical search applies the same three tokens:

```spl
| search application_model="$tok_model$"
         experiment_name="$tok_experiment$"
         scenario_key="$tok_scenario$"
```

The dropdowns default to `All` / `*`, and their choices are generated from the lookup.

Numeric fields are converted with `tonumber()` before aggregation. Each Experiment remains an explicit aggregation boundary through `experiment_id`; results from different Experiments are never silently combined as though they came from one run.

The executive charts reshape one aggregate per Experiment into model series with `chart ... over ... by series_label`. Trace charts use `trace_label` values `T01` through `T06`, which makes the six rounds easy to compare across models.

The readable definition is maintained in:

```text
banking_model_economics/dashboard_source/banking_multi_agent_model_economics.dashboard.json
```

The same JSON is embedded in:

```text
banking_model_economics/local/data/ui/views/banking_multi_agent_model_economics.xml
```

When changing the dashboard in source control, keep these two definitions synchronized. Splunk loads the XML view, not the standalone JSON file.

### 6. Apply display and semantic safeguards

- Ordinary metrics display two decimal places, such as `20.00`.
- Very small monetary values retain the first meaningful digits so they do not render as zero, such as `0.000013`.
- Workflow cost and Judge cost remain separate. Judge cost is excluded from executive and per-Trace application workflow cost charts.
- `supervisor_output_tokens` is the top-level aggregate and already includes downstream Agents.
- `supervisor_direct_output_tokens` represents only direct Supervisor reasoning. It must not be added to the aggregate value.
- Unscored Ground Truth rows are labeled `Not scored`; they are not counted as failures.
- The Ground Truth pass-rate denominator is the number of scored Traces, not the total row count.
- Error-free rows remain neutral. Only nonzero errors or LLM timeouts create a runtime warning.

### 7. Install, restart, and verify

The completed app directory was copied to the Splunk app directory, normally:

```text
/opt/splunk/etc/apps/banking_model_economics
```

Splunk was restarted because this instance cached Dashboard Studio view XML. After restart, the app and dashboard were verified in Splunk Web and the layout, filters, charts, drilldown, control totals, and warning behavior were manually accepted.

## Deploy to an existing Splunk Enterprise system

### Prerequisites

- Splunk Enterprise 9.3.1 is the tested version.
- The target Splunk Web and search services are healthy.
- You have shell access to the Splunk host and permission to write under `$SPLUNK_HOME/etc/apps`.
- You know the operating-system account that runs Splunk.
- The repository is available on the target host or on a staging host used to build an app package.

The examples below use `/opt/splunk`. Change `SPLUNK_HOME` if your installation uses another location.

### Standalone Splunk Enterprise deployment

1. Clone or transfer the repository and enter its root directory.

   ```bash
   git clone <repository-url> galileo-dashboard
   cd galileo-dashboard
   ```

2. Confirm that the packaged lookup matches the source CSV.

   ```bash
   cmp banking_cn_economics/banking_cn_economics.csv \
       banking_model_economics/lookups/banking_cn_economics.csv
   ```

   `cmp` should produce no output and exit successfully.

3. Set the target Splunk location.

   ```bash
   export SPLUNK_HOME=/opt/splunk
   ```

4. If an older copy of the app exists, back it up according to your organization's change procedure. Then copy the deployable app contents.

   ```bash
   sudo mkdir -p "$SPLUNK_HOME/etc/apps/banking_model_economics"
   sudo cp -a banking_model_economics/. \
       "$SPLUNK_HOME/etc/apps/banking_model_economics/"
   ```

5. Assign ownership to the operating-system account that runs Splunk. Replace `splunk:splunk` when necessary.

   ```bash
   sudo chown -R splunk:splunk \
       "$SPLUNK_HOME/etc/apps/banking_model_economics"
   ```

6. Check the app files and Splunk configuration.

   ```bash
   test -r "$SPLUNK_HOME/etc/apps/banking_model_economics/lookups/banking_cn_economics.csv"
   sudo -u splunk "$SPLUNK_HOME/bin/splunk" btool check --debug
   ```

   Review warnings before proceeding. Warnings from unrelated pre-existing apps are not fixed by this package.

7. Restart Splunk as the same operating-system account that normally runs it.

   ```bash
   sudo -u splunk "$SPLUNK_HOME/bin/splunk" restart --answer-yes --no-prompt
   ```

8. Confirm service state.

   ```bash
   sudo -u splunk "$SPLUNK_HOME/bin/splunk" status --no-prompt
   ```

9. Open Splunk Web and select **Apps > Banking Model Economics**, or browse directly to:

   ```text
   http://<splunk-host>:8000/en-US/app/banking_model_economics/banking_multi_agent_model_economics
   ```

10. If an older view remains in the browser, perform a hard refresh with `Ctrl+Shift+R`.

The app already contains the lookup file and lookup definition. Do not upload the CSV separately unless you intentionally want to replace the packaged snapshot.

### Splunk Web package deployment

If direct access to `$SPLUNK_HOME/etc/apps` is not available, build a standard Splunk app archive from the repository root:

```bash
tar -czf banking_model_economics-1.1.0.spl banking_model_economics
```

In Splunk Web, open **Apps > Manage Apps > Install app from file**, select `banking_model_economics-1.1.0.spl`, and allow upgrade only when intentionally replacing an earlier version. Restart Splunk if the installer requests it, then perform the same lookup and dashboard validation. The archive must retain `banking_model_economics` as its top-level directory.

### Search head cluster or managed deployment

Do not copy files independently to search-head cluster members. Package the `banking_model_economics` directory and distribute it with the organization's normal mechanism, such as the search head cluster deployer or a configuration-management pipeline. Preserve the directory name because it is the app ID. Perform the required cluster/apply-bundle operation and then run the validation searches below.

## Validate the deployment

In Splunk Web, open **Search & Reporting**, set the app context to **Banking Model Economics**, and run:

```spl
| inputlookup banking_cn_economics.csv
| stats count AS rows
        dc(experiment_id) AS experiments
        dc(application_model) AS models
        sum(comparison_ready) AS comparison_ready_rows
        dc(trace_id) AS unique_traces
```

Confirm `12`, `2`, `2`, `12`, and `12` respectively. Then open the dashboard and verify:

1. The Model, Experiment, and Scenario filters contain `All` and lookup-derived values.
2. The executive charts show both models when all filters are selected.
3. Ground Truth accuracy is `100.00%` for both full-snapshot Experiments.
4. Total durations agree with the raw control totals.
5. Workflow costs are nonzero and Judge cost is not included.
6. Trace charts show `T01` through `T06` for both models.
7. Clicking a Trace detail row loads the evidence panel.
8. The final Data integrity panel shows all six checks without scrolling.
9. Dataset versions `1, 6` produce the comparability warning.

## Update dashboard data from a new CSV

The dashboard reads `banking_cn_economics.csv` through `inputlookup` each time a panel search runs. A new CSV with the same schema can therefore update the dashboard without redesigning the visualizations. Treat the operation as a controlled snapshot replacement: validate the candidate first, keep the repository source and packaged lookup identical, deploy the file, and then validate the canonical lookup again.

### 1. Decide whether this is a compatible data update

A data-only update is compatible when:

- The filename remains `banking_cn_economics.csv`.
- The header names and field meanings remain unchanged.
- One physical CSV row still represents one root Trace.
- `experiment_id + trace_id` remains unique.
- Numeric fields still contain parseable numeric values.
- Literal line breaks in long text remain encoded so that one Trace does not span multiple physical CSV records.

Replacing a snapshot removes rows that are not present in the new file. If history must be retained, build the new CSV intentionally as a cumulative snapshot and ensure all old and new composite keys remain unique. The dashboard has no time picker and will aggregate every included Experiment unless a filter restricts it.

The following changes are not data-only updates and require corresponding SPL, Dashboard, validation, and documentation changes:

- Renaming or removing a field.
- Changing units, such as seconds to milliseconds or USD to another currency.
- Changing the meaning of Supervisor, cost, Judge, pass, or readiness fields.
- Changing one-row-per-root-Trace granularity.

Adding models, Experiments, Dataset versions, or scenarios is supported automatically by the dropdown searches. New chart series that do not have an explicit color mapping use the Splunk default palette. If Experiments no longer contain exactly six Traces, the integrity panel warns and the Trace-oriented layout should be reviewed for readability.

### 2. Preserve the current version and stage the candidate

Do not overwrite the accepted snapshot before validation. Keep the new export outside the two canonical repository paths, for example:

```text
/path/to/export/banking_cn_economics.csv
```

Record the current repository revision or create a normal source-control backup so the previous dataset and app version can be restored. Then compare basic file properties:

```bash
file /path/to/export/banking_cn_economics.csv
head -n 1 /path/to/export/banking_cn_economics.csv > /tmp/new-banking-header.txt
head -n 1 banking_cn_economics/banking_cn_economics.csv > /tmp/current-banking-header.txt
cmp /tmp/current-banking-header.txt /tmp/new-banking-header.txt
```

The header comparison should produce no output. Confirm that the candidate is UTF-8 text and that its multiline text fields have not introduced unintended physical records. A simple line count is not a substitute for CSV parsing when quoted fields may contain real newlines.

### 3. Validate the candidate in Splunk under a temporary name

The safest production workflow is to expose the new file as a temporary lookup before replacing the canonical file. Copy or upload it to the same App as:

```text
banking_cn_economics_candidate.csv
```

Using an explicit `.csv` filename with `inputlookup` does not require a permanent lookup definition for this temporary test. Run the following searches in the **Banking Model Economics** App context.

Check volume, dimensions, uniqueness, and readiness:

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

Find duplicate composite keys; a valid result returns no rows:

```spl
| inputlookup banking_cn_economics_candidate.csv
| stats count AS rows by experiment_id trace_id
| where rows!=1
```

Check the core numeric and identity fields for missing values:

```spl
| inputlookup banking_cn_economics_candidate.csv
| where isnull(experiment_id) OR isnull(trace_id)
     OR isnull(application_model) OR isnull(dataset_version)
     OR isnull(comparison_ready)
     OR isnull(trace_duration_seconds) OR isnull(trace_cost_usd)
     OR isnull(supervisor_output_tokens)
| stats count AS incomplete_rows
```

Review every Experiment rather than assuming the original two-model controls still apply:

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

Investigate every duplicate, incomplete row, unexpected unit, failed conversion, and Trace-count change. Do not use `fillnull value=0` to make the candidate pass validation. Save the expected aggregate results for post-deployment comparison.

### 4. Update both repository copies

After the candidate passes validation, replace the original snapshot and then copy that exact file into the deployable App:

```bash
cp /path/to/export/banking_cn_economics.csv \
   banking_cn_economics/banking_cn_economics.csv
cp banking_cn_economics/banking_cn_economics.csv \
   banking_model_economics/lookups/banking_cn_economics.csv
cmp banking_cn_economics/banking_cn_economics.csv \
    banking_model_economics/lookups/banking_cn_economics.csv
```

`cmp` must produce no output. Review the source-control diff and commit the source CSV, packaged lookup, and related documentation together.

For a controlled release, increment the version in `banking_model_economics/local/app.conf`. A patch increment such as `1.1.0` to `1.1.1` is appropriate for a compatible snapshot refresh. If Dashboard SPL or layout also changes, choose a version that matches the organization's release policy.

### 5. Deploy the updated lookup

Choose the same deployment method used for the original App.

For a standalone host and a data-only update, stage the file in the destination directory and rename it into place so a panel search does not read a partially copied CSV. Replace `splunk:splunk` when Splunk runs under another account:

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

If `app.conf` was versioned, deploy it with the same ownership and permissions. For Splunk Web, rebuild the `.spl` archive and install it as an upgrade. For a Search Head Cluster, distribute the updated App through the deployer or the organization's configuration-management pipeline; do not update members individually.

### 6. Refresh Splunk and the browser

A direct replacement of a file-based lookup normally becomes visible to newly dispatched searches without a Splunk restart. Reload the Dashboard or open it again so all panels create new search jobs. Reset Model, Experiment, and Scenario to `All`, because a previously selected token may not exist in the new snapshot.

Restart Splunk when any of the following applies:

- The installation or App-upgrade workflow requests it.
- Configuration files other than the CSV were changed and the instance does not reload them.
- New searches still return the old lookup after the deployed file and permissions have been verified.
- The environment's change policy requires a restart after App deployment.

Use the same operating-system account that normally runs Splunk:

```bash
sudo -u splunk "$SPLUNK_HOME/bin/splunk" restart --answer-yes --no-prompt
sudo -u splunk "$SPLUNK_HOME/bin/splunk" status --no-prompt
```

If the searches are current but the browser retains an old rendering, use `Ctrl+Shift+R`.

### 7. Validate the canonical lookup and Dashboard

Repeat the candidate validation searches with `banking_cn_economics.csv`, not the temporary filename. Confirm that the canonical lookup produces the saved expected aggregates.

Then verify the Dashboard end to end:

1. New models, Experiments, and scenarios appear in the dropdowns.
2. Executive accuracy, total duration, and total workflow cost match the validated aggregates.
3. Trace tables and charts contain the expected Trace set.
4. Workflow cost still excludes `judge_cost_usd`.
5. Supervisor aggregate and Supervisor direct values retain their distinct meanings.
6. Trace drilldown loads Ground Truth, generated output, Judge rationale, and tools from a newly added Trace.
7. The Data integrity panel reports the expected Dataset versions, readiness, and Trace coverage.
8. No panel reports lookup permission, SPL, or numeric conversion errors.

After acceptance, remove the temporary candidate lookup through the same managed process used to create it. Keep the previous release artifact until the new snapshot has passed operational monitoring.

### 8. Roll back an unsuccessful update

Restore the prior repository revision or previously accepted App package, redeploy its `banking_cn_economics.csv`, and repeat the canonical lookup validation. A data-only rollback normally follows the same no-restart behavior; restart when configuration was also rolled back or the target environment requires it. Do not reconstruct the previous snapshot manually from dashboard output.

## Dashboard user guide

### Global filters

All panels respond to all three filters.

| Filter | Purpose |
| --- | --- |
| **Model** | Select one `application_model` or compare all models. |
| **Experiment** | Restrict every metric to one Experiment name. |
| **Scenario** | Restrict the dashboard to one stable cross-Experiment `scenario_key`. |

The default is `All`. Because the data is a fixed lookup snapshot, there is no time-range input.

### Executive panels

#### Executive · Ground Truth accuracy

Compares the Ground Truth adherence pass percentage for each model/Experiment. The formula is:

```text
100 × sum(ground_truth_adherence_pass) / sum(ground_truth_adherence_scored)
```

Only scored Traces are in the denominator. A missing Judge score is `Not scored`, not a failure.

#### Executive · Total duration

Compares the sum of `trace_duration_seconds` within each Experiment. This is root Trace wall-clock duration in seconds, not the sum of every child Span duration.

#### Executive · Total workflow cost

Compares the sum of `trace_cost_usd` within each Experiment. It represents application workflow cost in USD. `judge_cost_usd` is an independent evaluation expense and is intentionally excluded.

### Six-Trace comparison panels

#### Ground Truth adherence by Trace

Shows one row per Trace/model with:

| Field | Meaning |
| --- | --- |
| **Trace** | `T01` through `T06` within an Experiment. |
| **Scenario** | The user question supplied to the banking application. |
| **Experiment** | Model, Dataset version, and shortened Experiment ID. |
| **Result** | `Pass`, `Fail`, or `Not scored`. |
| **Score** | Judge score when available. A strict score of `1` is a pass. |
| **Judge status** | Status reported by the Ground Truth evaluator. |

#### Supervisor aggregate output tokens by Trace

Compares `supervisor_output_tokens` for each Trace. This comes from the top-level Supervisor Span and includes downstream Agent output. It is the required primary Supervisor token metric, but it is not Supervisor-only generation.

#### Supervisor aggregate output tokens · total

Sums the same aggregate token metric inside each Experiment. Full-snapshot control totals are 16,878 for `qwen3.7-flash` and 4,917 for `qwen3.8-max`. The Trace coverage field warns when the selected Experiment scope is not six Traces.

#### Trace duration

Compares `trace_duration_seconds` for `T01` through `T06`. Use it to identify slow scenarios and to see whether one model is consistently slower or only has isolated latency peaks.

#### Application workflow cost by Trace

Compares `trace_cost_usd` on a linear USD scale. Small values retain enough precision to remain visible. Judge cost is excluded. The linear scale keeps the largest Trace cost visually prominent and avoids implying logarithmic distance.

### Token, efficiency, and runtime panels

#### Supervisor direct vs downstream output tokens

Displays the composition of root Trace output:

| Series | Meaning |
| --- | --- |
| **Supervisor direct** | `supervisor_direct_output_tokens`, generated only by direct Supervisor reasoning turns. |
| **Downstream agents** | `non_supervisor_output_tokens`, the remainder generated outside direct Supervisor reasoning. |

These components describe the root output composition. Do not add either component to `supervisor_output_tokens`; the aggregate already contains downstream activity.

#### Efficiency

Rates are recomputed from Experiment totals, not averaged from row-level rates.

| Metric | Meaning |
| --- | --- |
| **Output tokens** | Sum of `trace_output_tokens`. |
| **Total tokens** | Sum of `trace_total_tokens`, including input and output. |
| **Output tokens / second** | Total output tokens divided by total root Trace duration. |
| **Cost / 1K total tokens** | `1000 × total workflow cost / total tokens`. |

#### Runtime complexity & errors

| Metric | Meaning |
| --- | --- |
| **LLM calls** | Sum of `llm_call_count`. |
| **Tool calls** | Sum of `tool_call_count`. |
| **Retriever calls** | Sum of `retriever_call_count`. |
| **Spans** | Sum of all recorded `span_count` values. |
| **Error spans** | Sum of `error_span_count`. |
| **Timed-out LLM calls** | Sum of `timeout_llm_call_count`. |
| **Health** | `WARNING` when errors or LLM timeouts are nonzero; otherwise `OK`. |

### Trace evidence panels

#### Trace detail · click a row to inspect evidence

This table joins the key quality, duration, cost, token, call, and error metrics for every selected Trace. `Dataset v` makes the source-version difference visible. The `trace_id` is retained as the stable drilldown key.

| Column | Meaning |
| --- | --- |
| **Seq** | Integer Trace order within its Experiment. |
| **Scenario** | User input for the Trace. |
| **Model** | Application model observed for the Trace. |
| **Dataset v** | Dataset version used by the Experiment. |
| **GT result** | Derived `Pass`, `Fail`, or `Not scored` result. |
| **GT score** | Ground Truth Judge score when the Trace was scored. |
| **Supervisor aggregate** | Top-level Supervisor output token aggregate, including downstream Agents. |
| **Supervisor direct** | Output tokens from direct Supervisor reasoning only. |
| **Duration (s)** | Root Trace duration in seconds. |
| **Cost (USD)** | Root Trace application workflow cost; Judge cost is excluded. |
| **LLM calls** | Number of LLM invocations in the Trace. |
| **Tool calls** | Number of tool invocations in the Trace. |
| **Errors** | Number of error Spans in the Trace. |
| **trace_id** | Stable root Trace ID used by the drilldown. |

Clicking a row sets the `tok_trace` dashboard token and loads the selected evidence below.

#### Selected Trace evidence

Displays the selected Trace as labeled sections:

- Scenario
- Ground Truth
- Generated Output
- Judge Rationale
- Tools

Literal `\n` sequences are converted to readable line breaks only for display. The lookup file is not modified.

### Data integrity & comparability

The last panel expands validation into six visible rows:

| Check | Meaning |
| --- | --- |
| **Dataset version alignment** | Lists selected Dataset versions and warns when more than one version is present. |
| **Experiments** | Count of distinct `experiment_id` values. |
| **Models** | Count of distinct application models. |
| **Traces** | Number of selected root Trace rows. |
| **Comparison ready** | Complete core-metric rows divided by selected rows. |
| **Six-trace coverage** | Minimum and maximum selected Trace counts per Experiment; full Experiments should both be six. |

The warning about Dataset versions `1` and `6` is important: the dashboard accurately reports observed results, but the quality comparison is not a strict single-variable model experiment because the models used different Dataset versions.

## Metric definitions

| Metric or field | Definition and interpretation |
| --- | --- |
| `experiment_id` | Stable aggregation boundary. All Experiment totals group by this value. |
| `application_model` | Model observed in the application LLM Spans. |
| `dataset_version` | Dataset revision used by an Experiment. It must remain visible when versions differ. |
| `scenario_key` | Stable key used to align equivalent scenarios across Experiments. |
| `trace_sequence` | Trace order within an Experiment, displayed as `T01` to `T06`. |
| `trace_id` | Stable root Trace identifier and drilldown key. |
| `comparison_ready` | `1` only when core Trace, Judge, and Supervisor metrics are complete and the Trace succeeded. |
| `ground_truth_adherence_score` | Ground Truth Judge score. |
| `ground_truth_adherence_pass` | `1` only when the valid Judge score is strictly `1`; `0` is a scored failure; null is unscored. |
| `ground_truth_adherence_scored` | `1` when a valid Judge score exists; used as the accuracy denominator. |
| `supervisor_output_tokens` | Top-level Supervisor Span aggregate, including downstream Agents. |
| `supervisor_direct_output_tokens` | Output from direct Supervisor LLM reasoning only. |
| `non_supervisor_output_tokens` | Root Trace output not generated by direct Supervisor reasoning. |
| `trace_duration_seconds` | Root Trace duration in seconds. |
| `trace_cost_usd` | Application workflow platform cost for the root Trace. |
| `judge_cost_usd` | Separate Judge/evaluation cost; never automatically added to workflow cost. |
| `trace_input_tokens` | Root Trace input token aggregate. |
| `trace_output_tokens` | Root Trace output token aggregate. |
| `trace_total_tokens` | Root Trace input plus output token aggregate. |
| `span_count` | Total recorded Span count for a Trace. |
| `llm_call_count` | LLM invocation count. |
| `tool_call_count` | Tool invocation count. |
| `retriever_call_count` | Retrieval invocation count. |
| `error_span_count` | Count of Spans carrying an error. |
| `timeout_llm_call_count` | Count of timed-out LLM calls. |

## Maintenance and troubleshooting

### Replace the snapshot

Use the complete procedure in [Update dashboard data from a new CSV](#update-dashboard-data-from-a-new-csv). It covers candidate validation, repository synchronization, deployment, restart criteria, post-update acceptance, and rollback. Never replace only one of the two repository CSV copies, and do not use `fillnull value=0` to hide missing core metrics.

### Dashboard changes do not appear

1. Confirm the edited XML is under the deployed app's `local/data/ui/views` directory.
2. Confirm `dashboard_source/*.json` and the XML CDATA definition are synchronized.
3. Check that the app is enabled and visible.
4. Restart Splunk when the instance caches view XML.
5. Hard-refresh the browser with `Ctrl+Shift+R`.

### App or dashboard is missing

Check:

```bash
"$SPLUNK_HOME/bin/splunk" status --no-prompt
"$SPLUNK_HOME/bin/splunk" btool check --debug
ls -l "$SPLUNK_HOME/etc/apps/banking_model_economics/local/data/ui/views/"
```

Also verify file ownership and read permissions for the Splunk service account.

### Panels show no data

- Run `| inputlookup banking_cn_economics.csv | head 1` in the app context.
- Reset all three filters to `All`.
- Confirm the CSV header has not changed.
- Check search-job messages for invalid SPL or lookup permission errors.
- Confirm the view and lookup permissions in `metadata/local.meta` match the viewer's role.

### Cost interpretation

Never combine `trace_cost_usd` and `judge_cost_usd` unless a separate, explicitly labeled analysis requires total application-plus-evaluation spend. The supplied dashboard reports workflow economics and keeps Judge expense separate by design.
