# Detection as Code

This is the final installment of the current series - setting up the infra of my purple team lab.
The goal is/was the following: 
1. Allow practicing offensive techniques in the GOAD environment (+ custom additions)
2. Observe the resulting telemetry (or lack thereof)
3. Write detections for it (and implement required telemetry),
4. Lint, validate and deploy the detection to kibana

As such, the [`ares-dac`](https://github.com/manfred6/ares-dac/tree/main) repository is the source of truth with kibana merely acting as an executor, validator and data storage system.
The focus hereby is to practice offensive techniques and write useful detections rather than establishing a universal enterprise-grade detection framework.

As of this writing, the project implements the following features:

- filesystem-based detection discovery
- YAML metadata linting
- stable UUIDs for rule identity
- Query validation against the real Elasticsearch cluster
- generated JSON and Markdown artifacts
- MITRE ATT&CK enrichment
- a small internal Kibana Detection Engine client
- ownership tagging for DaC-managed rules
- desired-state reconciliation against Kibana

This project will be the subject of this article.

## Rule layout

Each detection lives in its own directory:

```text
rules/
  \_ windows/
    \_ powershell/
      \_ iwr-iex/
            |_ metadata.yml
            |_ rule.esql
            (rule.kql, etc..)
```
> note only one rule per type can be present in each rule directory

`metadata.yml` contains the rule metadata and Kibana configuration:

```yaml
uuid: "1ac562bf-8d3b-4aef-b155-0f20a76ab386"
name: "PowerShell IWR directly into IEX"
description: >
  Detects PowerShell download-and-execute behavior using
  Invoke-WebRequest and Invoke-Expression.
date:
  created: "2026-09-16"
  modified: "2026-09-19"
types:
  - esql
indices:
  - logs-windows.powershell_operational-*
author: manfred6
references: []
mitre:
  attack:
    - T1059.001
    - T1105
kibana:
  enabled: true
  interval: 5m
  from: now-6m
  to: now
  severity: medium
  risk_score: 47
```

The `uuid` is stable and maps directly to kibanas `rule_id`.

## Workflow

Once I identify an offensive tactic i want to practice, I manually execute it and work with my [`ares-infa`](https://github.com/manfred6/ares-infra) project to ensure the required telemetry is recorded into elastic.
I then manually create detections and see if I can detect what im doing in different ways / at different points.
Once ive determined good detection methodologies for each tactic, I manually create a rule in this repository in the [`./rules`](https://github.com/manfred6/ares-dac/tree/main/rules) folder structure.
The uuid can be generated using the [`uuid.sh`](https://github.com/manfred6/ares-dac/blob/main/scripts/uuid.sh) helper as follows:

```bash
bash scripts/uuid.sh <rule_folder_path>
```

Once the rule is created, the CI process kicks off upon push / pull using CI. This process looks as follows:

```text
rule added to rules/
    \_ lint metadata
      \_ validate query against elasticsearch
        \_ generate artifacts (meta.json, README.md)
          \_ build desired kibana ruleset
            \_ compare with current kibana ruleset
              \_ reconcile desired and current
```

These stages will now be discussed in more detail.


## Linting

`metadata.yml` is checked against the local schema before deployment. This schema is defined [`here`]().

## Query validation

`ES|QL` is validated against the elasticsearch cluster using the official Python SDK as follows:
```python
client.esql.query(
    query=f"{query.rstrip()}\n| LIMIT 0"
)
```

This catches syntax errors and problems the target elasticsearch instance cannot resolve.
Budget syntax validation which comes with perks such as catching missing fields and indices.

## Artifacts

The pipeline generates:
```text
artifacts/
    \_ README.md
    \_ meta.json
    \_ enterprise-attack.json
```
while `artifacts/enterprise-attack.json` can be pulled manually using [`attck-yoink.sh`](https://github.com/manfred6/ares-dac/blob/main/scripts/attck-yoink.sh):

```bash
bash scripts/attck-yoink.sh
```

`meta.json` is the machine-readable rule manifest, while `README.md` is a generated status table for linting and validation.

## MITRE ATT&CK

Rules only store ATT&CK IDs in `metadata.yml`:
```text
mitre:
  attack:
    - T1059.001
    - T1105
    ...
```

The MITRE python library enriches these from the ATT&CK STIX bundle in artifacts and converts them into a native kibana threat format.

## Kibana reconciliation

Rules deployed by [`ares-dac`](https://github.com/manfred6/ares-dac/tree/main) are tagged with `managed-by:ares-dac`. Only those rules are managed by the reconciler, manually added ones without this tag are thereby ignored.
This has the advantage that I can experiment with local rules without them being nuked by CI, or them being read into CI in a testing / incomplete state.

The reconciliation logic works as follows:

| State | Result |
|:----:|:------:|
| UUID missing in kibana | create |
| UUID with modified payload | update |
| UUID and payload match | noop |
| Kibana UUID missing from manifest | delete |

The desired payload itself defines which fields DaC owns. Adding fields under `kibana:` in `metadata.yml` makes those settings part of desired state.
Fields generated by kibana such as object IDs, revisions, timestamps, and execution state are ignored.

## Safety

If linting or validation fails, reconciliation is skipped.
This prevents a broken or incomplete manifest from being interpreted as intentional rule deletion.
Only rules tagged with `managed-by:ares-dac` are eligible for pruning, so prebuilt or manually-created rules are left alone.

## Configuration

Credentials and endpoints are supplied through environment variables:

```env
ELASTIC_URL=https://elasticsearch.example:9200
ELASTIC_API_KEY=...

KIBANA_URL=https://kibana.example:5601
KIBANA_API_KEY=...
```
> `.env` is kept out of vcs via `.gitignore`

The custom ARES root CA is used for TLS verification, and stored in `artifacts/ares.crt`. Gotta update this for yourself.

The code itself runs and is developed using a `venv` as follows:

```bash
bash scripts/venv.sh
```

## TODO

- Sigma compilation
- KQL, Lucene, EQL deployment
- TP/FP tests
- regression testing

---

