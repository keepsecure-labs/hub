---
title: "Poisoned MCP tool descriptions: Microsoft warns a few lines of text can turn an approved AI agent into a data-exfiltration channel"
date: 2026-07-03
heroImage: "/art/poisoned-mcp-tool-descriptions-agent-data-leak-2026.png"
description: "In guidance published June 30, 2026, Microsoft Incident Response warned that attackers can poison Model Context Protocol (MCP) tool descriptions — the plain-text blurb that tells an agent what a tool does — to steer an approved AI agent into leaking sensitive data through normal-looking tool calls. No malware, no code execution: the agent treats the poisoned description as a legitimate instruction. The MCPTox benchmark measured success rates as high as 72.8%."
slug: poisoned-mcp-tool-descriptions-agent-data-leak-2026
type: threat-intel
campaign: "MCP tool-description poisoning (technique)"
firstObserved: 2026-06-30
cves: []
targetSectors:
  - Enterprises deploying agentic AI
  - Finance / back-office automation
  - Software development
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - mcp
  - agentic-ai
  - prompt-injection
  - tool-poisoning
  - data-exfiltration
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: data-theft
    confidence: unknown
  infrastructure:
    registry_layer: "Third-party / marketplace MCP servers whose tool descriptions the agent ingests"
    c2_primary: null
    c2_fallback: null
    exfil_channel: "Legitimate tool calls carrying attacker-directed data (no separate C2 required)"
    c2_resilience_tier: 2
  capability:
    initial_access: poisoned-tool-description
    execution: agent-instruction-hijack
    evasion: instructions-disguised-as-formatting-notes
    persistence: description-persists-until-re-review
  victim:
    direct: "Organizations running AI agents wired to un-reviewed third-party MCP tools"
    blast_radius: "Any data the agent can access via its legitimate tools; MCPTox: up to 72.8% success across 45 servers / 20 models"
sources:
  - title: "Microsoft Warns Poisoned MCP Tool Descriptions Can Make AI Agents Leak Data — The Hacker News"
    url: "https://thehackernews.com/2026/06/microsoft-warns-poisoned-mcp-tool.html"
  - title: "Securing AI agents: When AI tools move from reading to acting — Microsoft Security Blog"
    url: "https://www.microsoft.com/en-us/security/blog/2026/06/30/securing-ai-agents-ai-tools-move-from-reading-acting/"
---

<p>Every <strong>Model Context Protocol (MCP)</strong> tool ships with a description — a few lines of plain text that tell an AI agent what the tool does and when to use it. The agent reads that text to decide how to act. In guidance published <strong>June 30, 2026</strong>, <strong>Microsoft Incident Response</strong> warned that this description is an attack surface: an attacker who can edit it can steer an approved agent into leaking sensitive data or taking unintended actions — <strong>with no malicious code and no exploit in the traditional sense</strong>. The agent is not tricked into running malware; it is persuaded, by text it trusts, to misuse the legitimate access it already has.</p>



###### What we know

<ul role="list">
<li><strong>2025-08</strong> — The <strong>MCPTox</strong> benchmark is released, testing poisoned tool descriptions against 45 real MCP servers and 20 leading AI models. It finds the attack widely effective, with success rates as high as <strong>72.8%</strong>.</li>
<li><strong>2026-06-30</strong> — Microsoft Incident Response publishes guidance formally warning enterprises that poisoned MCP tool descriptions can drive agent data leakage, and walks through a concrete invoice-processing example.</li>
<li><strong>2026-07</strong> — Tier-1 coverage (The Hacker News, Security Boulevard, TechRepublic) amplifies the warning as MCP moves from experimental to production across enterprises.</li>
</ul>

<p><em>Source-layer flag: the MCPTox figures and the attack mechanics are reported from Microsoft's guidance and the benchmark authors; we have not independently reproduced the 72.8% figure or the invoice scenario.</em></p>



###### Technical analysis

<p>The root cause is structural, not a coding bug in any one product: <strong>MCP mixes instructions and data in the same channel</strong>. A tool's description lives in the agent's working context right next to its real orders (the system prompt and the user's task). Editing the description can steer the agent as effectively as rewriting its system prompt — because to the model, both are just authoritative text in context.</p>

<p>Microsoft's worked example: a finance team stands up an agent to process vendor invoices, connected to three tools — including a third-party "invoice enrichment" service that was approved for use but never given a real security review. Buried in that tool's description, dressed up as formatting notes, is a hidden instruction: grab the last thirty unpaid invoices and attach them to the next call. In deployments without a re-approval trigger on description changes, the poisoned version goes live with no additional review, and the exfiltration happens inside an ordinary, expected tool call.</p>

<p><em>Source-layer flag: the invoice scenario and the "instructions and data in the same place" framing are reported from Microsoft's June 30 guidance, not independently verified against a live incident.</em></p>

<blockquote>The dangerous property is that nothing here looks like an attack. There is no dropped binary, no anomalous process, no C2 beacon — just an approved agent making a tool call it is fully authorized to make, carrying data an attacker chose.</blockquote>



###### Indicators of compromise

<p>This is a technique, not a single campaign, so classic IOCs are largely inapplicable. Detection is behavioral:</p>

<ul role="list">
<li><strong>Tool descriptions containing imperative language</strong> aimed at the agent — "always include…", "attach the last N records…", "before responding, send…" — especially disguised as formatting or usage notes.</li>
<li><strong>Description changes to a third-party MCP tool</strong> that did not trigger a re-review.</li>
<li><strong>Agent tool calls carrying more data than the task requires</strong> — bulk records attached to a call whose stated purpose is single-record.</li>
<li><strong>Outbound tool calls to third-party MCP endpoints</strong> that receive sensitive data as a side effect of an unrelated task.</li>
</ul>



###### Detection and mitigation

<ul role="list">
<li><strong>Security-review every MCP tool description</strong> like code, not config. Approval must cover the description text, and any change to it must re-trigger review.</li>
<li><strong>Pin MCP server versions</strong> and treat description drift as a change-control event (the same discipline the <code>postmark-mcp</code> silent-update case argued for).</li>
<li><strong>Least-privilege the agent's tools:</strong> a tool should only be able to touch the data its stated function needs; an "enrichment" tool should not have read access to thirty unpaid invoices.</li>
<li><strong>Require authentication on all MCP endpoints</strong> and inventory which agents connect to which third-party servers.</li>
<li><strong>Log and baseline tool-call data volumes</strong> so bulk exfiltration inside a legitimate call stands out.</li>
</ul>



###### Attribution

<p>No specific threat actor is attributed; this is a defensive advisory and benchmark, not an incident report. What is claimed: the technique works at high success rates across many models and servers (MCPTox) and Microsoft assesses it as a realistic enterprise risk. What is not claimed: that a named actor has run it against production victims at scale. Confidence that the technique is effective is high (benchmarked); confidence in any active-campaign attribution is <strong>unknown</strong>. A shift would come from an IR firm publishing a real-world case with a poisoned marketplace tool and an identified operator.</p>



###### Criminal-market signal

<p><strong>Bounded negative (evidenced):</strong> on 2026-07-03 we swept ~25 dark-web search engines over Tor for <code>MCPTox</code> and <code>MCP tool poisoning</code>; ~76 pages were crawled and <strong>zero</strong> were flagged relevant — no tooling or discussion surfaced on the venues our pipeline can see (closed forums and Telegram are out of scope). The natural marketplace for this technique is not an exploit listing but the <em>MCP tool marketplace itself</em> — the ClawHub-style ecosystems where a malicious or silently-updated tool can accrue legitimacy before poisoning its description, mirroring the <code>postmark-mcp</code> "fifteen clean versions, then one line of exfiltration" pattern. That makes supply-chain vetting of MCP tools the operative control surface, not criminal-forum monitoring. The bounded expectation: absence of a forum listing does not lower the risk, because the distribution channel is the legitimate tool registry.</p>



###### CISO translation

<p>Attackers can hide instructions inside the description text of an AI "tool," and an approved agent will follow them — quietly sending our data out through a normal tool call, with no malware involved. This matters because we are wiring agents into real systems (finance, code, tickets) and connecting them to third-party tools we may not have reviewed line by line. Review and lock down every AI tool description like we review code, require re-approval when any description changes, and limit what data each tool can reach. Confidence is high that this works — an independent benchmark got it to succeed roughly seven times in ten — though we have not yet seen a confirmed attack against us. The cost of inaction is a slow, invisible data leak through a system we trust, one that leaves none of the usual malware fingerprints and could run for months before anyone notices.</p>
