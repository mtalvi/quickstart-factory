---
name: rh-qs-discovery
description: >-
  Ideation and interview phase for new AI Quickstarts. Orchestrates PRD
  structuring and backlog matching subagents to produce a validated PRD.
  Supports structured interviews, uploaded documents, and gap analysis.
argument-hint: "Start a new quickstart idea, upload documents, or analyze coverage gaps"
allowed-tools: Bash, Read, Write, Edit, Agent
---

# rh-qs-discovery

**Category:** `inception/`

## Goal

Produce a validated Product Requirements Document (PRD) at `data/prds/<slug>.md` that is ready for handoff to `rh-qs-architect`. The PRD captures the user's quickstart idea with enough detail for architectural design without pre-deciding implementation choices.

## Input

- User says **"hello"** (guided from scratch)
- **"I want to build X"** (idea-driven)
- User uploads or pastes a document with ideas (design doc, meeting notes, issue text)
- **"Identify gaps"** / **"Coverage analysis"** (gap analysis mode — see below)

## Supporting Documents

**Main agent reads directly:**

| File | When |
|------|------|
| [reasoning-guardrails.md](./reasoning-guardrails.md) | Phase 7 — PRD validation and refinement |
| [spec-template.md](./spec-template.md) | Phase 4 — generating the discovery spec |
| [output-templates.md](./output-templates.md) | Phase 8 — writing the final PRD |

**Subagents read (pass by file path only — do NOT read these):**

| File | Subagent |
|------|----------|
| [subagents/prd-structurer-prompt.md](./subagents/prd-structurer-prompt.md) | PRD structurer |
| [subagents/backlog-matcher-prompt.md](./subagents/backlog-matcher-prompt.md) | Backlog matcher |

## Workflow

### Phase 1: Initial Input

Accept the user's input and determine the input type:
- **`document`**: User uploaded or pasted structured content (design doc, RFC, meeting notes)
- **`conversation`**: User is describing an idea interactively
- **`raw_idea`**: User gave a brief description (1-3 sentences)

Extract initial keywords for the backlog check: technologies mentioned, industry/domain, AI capabilities, use case pattern.

### Phase 2: Backlog Check

Check if this idea already exists in the backlog:

```bash
python3 core/skills/gh-backlog-reader/scripts/read_backlog.py --search "<keywords>"
```

Spawn the **backlog-matcher subagent** to compare the idea against the backlog:

```python
Agent(
    description="Check backlog for duplicate quickstart ideas",
    prompt=f"""
Read and follow instructions from:
core/skills/rh-qs-discovery/subagents/backlog-matcher-prompt.md

Idea summary: {idea_summary}
Idea keywords: {keywords}
Backlog data: {backlog_output}
"""
)
```

Based on the match report:
- **`duplicate`**: Present the matching issue to the user. Ask: continue anyway, or work on the existing issue?
- **`similar`**: Present matches and ask: extend the existing issue, or create a distinct PRD?
- **`unique`**: Proceed to Phase 3.

Record the user's decision in the discovery spec.

### Phase 3: Issue Destination

**Ask the user** where to store feature requests and backlog items before creating issues:

| Destination | Repo / tool | Use when |
|-------------|-------------|----------|
| **Quickstart backlog** (default) | `rh-ai-quickstart/ai-quickstart-contrib` | Standard quickstart suggestions — use **`gh-issue-creator`** |
| **RFE (Request for Enhancement)** | [opendatahub-io/rfe-creator](https://github.com/opendatahub-io/rfe-creator) | Platform or OpenShift AI product feedback; partners may lack access — confirm first |
| **Strategy / roadmap** | [opendatahub-io/strat-creator](https://github.com/opendatahub-io/strat-creator) | Strategic themes, not a single quickstart implementation |

Record the chosen destination. The issue URL will be added to the PRD header when an issue is created.

### Phase 4: Generate Discovery Spec

Read [spec-template.md](./spec-template.md). Based on the input type, initial analysis, and backlog check results, generate `.rhoai-qs/pipeline/discovery-spec.yaml` with:
- Completed sections (what we already know from the user's input)
- Remaining interview questions (what we still need to ask)
- Gap questions relevant to this idea
- Backlog check results and user decision

Present the spec to the user for approval before proceeding to the interview.

### Phase 5: Structured Interview / PRD Structuring

**If the user uploaded documents:** Spawn the **prd-structurer subagent** to extract structured PRD sections:

```python
Agent(
    description="Structure uploaded document into PRD sections",
    prompt=f"""
Read and follow instructions from:
core/skills/rh-qs-discovery/subagents/prd-structurer-prompt.md

Raw input: {document_content}
Input type: document
Output template path: core/skills/rh-qs-discovery/output-templates.md
"""
)
```

Review the subagent's output. For sections with `medium` or `low` confidence, ask follow-up questions from the interview table below.

**If conversational:** Run the structured interview. Ask until every PRD section can be filled:

| Topic | Question |
|-------|----------|
| Problem | What problem does this quickstart solve? (one sentence) |
| User | Who is the target user? (developer, data scientist, business analyst) |
| Flow | What's the primary user flow? (upload → process → display?) |
| UI | Does it need a UI or is API-only acceptable? |
| AI capability | What AI capability is central? (text generation, RAG, agents, classification, vision). **This must be true — if there is no AI capability, the quickstart should be rejected.** |
| Data | What data does it work with? (documents, transactions, images, real-time streams) |
| Storage | Does it need persistent storage? What kind? |
| Models | Any specific model requirements? (size constraints, safety, multilingual) |
| Deploy | Deploy target: OpenShift AI only, or also local dev with podman? |
| Compliance | Any compliance/security considerations? |

**Gap questions (ask when relevant to the idea):**

- Will this need RAG?
- Real-time inference or batch?
- How many concurrent users?

Update the discovery spec as answers come in: move items from `remaining_questions` to `completed_sections`.

### Phase 6: Requirement Mapping

Map vague ideas to concrete requirements using known patterns:

| Vague idea | Concrete requirement |
|------------|---------------------|
| Chat with documents | RAG + pgvector + ingestion pipeline |
| Agent with tools | Llama Stack + optional MCP servers |
| PDF upload and processing | MinIO + extraction service |
| On-cluster LLM | llm-service chart + GPU |

Record these mappings in the PRD but do not pre-decide the architecture — the mapping provides context for `rh-qs-architect`, not binding decisions.

### Phase 7: PRD Validation and Refinement

Read [reasoning-guardrails.md](./reasoning-guardrails.md). Validate the PRD draft for completeness:

**Required PRD sections** (all must have substantive content):
- Problem statement with target persona
- At least one step-by-step user flow
- Data model (input types, storage needs)
- AI touchpoints with rationale
- Deploy target with GPU determination
- Scope boundaries with at least one non-goal

Check each `validation_rules` entry from the discovery spec. If any `blocker` rule fails, fix the PRD before presenting it.

Present the draft PRD to the user. **Uncapped refinement** — the user can refine as many times as they want. This is a collaborative, user-driven conversation with no iteration limit.

For each refinement round:
1. Present the current PRD draft
2. User provides feedback (changes, additions, corrections)
3. Update the PRD
4. Re-validate against guardrails
5. Present again — repeat until the user approves

### Phase 8: Write PRD

Read [output-templates.md](./output-templates.md). Write the final PRD to `data/prds/<slug>.md` using the template format. Include the PRD header with:
- Issue destination and URL (if created)
- Slug
- Date

Slug convention: lowercase, hyphenated (e.g., `mortgage-application-processor`).

Confirm with the user before handing off to `rh-qs-architect`.

## Gap Analysis Mode

When the user asks to identify gaps (absorbs rh-qs-quickstart-identifier):

1. Fetch the backlog:
   ```bash
   python3 core/skills/gh-backlog-reader/scripts/read_backlog.py --summary
   python3 core/skills/gh-backlog-reader/scripts/read_backlog.py --detail
   ```
2. Read [references/industry-trends.md](./references/industry-trends.md) (if available)
3. Compare coverage by industry, technology, and use case
4. Propose 3-5 new quickstart ideas with rationale
5. Save report to `data/reports/gap-analysis-<date>.md` using the template from [references/gap-analysis-template.md](./references/gap-analysis-template.md)

If the user wants to pursue one of the proposed ideas, transition into the standard discovery flow (Phase 1 onward) with the gap analysis context as initial input.

## Guidelines

**DO:**
- Preserve the user's language — use their words in the PRD, not your rephrasing
- Flag ambiguities as open questions rather than resolving them yourself
- Use the backlog-matcher subagent before starting the interview
- Update the discovery spec as the interview progresses
- Let the user refine the PRD as many times as they want

**DON'T:**
- Read subagent prompt files directly — pass them by file path to the Agent tool
- Invent requirements the user didn't state
- Pre-decide the technology stack — that's `rh-qs-architect`'s job
- Assume GPU is needed without evidence
- Skip the backlog check
- Resolve open questions without asking the user

## Next Skill

When PRD is approved → **`rh-qs-architect`**
