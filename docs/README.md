# Converting Sigma Rules to Splunk

This guide explains how to turn the Sigma rules in this repo into Splunk SPL searches with
[sigma-cli](https://github.com/SigmaHQ/sigma-cli) and the
[pySigma Splunk backend](https://github.com/SigmaHQ/pySigma-backend-splunk).

```
Sigma rule (generic YAML)
  → splunk_windows pipeline   generic Windows log sources → Splunk sources
  → custom pipeline           this repo's sources, EventCodes and field names
  → splunk backend            renders SPL
  → SPL query / savedsearches.conf
```

---

## 1. Prerequisites

```bash
python3 -m venv ~/sigma-venv
source ~/sigma-venv/bin/activate
pip install sigma-cli
sigma plugin install splunk       # installs pySigma-backend-splunk
```

Check what's available:

```bash
sigma list targets                # should include: splunk
sigma list pipelines splunk       # splunk_windows, splunk_cim, splunk_sysmon_acceleration
sigma list formats splunk         # default, savedsearches, data_model
```

> Tested with `sigma-cli 3.1.0`, `pySigma 1.5.1` and `pysigma-backend-splunk 2.1.0`.

---

## 2. Repository layout

```
detection-as-code/
├── pipelines/
│   └── splunk_pipeline.yml        # custom processing pipeline for our Splunk environment
├── rules/
│   ├── endpoint/
│   │   ├── windows/<tactic>/      # e.g. credential_access/lsass_procdump.yml
│   │   └── linux/
│   ├── network/
│   └── cloud/aws/{cloudtrail,iam}/
└── docs/
    └── README.md                  # this file
```

**Run every command from the repo root.** All paths below are relative to it.

---

## 3. Adding a rule from SigmaHQ

You can download a single rule straight from the
[SigmaHQ repo](https://github.com/SigmaHQ/sigma) without cloning it. Pin the URL to a
**release tag** (`rYYYY-MM-DD`) so the rule can't change unexpectedly:

```bash
SIGMA_REF=r2026-07-01   # release tag, or a full commit hash

curl -fsSL -o rules/endpoint/windows/credential_access/lsass_procdump.yml \
  https://raw.githubusercontent.com/SigmaHQ/sigma/${SIGMA_REF}/rules/windows/process_creation/proc_creation_win_sysinternals_procdump_lsass.yml
```

To get the raw URL for any rule, open it on GitHub, click **Raw**, and replace `master` with your tag.

List recent release tags:

```bash
git ls-remote --tags https://github.com/SigmaHQ/sigma.git | tail -5
```

Record the source tag in your commit message, e.g. `Add LSASS ProcDump rule (SigmaHQ r2026-07-01)`.

---

## 4. Validating rules

Lint the rules before converting them:

```bash
sigma check rules/
```

Expected output: `Found 0 errors, 0 condition errors and 0 issues.`

---

## 5. Converting rules

### Single rule

```bash
sigma convert -t splunk \
  -p splunk_windows \
  -p pipelines/splunk_pipeline.yml \
  rules/endpoint/windows/credential_access/lsass_procdump.yml
```

| Part | Meaning |
|---|---|
| `sigma convert` | Translate Sigma rules into a backend query language |
| `-t splunk` | **Target** backend, which outputs Splunk SPL |
| `-p splunk_windows` | Built-in **pipeline** that maps Windows log sources to Splunk |
| `-p pipelines/splunk_pipeline.yml` | Our **custom pipeline** (sources, EventCodes, field names). `-p` can be repeated |
| `rules/...yml` | **Input**: one or more rule files or directories. Always comes last |
| `\` | Bash line continuation. Nothing may follow it on the line, not even a space |

Example output:

```
source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 process_command_line="* -ma *" OR ... process_command_line IN ("* ls*", "* keyiso*", "* samss*")
```

### All rules in a directory

```bash
sigma convert -t splunk -p splunk_windows -p pipelines/splunk_pipeline.yml rules/
```

### Write to a file

`-o` does not create directories, so create the output folder first:

```bash
mkdir -p output
sigma convert -t splunk -p splunk_windows -p pipelines/splunk_pipeline.yml \
  -o output/detections.spl rules/
```

### Splunk `savedsearches.conf` (for deployment)

```bash
sigma convert -t splunk -p splunk_windows -p pipelines/splunk_pipeline.yml \
  -f savedsearches -o output/savedsearches.conf rules/
```

This produces one stanza per rule, using the rule's `title` as the stanza name and its
`description` and `search`. Drop the file into a Splunk app's `local/` directory to deploy.

### Output formats

| `-f` value | Output |
|---|---|
| `default` | Plain SPL, one query per rule |
| `savedsearches` | A `savedsearches.conf` file |
| `data_model` | `tstats` queries against CIM data models (use with `-p splunk_cim`) |

---

## 6. Custom pipeline (`pipelines/splunk_pipeline.yml`)

The custom pipeline adapts generic Sigma rules to how **our** Splunk ingests data.

```yaml
name: DaC Custom Splunk Pipeline
priority: 20
transformations:
  - id: sysmon_process_creation
    type: add_condition
    conditions:                      # fields ADDED to the query
      source: "WinEventLog:Microsoft-Windows-Sysmon/Operational"
      EventCode: 1
    rule_conditions:                 # WHICH rules this applies to
      - type: logsource
        category: process_creation
        product: windows

  - id: field_mapping
    type: field_name_mapping         # Sigma field → Splunk field
    mapping:
      Image: process
      ParentImage: parent_process
      CommandLine: process_command_line
      User: user
      OriginalFileName: original_file_name
```

Key points:

- For `add_condition`, **`conditions` is the `field: value` map that gets added** to the query.
  The filter that decides which rules it applies to is **`rule_conditions`**.
- A transformation with no `rule_conditions` applies to every rule.
- `priority` sets the order pipelines run in, lowest first. Built-in pipelines use 20 to 25.

### Adding a new log source

1. Copy an `add_condition` block.
2. Set `conditions` to the Splunk `source`/`sourcetype`/`index`/`EventCode` for that data.
3. Set `rule_conditions` to the Sigma `logsource` it represents (`product`, `category`, `service`).
4. Convert a rule that uses that log source and check the SPL.

---

## 7. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Invalid value for 'INPUT...': Path '...' does not exist` | The rule file isn't there, or you aren't in the repo root | `cd` to the repo root, check the path with `ls`, or download the rule (section 3) |
| `AddConditionTransformation.__init__() got an unexpected keyword argument '...'` | Wrong keys in an `add_condition` block, e.g. `detection:` | Use `conditions:` (fields to add) and `rule_conditions:` (filter). See section 6 |
| `Error: Invalid value for '--target'` / unknown target | Splunk backend not installed | `sigma plugin install splunk` |
| `Processing pipeline required by backend` | No `-p` given | Add `-p splunk_windows`. Only use `--without-pipeline` if you really want no pipeline |
| Fields in the output don't match your data | Missing field mapping | Add the field to `field_name_mapping` in the custom pipeline |
| `Could not open file 'output/...': No such file or directory` | Output folder doesn't exist | `mkdir -p output` |
| `sigma: command not found` | Virtualenv not active | `source ~/sigma-venv/bin/activate` |

---

## 8. Quick reference

```bash
source ~/sigma-venv/bin/activate
cd detection-as-code

sigma check rules/                                                          # lint
sigma convert -t splunk -p splunk_windows -p pipelines/splunk_pipeline.yml rules/        # convert all
mkdir -p output && \
sigma convert -t splunk -p splunk_windows -p pipelines/splunk_pipeline.yml \
  -f savedsearches -o output/savedsearches.conf rules/                     # build deployable conf
```
