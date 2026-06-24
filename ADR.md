# ADR-001: Upgrade Factory Skills Into Production Ready

```
Status: Proposed
Date: 2026-06-24
Authors: Matan Talvi
```

## Context

The Quickstart Factory currently operates a 7-stage pipeline with 10 skills (7 pipeline + 3 utility). Each skill is a flat `SKILL.md` that the agent reads and follows directly — no subagent orchestration, no feedback loops, no validation beyond `make lint && make test`, no hooks, and no knowledge base.

Production-grade AI agent skill architectures demonstrate significantly more advanced patterns: subagent orchestration with parallel execution, spec-as-contract workflows, bounded feedback loops, namespace-scoped security hooks, living knowledge bases with scored retrieval, temp-file handoff between phases, and reasoning guardrails. Output quality is measurably higher because errors are caught in validation loops before the user ever sees them.

This ADR proposes upgrading all factory skills to this level of sophistication, while preserving the factory's existing strengths (agentskills.io spec compliance, multi-client portability, Makefile-driven validation, `skill-validator` CI).

Additionally, this ADR introduces 3 new pipeline skills and 2 new utility skills to close gaps in the current lifecycle.

## Decision

### New Pipeline (9 stages + 6 utilities)

```
MAIN PIPELINE (new quickstarts):

 1. rh-qs-discovery         → PRD
 2. rh-qs-architect         → Design doc
 3. rh-qs-scaffold          → GitHub repo
 4. rh-qs-implement         → Working code  (test subagent loop)
 5. rh-qs-deploy            → Helm + compose (deploy review subagent loop)
 6. rh-qs-debug-and-deploy  → Deployed + validated on OpenShift cluster  [NEW]
 7. rh-qs-security          → Security verified                          [NEW]
 8. rh-qs-document          → README + docs
 9. rh-qs-ship              → PR + blog

UTILITY SKILLS:

 - rh-qs-gh-backlog-reader     (existing)
 - rh-qs-gh-issue-creator      (existing)
 - rh-qs-pipeline-grooming     (existing)
 - rh-qs-update                (update released quickstarts)              [NEW]
 - rh-qs-handoff               (continue partial implementations)         [NEW]
```

---

## Architectural Patterns (All Skills)

Every factory skill will be restructured to follow 8 architectural patterns.

### Pattern 1: Subagent Orchestration

Each skill's `SKILL.md` becomes an **orchestrator** that delegates heavy work to subagent prompts in a `subagents/` directory.

**Critical design rule:** The main agent must NOT read subagent prompt files. It passes them by file path to the Task/Agent tool. This saves 60–70% of context window.

**Directory structure per skill:**

```
rh-qs-<name>/
├── SKILL.md                          # Orchestrator instructions
├── reasoning-guardrails.md           # Concern areas for organic self-checking
├── spec-template.md                  # YAML spec structure definition
├── output-templates.md               # Output format definitions
├── subagents/
│   ├── README.md                     # Subagent index and roles
│   └── <role>-prompt.md              # One file per subagent
├── knowledge-base/                   # If applicable
│   └── *.md
└── references/
    └── *.md
```

### Pattern 2: Spec-as-Contract

Before any skill modifies files, it generates a YAML spec describing *what* it will do, validates the spec with parallel subagents, refines it based on validation results, and only then hands it to an implementer subagent.

```
Analyze → Generate Spec → Validate Spec (parallel) → Refine → Implement → Post-validate
```

Each spec is written to a temp file (`/tmp/<skill>-spec.yaml`) following a structure defined in `spec-template.md`.

### Pattern 3: Feedback Loops with Bounded Iteration

Every skill that produces artifacts validates its output and iterates on failures, with a hard maximum to prevent infinite loops.

**Standard loop protocol (all skills):**

```
1. Run validation
2. If failures:
   a. Analyze errors
   b. Fix root cause (not symptoms)
   c. Re-run validation
3. Max N iterations (skill-specific, typically 3)
4. If still failing after max:
   a. Document remaining failures in temp file
   b. Ask user: proceed, provide guidance, or stop
5. Only advance to next phase when validation passes
```

### Pattern 4: Temp-File Handoff

Skills pass structured artifacts to each other via temp files, not conversation context.

| Transition | Temp File | Format |
|-----------|----------|--------|
| Discovery → Architect | `data/prds/<slug>.md` | Markdown (existing) |
| Architect → Scaffold | `/tmp/architecture-spec.yaml` | YAML |
| Scaffold → Implement | `/tmp/scaffold-manifest.yaml` | YAML |
| Implement → Deploy | `/tmp/implementation-manifest.yaml` | YAML |
| Deploy → Debug-and-Deploy | `/tmp/deploy-manifest.yaml` | YAML |
| Debug-and-Deploy → Security | `/tmp/deploy-state.yaml` | YAML |
| Security → Document | `/tmp/security-report.yaml` | YAML |
| Document → Ship | `/tmp/doc-manifest.yaml` | YAML |

### Pattern 5: Reasoning Guardrails

Each skill gets a `reasoning-guardrails.md` file — NOT a checklist, but a set of concern areas the agent should reason about organically and self-check periodically during execution.

### Pattern 6: Hooks

Project-level hooks for safety and automation, gating destructive commands, enforcing namespace-scoped cluster operations, and auto-formatting edited files.

### Pattern 7: Knowledge Base

A shared, tagged, scored knowledge base of reusable patterns mined from completed quickstarts. Each KB file has structured frontmatter with a Chain of Density summary for efficient retrieval without loading full files.

### Pattern 8: Multi-Layer Validation

| Layer | When | What | Mechanism |
|-------|------|------|-----------|
| **Pre-implementation** | After spec, before code | Spec correctness: valid chart versions, coherent schemas, valid model IDs | Parallel validator subagents per component |
| **Post-implementation** | After code, before deploy | Code quality + functional: lint, test, no placeholders, env vars match | Validator subagent + `make lint && make test` |
| **Deploy-time** | After deploy to cluster | Cluster health: resources running, endpoints responding | Health scanner + debug loop |
| **Security** | After deploy verified | Compliance: no secrets in code, non-root containers, minimal SCCs, no CVEs | 5 parallel security scanner subagents |

---

## Skill-by-Skill Specifications

### 1. rh-qs-discovery (Enhanced)

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `prd-structurer-prompt.md` | Convert unstructured notes/docs into PRD sections | User-uploaded documents, conversation notes | Structured PRD sections in JSON |
| `backlog-matcher-prompt.md` | Check if idea duplicates existing backlog issues | Idea summary + backlog data | Match report (duplicate/similar/unique) |

**Spec file:** `/tmp/discovery-spec.yaml` (interview plan based on initial input)

**Loop:** Validate → confirm with user (max 2 refinement rounds)

**Guardrails:** Scope creep (don't invent requirements), technology bias (don't pre-decide stack), GPU assumptions.

---

### 2. rh-qs-architect (Enhanced)

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `architecture-analyzer-prompt.md` | Parse PRD into feature vector | PRD markdown | JSON feature vector (AI capabilities, data needs, UI/API, scale) |
| `chart-selector-prompt.md` | Match features to ai-architecture-charts | Feature vector + KB | Component bill of materials with rationale |
| `diagram-generator-prompt.md` | Generate Mermaid architecture diagram | Bill of materials | Mermaid diagram code |

**Spec file:** `/tmp/architecture-spec.yaml`

**Knowledge base:** `knowledge-base/components/`, `knowledge-base/deployment-types/`, `knowledge-base/industries/`

**KB scoring:** `knowledge-scorer-prompt.md` scores KB files by relevance using component (+10), deployment type (+5), architecture (+5), industry (+3) tag matching. Returns XML summaries from frontmatter `summary:` fields without loading full files.

**Loop:** Chart selector → validate charts exist (helm search, ArtifactHub) → refine (max 2 iterations)

**Guardrails:** Model sizing, GPU dependency, data residency, chart compatibility, scope creep.

---

### 3. rh-qs-scaffold (Enhanced)

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `structure-generator-prompt.md` | Generate directory tree and file skeletons | Architecture spec | File list with skeleton contents |
| `ci-generator-prompt.md` | Generate GitHub Actions workflows | Component list from spec | CI/CD YAML files |
| `scaffold-validator-prompt.md` | Validate generated scaffold | Generated files | Validation report (READY/BLOCKED) |

**Spec file:** `/tmp/scaffold-spec.yaml` (repo name, packages to create, CI jobs, linting config)

**Output manifest:** `/tmp/scaffold-manifest.yaml` (lists all created files/paths for rh-qs-implement)

**Loop:** Generate → validate (files parseable, paths consistent, no placeholder strings) → fix (max 2 iterations)

---

### 4. rh-qs-implement (Major Restructure)

This is the most significant change. The implementation skill gains a **two-agent feedback loop** between a test-writing subagent and the main agent.

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `test-writer-prompt.md` | Generate tests from PRD + architecture | PRD, design doc, scaffold manifest | Test files (pytest, vitest) |
| `backend-implementer-prompt.md` | Implement FastAPI backend | Implementation spec | Backend code |
| `frontend-implementer-prompt.md` | Implement React frontend | Implementation spec | Frontend code |
| `db-schema-prompt.md` | Generate SQLAlchemy models + Alembic migration | Implementation spec | DB package code |
| `implementation-validator-prompt.md` | Post-implementation validation | All generated code | Validation report |

**Spec file:** `/tmp/implementation-spec.yaml` (endpoints, schemas, services, DB models, UI routes)

**The Test Loop (core innovation):**

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  test-writer subagent                                       │
│  (reads PRD + architecture + scaffold manifest)             │
│  → generates test files                                     │
│  → writes to /tmp/test-spec.yaml (test inventory)           │
│                                                             │
│          ↓ test files                                       │
│                                                             │
│  main agent (orchestrator)                                  │
│  → runs tests: make test                                    │
│  → tests fail                                               │
│  → main agent fixes CODE ONLY (not tests)                   │
│  → re-runs tests                                            │
│  → writes results to /tmp/test-results.yaml                 │
│                                                             │
│          ↓ if tests still fail after code fixes              │
│                                                             │
│  test-writer subagent (resumed)                             │
│  → reads /tmp/test-results.yaml                             │
│  → reviews failures: are tests wrong or is code wrong?      │
│  → adjusts tests ONLY if they were over-specified or wrong  │
│  → writes updated tests + rationale                         │
│                                                             │
│          ↓ back to main agent                               │
│                                                             │
│  main agent re-runs tests, fixes code if needed             │
│                                                             │
│  MAX 3 full loops. After 3: document failures, ask user.    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**The critical rule:** The main agent fixes CODE to make tests pass. It NEVER modifies tests. Only the test-writer subagent can adjust tests, and only when it determines the tests themselves were wrong (not the code).

**Parallelism:** Backend + frontend implementer subagents run in parallel. DB schema runs first (dependency).

**Output manifest:** `/tmp/implementation-manifest.yaml` (lists endpoints, DB models, packages, test coverage)

**Guardrails:** Vertical slice (thinnest path), hardcoded values, error handling, async consistency, security.

---

### 5. rh-qs-deploy (Enhanced)

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `helm-wiring-prompt.md` | Generate Chart.yaml dependencies + values.yaml | Deploy spec, implementation manifest | Helm chart files |
| `compose-generator-prompt.md` | Generate/update compose.yml | Deploy spec | compose.yml |
| `containerfile-generator-prompt.md` | Generate multi-stage Containerfiles | Package list from implementation manifest | Containerfiles per package |
| `deploy-reviewer-prompt.md` | Validate all deployment artifacts | All deploy files | Validation report (READY/BLOCKED/PARTIAL) |

**Spec file:** `/tmp/deploy-spec.yaml` (chart dependencies, values overrides, compose services, Containerfile specs)

**The Deploy Review Loop:**

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  main agent generates deploy artifacts                   │
│          ↓                                               │
│  deploy-reviewer subagent validates:                     │
│  - Chart.yaml dependencies valid (helm search)           │
│  - values.yaml internally consistent                     │
│  - compose.yml services match Helm services              │
│  - Containerfiles build (podman build --dry-run)         │
│  - helm lint passes                                      │
│  - helm template renders both modes (remote + on-cluster)│
│  → writes /tmp/deploy-validation.yaml                    │
│          ↓                                               │
│  if BLOCKED: main agent fixes → re-validate              │
│  MAX 3 iterations                                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Output manifest:** `/tmp/deploy-manifest.yaml` (charts used, services, routes, env vars, validation status)

**Guardrails:** Chart version compatibility, secret exposure, GPU resource configuration, image registry accessibility.

---

### 6. rh-qs-debug-and-deploy [NEW]

**Category:** `core/skills/deployment/rh-qs-debug-and-deploy/`

**Purpose:** Take source code and YAML/Helm files from rh-qs-deploy, deploy to an OpenShift cluster, iteratively debug and fix failures, and run E2E verification from TEST-PLAN.md.

**Subagents:**

| Subagent | Phase | Role | Input | Output |
|----------|-------|------|-------|--------|
| `cluster-access-validator-prompt.md` | 1a | Validate login, namespace, permissions | Namespace name | Access report |
| `project-analyzer-prompt.md` | 1b | Discover deploy commands and dependencies | Project path | `/tmp/qs-deploy-analysis.yaml` |
| `health-scanner-prompt.md` | 3, 4c | Scan all resources, produce health snapshot | Namespace, expected resources | `/tmp/qs-deploy-state.yaml` |
| `resource-debugger-prompt.md` | 4a | Root-cause analysis per failing resource | Resource name/kind, namespace | `/tmp/qs-debug-{resource}.yaml` |
| `fix-applier-prompt.md` | 4b | Validate fix against Red Hat best practices, apply | Debug report | `/tmp/qs-fix-{resource}.yaml` |
| `e2e-tester-prompt.md` | 5 | Run TEST-PLAN.md end-to-end verification | Project path, namespace | `/tmp/qs-e2e-results.yaml` |

**Workflow:**

```
Phase 1a: cluster-access-validator → verify login + namespace + policy
Phase 1b: project-analyzer → discover deploy commands + dependency order
Phase 2:  main agent runs deploy commands from analysis
Phase 3:  health-scanner → /tmp/qs-deploy-state.yaml

Phase 4:  DEBUG LOOP (per unhealthy resource, dependency order):

          ┌──────────────────────────────────────────────────┐
          │ 4a. resource-debugger → root cause + proposed fix│
          │ 4b. fix-applier → validate + apply + re-deploy   │
          │ 4c. health-scanner → re-scan resource            │
          │                                                  │
          │ MAX 3 attempts per resource                      │
          │ On attempt 3 failure: ask user                   │
          │   - skip resource                                │
          │   - provide guidance                             │
          │   - stop deployment                              │
          └──────────────────────────────────────────────────┘

Phase 5:  e2e-tester → run TEST-PLAN.md (no re-entry to debug on E2E fail)
Phase 6:  Final report → copy to {project}/.rhoai/deploy-report.yaml
```

**Fix boundary rules:**

- **Auto-apply:** Config/infra changes (ports, SCCs, resource limits, env vars, image tags, probes, volume mounts, init containers, route configs, PVC sizes)
- **Auto-apply:** OpenShift-specific source changes (inside `if openshift_mode` blocks, OCP config files)
- **Ask user:** Non-OpenShift source code changes (application logic, business logic, data flow)

**Attempt tracking:** Debug and fix files append `attempt_N` keys — never overwrite previous attempts. The debugger reviews previous attempts before proposing a new fix to avoid repeating failed approaches.

**Output:** `/tmp/qs-deploy-state.yaml` (final health state), `{project}/.rhoai/deploy-report.yaml` (persistent copy)

**Guardrails:** Namespace isolation (every `oc`/`helm` command must use `-n <namespace>`), don't change application intent, dependency-order debugging (fix leaves first then work up), max 3 attempts per resource.

---

### 7. rh-qs-security [NEW]

**Category:** `core/skills/security/rh-qs-security/`

**Purpose:** Comprehensive security verification of the quickstart before documentation and shipping. Runs after rh-qs-debug-and-deploy to verify the deployed and working quickstart meets security standards.

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `code-security-scanner-prompt.md` | Scan for hardcoded secrets, injection vulnerabilities, insecure patterns | Source code | `/tmp/qs-security-code.yaml` |
| `container-security-reviewer-prompt.md` | Verify Containerfiles (non-root, minimal base, no secrets baked in) | Containerfiles | `/tmp/qs-security-containers.yaml` |
| `helm-security-reviewer-prompt.md` | Verify Helm charts (RBAC, NetworkPolicies, SCCs, secret management) | Helm chart | `/tmp/qs-security-helm.yaml` |
| `dependency-scanner-prompt.md` | Check Python/Node dependencies for known CVEs | pyproject.toml, package.json | `/tmp/qs-security-deps.yaml` |
| `cluster-security-auditor-prompt.md` | Audit running deployment (SCCs, network exposure, secret mounting) | Namespace | `/tmp/qs-security-cluster.yaml` |

**Workflow:**

```
Phase 1: Parallel scan (all 5 subagents run simultaneously)
         → 5 temp files with findings

Phase 2: Main agent aggregates findings into severity categories:
         - CRITICAL: Must fix before ship (secrets in code, root containers,
                     CVEs with known exploits)
         - HIGH:     Should fix (overly permissive RBAC, missing NetworkPolicy)
         - MEDIUM:   Recommended (dependency updates, hardening suggestions)
         - LOW:      Informational (best practice deviations)

Phase 3: FIX LOOP for CRITICAL + HIGH findings:

         ┌──────────────────────────────────────────┐
         │ main agent applies fixes                 │
         │ re-runs relevant scanner subagent        │
         │ MAX 3 iterations                         │
         │ If CRITICAL remains: block ship, ask user│
         │ If only HIGH remains: warn, allow proceed│
         └──────────────────────────────────────────┘

Phase 4: Generate security report → /tmp/qs-security-report.yaml
         Persist to {project}/.rhoai/security-report.yaml
```

**Security checklist areas:**

- No API keys, tokens, passwords in source code or Helm values
- Containers run as non-root (USER 1001)
- Minimal base images (python:slim, UBI, alpine)
- No `privileged: true` or `allowPrivilegeEscalation: true` without justification
- SCCs are minimum necessary (prefer `restricted-v2`, document if `anyuid` required)
- Routes use TLS edge termination
- CORS is not `*` in production config
- Dependencies have no known critical CVEs
- Secrets use Kubernetes Secrets, not ConfigMaps
- NetworkPolicies restrict pod-to-pod traffic

**Guardrails:** False positive filtering (some warnings are acceptable for demos), compliance context (HIPAA/PCI if PRD specifies), GPU security contexts (some GPU workloads require elevated permissions — document the justification).

---

### 8. rh-qs-document (Enhanced)

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `readme-generator-prompt.md` | Generate README from implementation + deploy manifests | All manifests + security report | README.md draft |
| `doc-validator-prompt.md` | Validate all documented commands exist and work | README + Makefile + repo | Validation report |

**Loop:** Generate → validate (every `make` target exists, every URL well-formed, diagram matches components) → fix (max 2 iterations)

---

### 9. rh-qs-ship (Enhanced)

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `pr-body-generator-prompt.md` | Generate PR description from all manifests | All temp manifests + security report | PR body markdown |
| `blog-writer-prompt.md` | Generate blog post draft | PRD + architecture + README | Blog post markdown |

**Loop:** Create PR → wait for CI → if CI fails, diagnose + fix → re-push (max 3 iterations)

---

### 10. rh-qs-update [NEW]

**Category:** `core/skills/lifecycle/rh-qs-update/`

**Purpose:** Update existing, released quickstarts. Handles image tag bumps, dependency updates, chart version updates, security patches, and feature additions.

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `change-analyzer-prompt.md` | Analyze what needs updating and scope the change | Repo + update request | `/tmp/qs-update-analysis.yaml` |
| `impact-assessor-prompt.md` | Determine which pipeline stages need re-running | Update analysis | `/tmp/qs-update-impact.yaml` |
| `update-applier-prompt.md` | Apply the updates to source files | Update analysis | Modified files |
| `update-validator-prompt.md` | Validate updates don't break existing functionality | Modified files | Validation report |

**How it fits in the pipeline flow:**

```
User: "Update the spending-transaction-monitor to use Llama 3.3"

rh-qs-update analyzes the change:
  → change-analyzer: "Image tag update + values.yaml + env config"
  → impact-assessor: "Re-run stages 5–9 (deploy through ship)"

rh-qs-update applies the change:
  → update-applier: modifies files
  → update-validator: make lint + make test pass

Then hands off to the pipeline at the detected re-entry point:
  → rh-qs-deploy         (re-validate Helm)
  → rh-qs-debug-and-deploy (re-deploy to cluster)
  → rh-qs-security       (re-scan)
  → rh-qs-document       (update README if needed)
  → rh-qs-ship           (create PR for update)
```

**Update types and re-entry points:**

| Update Type | Example | Re-entry Stage |
|-------------|---------|---------------|
| Image tag bump | `vLLM 0.6 → 0.7` | 5 (deploy) |
| Chart version bump | `llama-stack 0.7.3 → 0.8.0` | 5 (deploy) |
| Dependency update | `FastAPI 0.110 → 0.115` | 4 (implement: re-run tests) |
| Security patch | CVE fix in base image | 5 (deploy) + 7 (security) |
| Feature addition | Add new endpoint | 4 (implement) |
| Configuration change | Switch model size | 5 (deploy) |

**Guardrails:** Backward compatibility, semantic versioning awareness, test regression detection.

---

### 11. rh-qs-handoff [NEW]

**Category:** `core/skills/lifecycle/rh-qs-handoff/`

**Purpose:** Assess a partially implemented quickstart and continue from wherever it left off. The factory detects the current state and routes to the appropriate pipeline stage.

**Subagents:**

| Subagent | Role | Input | Output |
|----------|------|-------|--------|
| `state-detector-prompt.md` | Analyze repo and determine completion state per stage | Repo path | `/tmp/qs-handoff-state.yaml` |
| `gap-analyzer-prompt.md` | Identify specific gaps within each incomplete stage | State report + repo | `/tmp/qs-handoff-gaps.yaml` |
| `context-reconstructor-prompt.md` | Reconstruct PRD/design from existing code (if missing) | Repo code | Reconstructed PRD or design doc |

**Workflow:**

```
Phase 1: state-detector scans the repo for evidence of each stage:

         ┌──────────────────────────────────────────────────────┐
         │ Stage 1 (Discovery):  data/prds/<slug>.md exists?   │
         │ Stage 2 (Architect):  data/designs/<slug>.md exists? │
         │ Stage 3 (Scaffold):   .github/workflows/ exists?     │
         │ Stage 4 (Implement):  packages/api/src/main.py?      │
         │ Stage 5 (Deploy):     deploy/helm/? compose.yml?     │
         │ Stage 6 (Debug):      .rhoai/deploy-report.yaml?     │
         │ Stage 7 (Security):   .rhoai/security-report.yaml?   │
         │ Stage 8 (Document):   README.md beyond placeholder?  │
         │ Stage 9 (Ship):       Open PR exists?                │
         └──────────────────────────────────────────────────────┘

Phase 2: gap-analyzer identifies specific missing pieces within
         each partially complete stage

Phase 3: If PRD or design doc is missing but code exists,
         context-reconstructor reverse-engineers the PRD/design
         from the existing implementation

Phase 4: Present handoff report to user:

         "This quickstart is at Stage 4 (Implement).
          Backend is 80% complete, frontend is missing.
          Stage 3 (Scaffold) is fully complete.
          Recommended: Continue from Stage 4 with frontend
          implementation."

Phase 5: User confirms → route to appropriate skill with
         reconstructed context in temp files
```

**Guardrails:** Don't redo completed work, preserve existing code patterns, match the existing codebase's style conventions.

---

## Hooks Specification

### `.cursor/hooks.json`

```json
{
  "version": 1,
  "hooks": {
    "afterFileEdit": [
      {
        "command": ".cursor/hooks/format-code.sh",
        "matcher": "\\.(py|ts|tsx)$"
      }
    ],
    "beforeShellExecution": [
      {
        "command": ".cursor/hooks/safety-gate.sh",
        "matcher": "gh repo delete|git push.*--force|rm -rf|helm uninstall|oc delete project",
        "failClosed": true
      },
      {
        "command": ".cursor/hooks/namespace-policy.sh",
        "matcher": "\\boc\\b|\\bkubectl\\b|\\bhelm\\b",
        "failClosed": true
      }
    ],
    "subagentStart": [
      {
        "command": ".cursor/hooks/log-subagent.sh"
      }
    ]
  }
}
```

**Hook 1: `format-code.sh`** — Auto-format Python (ruff) and TypeScript (prettier) after every file edit.

**Hook 2: `safety-gate.sh`** — Block destructive commands (repo delete, force push, rm -rf, helm uninstall, oc delete project). Returns `"permission": "ask"` so the user can approve.

**Hook 3: `namespace-policy.sh`** — Enforce namespace-scoped oc/kubectl/helm commands. Hard-deny missing `-n` flag and output redirects (`>`, `>>`).

**Hook 4: `log-subagent.sh`** — Log which subagents are spawned for debugging orchestration issues.

Each hook requires a test script (`.cursor/hooks/test-<name>.sh`) with comprehensive test cases covering verbs, namespaces, pipes, subshells, and edge cases.

---

## Knowledge Base Specification

### Location: `core/knowledge-base/` (shared across skills)

```
core/knowledge-base/
├── README.md                          # Index + scoring algorithm
├── components/
│   ├── llama-stack-patterns.md
│   ├── pgvector-patterns.md
│   ├── minio-patterns.md
│   ├── vllm-serving-patterns.md
│   ├── fastapi-patterns.md
│   └── react-frontend-patterns.md
├── deployment-types/
│   ├── helm-with-ai-architecture-charts.md
│   ├── compose-local-dev.md
│   └── github-actions-ci.md
├── industries/
│   ├── financial-services.md
│   ├── healthcare.md
│   ├── supply-chain.md
│   └── retail.md
└── security/
    ├── openshift-scc-patterns.md
    ├── container-hardening.md
    └── secret-management.md
```

**Frontmatter schema per KB file:**

```yaml
---
type: component-pattern | deployment-pattern | industry-pattern | security-pattern
components: [llama-stack, pgvector]
deployment_types: [helm]
industries: [financial-services]
source_quickstarts: [ai-supply-chain-agent, spending-transaction-monitor]
summary: "4-sentence Chain of Density summary for scored retrieval..."
---
```

**Growth mechanism:** After `rh-qs-ship` completes a quickstart, it (or a future extraction subagent) mines the repo for reusable patterns and creates/updates KB files with Chain of Density summaries.

**Scoring algorithm:** Component match (+10), deployment type (+5), architecture (+5), industry (+3). Knowledge scorer returns XML with summaries; full files loaded only for top matches.

---

## Updated AGENTS.md Routing

| User Request | Route To |
|--------------|----------|
| (all existing routes unchanged) | ... |
| "Deploy to cluster" / "Debug deployment" / "Fix deployment" | rh-qs-debug-and-deploy |
| "Security check" / "Security audit" / "Is it secure?" | rh-qs-security |
| "Update quickstart" / "Bump image" / "Upgrade dependencies" | rh-qs-update |
| "Continue this" / "Pick up where we left off" / "Finish this quickstart" | rh-qs-handoff |

---

## Implementation Phases

| Phase | Workstreams | Risk |
|-------|------------|------|
| **Phase 1: Foundation** | Hooks + guardrails files for all existing skills | Low — additive, no restructuring |
| **Phase 2: Validation** | Feedback loops + multi-layer validation for existing skills | Medium — modifies SKILL.md files |
| **Phase 3: Orchestration** | Subagent prompts + spec-as-contract for all existing skills (~25 subagent prompts) | High — core architecture change |
| **Phase 4: New Skills** | rh-qs-debug-and-deploy + rh-qs-security (~11 subagent prompts) | High — new deployment infrastructure |
| **Phase 5: Lifecycle** | rh-qs-update + rh-qs-handoff (~7 subagent prompts) | Medium — utility skills |
| **Phase 6: Knowledge Base** | KB creation, extraction from completed quickstarts, scoring | Low — grows over time |

**Totals:**

- New subagent prompts: ~43
- New skills: 3 (pipeline) + 2 (utility) = 5
- Total skills after upgrade: 15 (9 pipeline + 6 utility)

---

## Consequences

### Positive

- **Dramatically higher output quality** — errors caught in validation loops before users see them
- **Faster execution** — parallel subagents (backend + frontend, 5 security scanners simultaneously)
- **Context efficiency** — subagent delegation saves 60–70% context window
- **Auditability** — temp files create an audit trail of every decision and fix attempt
- **Growing intelligence** — knowledge base improves with each completed quickstart
- **Safety** — hooks prevent destructive operations and enforce namespace-scoped cluster commands
- **Full lifecycle coverage** — update and handoff skills close gaps for released and partial quickstarts
- **Security by default** — dedicated security stage before documentation and shipping
- **Test quality** — separated test authoring from code fixes ensures tests represent real requirements, not implementation artifacts

### Negative

- **Complexity increase** — from 10 flat SKILL.md files to 15 skills with ~43 subagent prompts, hooks, KB, and guardrails
- **Maintenance burden** — more files to keep consistent and up to date
- **Debugging difficulty** — multi-subagent orchestration is harder to debug than flat skills
- **Onboarding cost** — new contributors must understand the orchestration patterns
- **Temp file dependency** — `/tmp/` files are ephemeral; long-running sessions across reboots could lose state

### Mitigations

- Subagent logging hook provides visibility into orchestration
- Each skill's `subagents/README.md` documents all subagent roles clearly
- `make skills-check` continues to validate all skills against agentskills.io spec
- Persistent reports (`.rhoai/deploy-report.yaml`, `.rhoai/security-report.yaml`) survive beyond `/tmp/`
- Phase-gated rollout allows validating each change before the next

---

## References

- [agentskills.io](https://agentskills.io) — skill spec standard
- [ai-architecture-charts](https://github.com/rh-ai-quickstart/ai-architecture-charts) — Helm chart repo
- [ai-quickstart-template](https://github.com/rh-ai-quickstart/ai-quickstart-template) — monorepo template
- [docs/skills-development.md](../skills-development.md) — existing development guide
