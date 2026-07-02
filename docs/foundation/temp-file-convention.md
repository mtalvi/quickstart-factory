# Temp-File Namespace Scoping Convention

This document defines how factory skills use temp files to pass structured artifacts between pipeline stages. All temp files are scoped to a project-specific namespace directory to prevent collisions and enable resumability.

## Namespace Structure

All temp files live under a project-scoped directory:

```
/tmp/qs-<slug>/
```

**Example:** `/tmp/qs-spending-transaction-monitor/`

The `<slug>` is the lowercase, hyphenated project identifier — the same value used in PRD filenames (`data/prds/<slug>.md`) and repo names.

## Constructing the Scoped Path

Skills construct temp paths using the slug from their input (typically from the upstream spec or manifest):

```bash
QS_SLUG="spending-transaction-monitor"
QS_TMP="/tmp/qs-${QS_SLUG}"
mkdir -p "$QS_TMP"
```

The slug is derived from the quickstart name:
- Lowercase all characters
- Replace spaces with hyphens
- Remove special characters
- Example: `"Spending Transaction Monitor"` -> `spending-transaction-monitor`

Every skill that reads or writes temp files must use the `QS_TMP` prefix. Hardcoding `/tmp/` paths without the `qs-<slug>/` prefix is a bug.

## File Categories

Four categories of temp files exist within `/tmp/qs-<slug>/`:

### 1. Spec files

Generated during the Analyze phase, consumed by validators and implementers.

| File | Producing Skill | Purpose |
|------|----------------|---------|
| `discovery-spec.yaml` | rh-qs-discovery | Interview plan |
| `architecture-spec.yaml` | rh-qs-architect | Component bill of materials |
| `scaffold-spec.yaml` | rh-qs-scaffold | Repo structure plan |
| `implementation-spec.yaml` | rh-qs-implement | Endpoints, schemas, services |
| `deploy-spec.yaml` | rh-qs-deploy | Chart deps, values, Containerfiles |
| `security-spec.yaml` | rh-qs-security | Scan targets, severity thresholds |
| `update-spec.yaml` | rh-qs-update | Change type, affected files |

After validation, a refined variant is written:

```
<skill>-spec.yaml          → initial spec
<skill>-spec-refined.yaml  → after validator feedback
```

### 2. Handoff manifests

Output artifacts that the next pipeline stage consumes. These are the inter-skill contracts.

| File | Producer | Consumer |
|------|----------|----------|
| `architecture-spec.yaml` | rh-qs-architect | rh-qs-scaffold |
| `scaffold-manifest.yaml` | rh-qs-scaffold | rh-qs-implement |
| `implementation-manifest.yaml` | rh-qs-implement | rh-qs-deploy |
| `deploy-manifest.yaml` | rh-qs-deploy | rh-qs-security |
| `security-report.yaml` | rh-qs-security | rh-qs-debug-and-deploy |
| `deploy-state.yaml` | rh-qs-debug-and-deploy | rh-qs-document |
| `doc-manifest.yaml` | rh-qs-document | rh-qs-ship |

See [temp-file-contracts.md](temp-file-contracts.md) for the YAML schema of each handoff file.

Note: `architecture-spec.yaml` serves double duty — it is both the architect's spec file and the handoff manifest to scaffold.

### 3. Internal working files

Temp files used within a single skill's execution, not consumed by other skills.

| File Pattern | Skill | Purpose |
|-------------|-------|---------|
| `test-spec.yaml` | rh-qs-implement | Test inventory from test-writer |
| `test-results.yaml` | rh-qs-implement | Test run outcomes |
| `deploy-validation.yaml` | rh-qs-deploy | Deploy reviewer results |
| `qs-security-code.yaml` | rh-qs-security | Code scanner findings |
| `qs-security-containers.yaml` | rh-qs-security | Container scanner findings |
| `qs-security-helm.yaml` | rh-qs-security | Helm scanner findings |
| `qs-security-deps.yaml` | rh-qs-security | Dependency scanner findings |
| `qs-deploy-analysis.yaml` | rh-qs-debug-and-deploy | Deploy command analysis |
| `qs-e2e-results.yaml` | rh-qs-debug-and-deploy | E2E test results |
| `qs-update-analysis.yaml` | rh-qs-update | Change analysis |
| `qs-update-impact.yaml` | rh-qs-update | Impact assessment |
| `qs-handoff-state.yaml` | rh-qs-handoff | Pipeline state detection |
| `qs-handoff-gaps.yaml` | rh-qs-handoff | Gap analysis |
| `qs-kb-components.yaml` | rh-qs-extract-knowledge | Component pattern extraction |
| `qs-kb-deployment.yaml` | rh-qs-extract-knowledge | Deployment pattern extraction |
| `qs-kb-industry.yaml` | rh-qs-extract-knowledge | Industry pattern extraction |
| `qs-kb-security.yaml` | rh-qs-extract-knowledge | Security pattern extraction |
| `qs-kb-update-plan.yaml` | rh-qs-extract-knowledge | KB dedup/update plan |
| `qs-kb-extraction-report.yaml` | rh-qs-extract-knowledge | Extraction summary |

### 4. Debug and fix files

Per-resource debug artifacts that accumulate during the debug loop.

| File Pattern | Skill | Purpose |
|-------------|-------|---------|
| `qs-debug-{resource}.yaml` | rh-qs-debug-and-deploy | Root-cause analysis per resource |
| `qs-fix-{resource}.yaml` | rh-qs-debug-and-deploy | Fix attempt records per resource |

Debug and fix files append `attempt_N` keys — they never overwrite previous attempts. This lets the debugger review what was already tried before proposing a new fix.

## Directory Layout Example

A fully populated temp directory mid-pipeline looks like:

```
/tmp/qs-spending-transaction-monitor/
├── discovery-spec.yaml
├── architecture-spec.yaml              # Also the handoff to scaffold
├── architecture-spec-refined.yaml
├── scaffold-spec.yaml
├── scaffold-manifest.yaml              # Handoff to implement
├── implementation-spec.yaml
├── implementation-spec-refined.yaml
├── test-spec.yaml                      # Internal: test inventory
├── test-results.yaml                   # Internal: test outcomes
├── implementation-manifest.yaml        # Handoff to deploy
├── deploy-spec.yaml
├── deploy-validation.yaml              # Internal: reviewer results
├── deploy-manifest.yaml                # Handoff to security
├── qs-security-code.yaml               # Internal: scanner output
├── qs-security-containers.yaml
├── qs-security-helm.yaml
├── qs-security-deps.yaml
├── security-report.yaml                # Handoff to debug-and-deploy
├── qs-deploy-analysis.yaml             # Internal: deploy analysis
├── deploy-state.yaml                   # Handoff to document
├── qs-debug-redis.yaml                 # Internal: per-resource debug
├── qs-fix-redis.yaml
├── qs-e2e-results.yaml                 # Internal: E2E results
├── doc-manifest.yaml                   # Handoff to ship
└── qs-kb-extraction-report.yaml        # Internal: KB extraction
```

## Cleanup Policy

### During execution

Skills do NOT clean up temp files during execution. Files accumulate throughout the pipeline run. This is intentional — it creates an audit trail and supports resumability.

### On reboot

`/tmp/` is cleared by the operating system on reboot. This means:
- Temp files do not survive reboots
- Long-running pipelines that span reboots will lose intermediate state

### Persistent copies

To mitigate the reboot risk, key reports are copied to persistent locations within the project repo:

| Temp File | Persistent Copy | Producing Skill |
|-----------|----------------|-----------------|
| `security-report.yaml` | `{project}/.rhoai/security-report.yaml` | rh-qs-security |
| `deploy-state.yaml` | `{project}/.rhoai/deploy-report.yaml` | rh-qs-debug-and-deploy |
| `qs-kb-extraction-report.yaml` | `{project}/.rhoai/kb-extraction-report.yaml` | rh-qs-extract-knowledge |

The `.rhoai/` directory in the project repo is gitignored by default. Skills that need post-pipeline access to reports read from `.rhoai/` rather than `/tmp/`.

### Manual cleanup

To reset a project's temp state:

```bash
rm -rf /tmp/qs-<slug>/
```

This is safe — it only removes temp files for one quickstart, not others.

## Concurrency

### Different quickstarts

Different quickstarts use different slugs, so they get separate temp directories. Running `rh-qs-implement` for `spending-transaction-monitor` and `supply-chain-agent` simultaneously is safe — they write to `/tmp/qs-spending-transaction-monitor/` and `/tmp/qs-supply-chain-agent/` respectively.

### Same quickstart

Running the same quickstart concurrently (e.g., two agents both running `rh-qs-deploy` for `spending-transaction-monitor`) is NOT safe. Both would write to the same `/tmp/qs-spending-transaction-monitor/deploy-spec.yaml`, causing race conditions.

This is a known constraint. The factory is designed for one pipeline execution per quickstart at a time.

## Resumability

When a skill starts, it should check whether its temp directory already contains artifacts from a previous run:

```
1. Check if /tmp/qs-<slug>/ exists
2. If it exists, check for artifacts from this skill and upstream skills
3. If upstream handoff manifests exist:
   → Offer to resume from current stage
   → Or start fresh (rm -rf /tmp/qs-<slug>/)
4. If this skill's own spec exists:
   → Offer to reuse the existing spec
   → Or regenerate
```

This enables the "pick up where you left off" pattern, especially useful when a skill fails mid-execution and the user restarts it.

For full pipeline resumability across reboots, `rh-qs-handoff` reconstructs state from the project repo (persistent `.rhoai/` files, existing code, PRD) rather than relying on temp files.

## Relationship to Other Foundation Docs

- **[temp-file-contracts.md](temp-file-contracts.md)** — defines the YAML schema for each handoff manifest
- **[spec-as-contract.md](spec-as-contract.md)** — defines the spec file format (category 1 above)
- **[skill-directory-structure.md](skill-directory-structure.md)** — each skill's `output-templates.md` defines the format of its temp file outputs
