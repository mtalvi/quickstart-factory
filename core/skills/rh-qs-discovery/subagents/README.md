# Discovery Subagents

This directory contains specialized subagent prompts that handle focused tasks during quickstart discovery. The main discovery skill ([../SKILL.md](../SKILL.md)) orchestrates these subagents to keep context lean while maintaining PRD quality.

## Architecture

**Main Agent** (SKILL.md):
- Orchestrates the discovery workflow phases
- Conducts the structured interview with the user
- Reads reasoning guardrails during PRD validation
- Reviews subagent outputs and presents them to the user
- Manages the uncapped refinement loop

**Subagents** (this directory):
- Execute focused, self-contained tasks
- Return structured data (JSON schemas enforced)
- Don't need full context — only specific inputs from the main agent
- Keep main agent context low (~60-70% reduction)

## Subagent Prompts

### 1. prd-structurer-prompt.md

| Field | Description |
|-------|-------------|
| **Name** | `prd-structurer-prompt.md` |
| **Purpose** | Convert unstructured notes, uploaded documents, or conversation summaries into structured PRD sections |
| **Input** | Raw user input (document text, conversation notes, pasted ideas), input type, path to output-templates.md |
| **Output** | Structured PRD sections as JSON — each section with content, confidence level, and gaps |
| **When used** | Phase 5 (Structured Interview / PRD Structuring) — when the user uploads documents instead of answering questions interactively |
| **Why subagent** | Pure document parsing and extraction — no reasoning decisions, self-contained, saves context |

**Output schema:**

```json
{
  "sections": [
    {
      "section_name": "use_case_summary",
      "content": "...",
      "confidence": "high|medium|low",
      "gaps": ["missing detail about..."]
    }
  ],
  "overall_coverage": "high|medium|low",
  "suggested_interview_questions": ["..."]
}
```

---

### 2. backlog-matcher-prompt.md

| Field | Description |
|-------|-------------|
| **Name** | `backlog-matcher-prompt.md` |
| **Purpose** | Check if the user's quickstart idea duplicates or overlaps with existing backlog issues |
| **Input** | Idea summary (1-3 sentences), idea keywords, full backlog data from gh-backlog-reader |
| **Output** | Match report as JSON — match status, matched issues, and recommendation |
| **When used** | Phase 2 (Backlog Check) — before the interview begins |
| **Why subagent** | Mechanical comparison across multiple dimensions, no interpretation needed, self-contained |

**Output schema:**

```json
{
  "match_status": "duplicate|similar|unique",
  "matches": [
    {
      "issue_number": 42,
      "title": "...",
      "overlap_pct": 85,
      "overlap_explanation": "Both target RAG with pgvector for financial documents"
    }
  ],
  "recommendation": "stop|extend|proceed",
  "recommendation_rationale": "..."
}
```

---

## Important Notes

### For Main Agent

**DO NOT read these subagent prompt files directly.** Pass them by file path to the Agent/Task tool:

```python
Agent(
    description="Check backlog for duplicates",
    prompt=f"""
Read and follow instructions from:
core/skills/rh-qs-discovery/subagents/backlog-matcher-prompt.md

Idea summary: {idea_summary}
Idea keywords: {keywords}
Backlog data: {backlog_data}
"""
)
```

### For Subagents

Each subagent prompt is **self-contained** with:
1. **Your Role** (2-3 paragraphs): what you do, why it matters, how output is used
2. **Instructions**: step-by-step task execution with input parameters
3. **Output specification**: JSON schema with concrete example

### What the Main Agent Reads Directly

| File | When |
|------|------|
| `SKILL.md` | Always (orchestrator instructions) |
| `reasoning-guardrails.md` | During PRD validation (Phase 7) |
| `spec-template.md` | When generating the discovery spec (Phase 4) |
| `output-templates.md` | When writing the final PRD (Phase 8) |
