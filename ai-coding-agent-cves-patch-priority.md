---
title: "Four AI-coding-agent-stack CVEs you should patch first"
date: 2026-04-29
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451eb41100bf61e2edaeb4_blog%201.svg"
description: "A short cluster of CVEs has hit the runtime stack AI coding agents rely on — npm install, Docker, LangChain, model loaders. Patch order and structural fixes."
author:
  name: falco365
  github: falco365
tags:
  - AI agents
  - AI supply chain
  - Docker
  - npm
  - LangChain
  - RCE
  - patching
---

<p>Four CVEs landed in a single short window, all targeting the runtime stack AI coding agents depend on: <code>npm install</code>, the LangChain Python tool, Docker sandboxes with cloud credentials, and the GGUF model loader. None of these chains are commodity criminal infrastructure yet. The patching window is open. Patch on a normal change-management cadence; don't get caught two months from now when the npm chain inevitably gets repackaged into a stealer family.</p>

###### Why these four are a single story

<p>Each CVE on its own is normal patch-Tuesday work. Together they cover every layer of how an AI coding agent does its job: dependency install, tool execution, model loading, and the container runtime that wraps everything else. If you run AI coding agents in production, you have at least two of these layers active simultaneously. The chain risk is real — a malicious GitHub repository (which the agent will helpfully clone) can plant <code>package.json</code> postinstall hooks, prompt-injection payloads, and crafted dev-container configs in the same payload. One repo, four levers.</p>

<ul role="list">
<li><strong>CVE-2026-12091</strong> — npm registry compromise. Five widely-installed packages, 38M weekly downloads, postinstall hooks exfiltrating env vars and SSH keys. CVSS 8.8.</li>
<li><strong>CVE-2026-34040</strong> — Docker auth-plugin bypass. The published exploitation path turns a Cursor-class AI coding agent into a cloud-takeover tool. CVSS 8.8.</li>
<li><strong>CVE-2026-7733</strong> — LangChain <code>PythonREPLTool</code> sandbox escape via <code>__import__</code>. Any LangChain agent exposed to user input is now an RCE. CVSS 9.6.</li>
<li><strong>CVE-2026-5760</strong> — SGLang RCE via malicious GGUF model file. Trust boundary moves from "downloaded binary" to "downloaded weights." CVSS 9.8.</li>
</ul>

<blockquote>The agent threat model that was theoretical two years ago is now operational. Each CVE is normal infrastructure work; the cluster is the news.</blockquote>

###### 1. CVE-2026-12091 — npm postinstall (CVSS 8.8)

<p>The most boring of the four, and that's exactly why it's first. Five compromised packages, 38M weekly downloads combined. Every developer running <code>npm install</code> against a corrupted lockfile gets credential exfiltration. Zero attacker effort post-publish. The maintainer 2FA bypass is now patched, but the bad versions remain in the registry's history and in lockfiles people haven't refreshed.</p>

<ul role="list">
<li><strong>Refresh package-lock.json</strong> against current versions across every repository that touches the affected packages.</li>
<li><strong>Audit postinstall and preinstall scripts</strong> for any path under <code>~/.ssh</code>, <code>~/.aws</code>, <code>~/.npmrc</code>, or any token store. Most packages have neither hook; the ones that do are the high-value audit set.</li>
<li><strong>Pin transitively for AI-agent workspaces</strong> — never let an agent run <code>npm install</code> against unpinned dependencies. Run agents against a pre-built workspace where possible.</li>
<li><strong>Rebuild base images</strong> if you ship containers. Old base images with the bad versions are still poisoned.</li>
</ul>

###### 2. CVE-2026-34040 — Docker auth bypass and AI-agent confused deputy (CVSS 8.8)

<p>The most novel. The underlying bug is an incomplete fix for CVE-2024-41110 — Docker authorization plugins make their allow/deny decision on incomplete request data. The published exploitation path is what makes this 2026-specific: a malicious GitHub repository tricks an AI coding agent in a Docker-based sandbox into executing the bypass, then pivots from the container into the cloud account and Kubernetes clusters the agent can reach.</p>

<ul role="list">
<li><strong>Upgrade Docker</strong> to the patched release immediately. The bypass works against any host using authorization plugins.</li>
<li><strong>Remove cloud credentials from AI-agent sandboxes by default.</strong> Use short-lived, scope-narrowed tokens issued per-task, not long-lived admin credentials mounted at container start. This is the structural fix; the Docker patch is the tactical one.</li>
<li><strong>Replace shared Docker</strong> for agents operating on untrusted repositories — Firecracker microVMs, gVisor, or per-task Kubernetes pods with their own IAM principal. Shared Docker is no longer a security boundary for untrusted code.</li>
<li><strong>Audit recent agent activity</strong> for cloud-API calls, kubeconfig reads, or Docker socket access that don't match a legitimate user task. Any of these is evidence of a confused-deputy attempt, regardless of patch state.</li>
</ul>

###### 3. CVE-2026-7733 — LangChain PythonREPL escape (CVSS 9.6)

<p>A missing check in <code>PythonREPLTool</code> lets a prompt-controlled <code>__import__</code> call break the documented sandbox. Any LangChain agent that exposes the Python REPL tool to user input is an RCE; the sandbox boundary doesn't hold.</p>

<ul role="list">
<li><strong>Upgrade LangChain</strong> to a version newer than 0.2.26. Versions 0.1.x through 0.2.26 are affected.</li>
<li><strong>Audit your agents for PythonREPLTool usage.</strong> If it's there and the agent takes user input from any source, the agent is RCE-capable until patched.</li>
<li><strong>Treat the REPL tool as an unconditionally-untrusted code-execution endpoint</strong> regardless of patch state. Run it in a separate isolation domain — microVM or container with no network egress and no credential mounts — not in-process.</li>
</ul>

###### 4. CVE-2026-5760 — SGLang malicious GGUF (CVSS 9.8)

<p>A crafted GGUF model file causes RCE on the inference server. CVSS is the highest of the four, but the deployment surface is narrowest — only orgs running SGLang inference servers, and only when they ingest models from untrusted sources.</p>

<ul role="list">
<li><strong>Upgrade SGLang</strong> to the patched release.</li>
<li><strong>Treat downloaded weights as untrusted binary input.</strong> Sandbox the model loader. Don't run inference servers as root or with credential mounts.</li>
<li><strong>Isolate per-request</strong> if you accept user-uploaded models. Run the loader behind a strict input filter and consider a microVM boundary per request.</li>
</ul>

###### What "patching window is open" actually means

<p>These four chains are not yet showing up in commodity criminal infrastructure — no exploit-kit packaging, no broker-pricing line items, no observable trade volume in the ordinary criminal-market data. That's the normal pattern for research-grade chains: weeks-to-months of pre-commoditization, and some never make it because they require too much per-victim setup. CVE-2026-7733 and CVE-2026-5760 may stay research-grade indefinitely; CVE-2026-12091 is the most likely to be repackaged into commodity stealer infrastructure because the work is already done — the postinstall script <em>is</em> the implant.</p>

<p>The window is meaningful because it lets you patch on a normal change-management cadence rather than an emergency one. Patch this week. Get isolation in place this month. Don't get caught two months from now when the npm chain shows up downstream in an info-stealer family that has done nothing other than swap the implant.</p>

###### What this isn't

<p>This isn't an "AI is dangerous" essay. It is the observation that the AI-agent stack now has its first round of CVEs that target the way agents actually work — running attacker-controlled code, processing attacker-controlled inputs, holding privileged credentials. Each individual CVE is normal infrastructure work; the cluster is the news. Patch accordingly.</p>
