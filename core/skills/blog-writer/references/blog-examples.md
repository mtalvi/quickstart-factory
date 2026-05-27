# Blog Post Examples

These are real published posts and READMEs that represent the tone, structure, and business perspective we want in our blog drafts. Before drafting a blog post, fetch each example below using WebFetch and study how they approach the topic. Use them as stylistic guides — match their narrative depth and enterprise perspective, not just the template structure.

When analyzing each example, pay attention to:
- How the business problem is introduced before any technical detail
- How architecture decisions are tied to enterprise needs (cost, compliance, scalability)
- The balance between business narrative and technical walkthrough
- How the audience (enterprise architects, platform teams) is addressed
- Progressive disclosure: summary first, detail later

---

## Stock Recommendation AI Agents

End-to-end blog post about building agentic AI for stock analysis on OpenShift, using LangGraph, AWS Bedrock RAG, and TrustyAI. Demonstrates how a fintech team moves from POCs to production-grade AI agents across a hybrid multi-cloud environment.

- **URL:** https://raw.githubusercontent.com/luis5tb/luis5tb.github.io/main/_posts/2025-03-26-stock-ai-agents.md
- **Type:** Full blog post (published)
- **Learn from:**
  - Business-first framing: opens with enterprise AI transition narrative, not technology
  - Architecture presented through the lens of a fictitious enterprise team's goals and constraints
  - Detailed walkthrough that maintains business context throughout
  - Multi-cloud and hybrid positioning tied to existing enterprise investments
  - Responsible AI as a business requirement, not just a feature checkbox
  - Future roadmap section that signals strategic direction

## AI Virtual Agent

Platform README for building conversational AI virtual agents on OpenShift AI. Covers agent management, knowledge base integration via RAG, MCP tool ecosystem, and guardrails — with both local dev and cluster deployment paths.

- **URL:** https://raw.githubusercontent.com/rh-ai-quickstart/ai-virtual-agent/refs/heads/dev/README.md
- **Type:** Quickstart README (technical)
- **Learn from:**
  - Clean progressive disclosure: one-line summary → features → architecture → deploy → advanced
  - Audience segmentation in advanced section (developers, deployment, integration)
  - Concrete code examples that serve as both documentation and quick-start tutorial
  - Concise deploy path (3 commands) with links to deeper guides
  - Feature list that emphasizes capabilities from a user perspective, not implementation details

## Oracle SQLcl MCP Server on OpenShift

Red Hat Developer article showing how to deploy Oracle's SQLcl MCP server on OpenShift and connect it to the AI Virtual Agent. Enables natural language querying of Oracle databases through agentic AI, grounded in a sales analyst scenario using TPC-DS benchmark data.

- **URL:** https://developers.redhat.com/articles/2026/01/20/deploy-oracle-sqlcl-mcp-server-openshift
- **Type:** Developer article (published on Red Hat Developer)
- **Learn from:**
  - Grounds the demo in a concrete business scenario ("sales analyst team wants natural language access to sales data")
  - Enterprise security framing: read-only replicas, Oracle RBAC, DBA audit trails for AI-originated queries
  - Practical developer-to-developer tone that stays accessible without dumbing down
  - Acknowledges real deployment constraints (GPU availability, model selection tradeoffs)
  - Bridges existing enterprise investments (Oracle backends) with new AI capabilities — no rip-and-replace narrative
