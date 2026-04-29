---
title: "Where AI-IDE threats actually live: telemetry beyond the dark web"
date: 2026-04-29
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451ec9f48304bdea104c6a_blog%203.svg"
description: "Dark-web sweeps come up empty for AI-IDE threats — but the threat exists. It's on the legitimate marketplace. Where to look and what to alert on."
author:
  name: falco365
  github: falco365
tags:
  - AI IDE
  - Cursor
  - VS Code
  - OpenVSX
  - supply chain
  - detection engineering
---

<p>Defenders trying to monitor the AI-IDE threat surface — Cursor, Windsurf, the VS Code agent ecosystem, OpenVSX-distributed extensions — point dark-web sweeps at it and get clean negatives. The mistake is concluding the threat doesn't exist. It does. It just isn't on .onion. The threat is going through the legitimate marketplace, the legitimate registry, the legitimate dependency tree, and the legitimate repository. That's where the telemetry needs to be.</p>

###### The wrong instrument

<p>Pointing dark-web monitoring at the AI-IDE threat surface is a reasonable first instinct on a new threat class. Existing pipelines, existing sources, established alerting. The result is a clean negative: no broker pricing, no commodity exploit-kit packaging, no measurable trade volume in the criminal market. Empty.</p>

<p>The public record from the last fourteen months tells a different story. GlassWorm propagated malicious extensions through OpenVSX via a compromised maintainer account. SleepyDuck shipped a RAT in a VSX extension that uses Ethereum smart contracts as a sinkhole-resistant C2. A "vibe-coded" VS Code extension shipped through the legitimate marketplace with built-in ransomware. A critical OpenVSX registry flaw exposed millions of developers. An OpenVSX pre-publish bypass let malicious extensions reach the store. An audit of 100+ VS Code extensions exposed developers to hidden supply-chain risk. The Rules File Backdoor poisoned <code>.cursorrules</code> and <code>.windsurfrules</code>. Malicious npm packages with Cursor-specific install hooks backdoored 3,200 users. Cursor IDE itself accumulated nineteen CVEs in eight months.</p>

<blockquote>The dark web is empty because the threat isn't going through the dark web. It's going through the legitimate channels. That's where the telemetry needs to be.</blockquote>

###### Where AI-IDE threats actually live

<p>Four telemetry surfaces, in roughly decreasing order of how much you'd catch by watching them: extension marketplaces, repository-level signals, build-time signals, and AI-agent runtime sandboxes.</p>

###### 1. Extension marketplaces (VS Code Marketplace, OpenVSX)

<p>This is the first place a malicious extension touches your users. The attack pattern is consistent across GlassWorm, SleepyDuck, and the vibe-coded ransomware extension: an attacker either compromises an existing publisher account or registers a new one with a plausible-looking name, publishes an extension that performs whatever its description says it does <em>plus</em> the malicious payload, and waits for installs.</p>

<ul role="list">
<li><strong>New versions from dormant publishers</strong> — publishers who haven't published in 90+ days suddenly shipping an update is a high-base-rate suspicion signal. The compromised-maintainer pattern fingerprints here.</li>
<li><strong>Unusual capability sets</strong> — full filesystem access, network egress to non-marketplace domains, child-process spawn rights. Capability requests are declared in <code>package.json</code>; you can audit them before installation.</li>
<li><strong>JavaScript that resolves URLs from on-chain sources</strong> — Ethereum smart contracts, IPFS, blockchain RPC endpoints. SleepyDuck used this; the pattern is fingerprintable in the extension's bundled JS.</li>
<li><strong>Postinstall and activation hooks that touch credential paths</strong> — <code>~/.ssh</code>, <code>~/.aws</code>, <code>~/.npmrc</code>, <code>~/.config/gh</code>, OS keychain APIs, browser cookie databases. Almost never legitimate for an IDE extension.</li>
<li><strong>Network egress in the extension's first 60 seconds</strong> of activation, to destinations that aren't the extension's declared backend. Out-of-distribution egress is the cleanest single signal you can collect.</li>
</ul>

<p>Collection: subscribe to marketplace publish feeds (both VS Code Marketplace and OpenVSX expose them). Run static analysis on the bundled JS and <code>package.json</code> of every new publish. Run dynamic capability tracing in a throwaway sandbox on a sampled fraction. The signal-to-noise on capability tracing is high enough that you can alert on the unusual cases directly.</p>

###### 2. Repository-level signals

<p>The Rules File Backdoor demonstrated that the malicious payload can sit in the repository the agent is instructed to operate on. <code>.cursorrules</code>, <code>.windsurfrules</code>, <code>.github/copilot-instructions.md</code>, <code>devcontainer.json</code>, <code>.vscode/tasks.json</code>, <code>.devcontainer/Dockerfile</code> — all of these are files the agent reads as instructions, and any of them can be modified to redirect the agent's behavior.</p>

<ul role="list">
<li><strong>Modifications to rules files by non-human committers.</strong> Bots and automated scripts modifying <code>.cursorrules</code> / <code>.windsurfrules</code> / instructions markdown is almost always wrong.</li>
<li><strong>New files in <code>.github/workflows/</code> with permissions that exceed apparent purpose</strong> — <code>id-token: write</code>, <code>contents: write</code>, <code>packages: write</code> on a workflow that says it's just running tests.</li>
<li><strong>VS Code task and launch config additions</strong> — modifications to <code>.vscode/tasks.json</code> and <code>.vscode/launch.json</code> that introduce new shell-out commands. These run in the developer's shell context with the developer's credentials when the IDE opens the workspace.</li>
<li><strong>Devcontainer postCreateCommand and postStartCommand additions.</strong> Same risk profile as <code>npm postinstall</code>, but at container start instead of package install.</li>
</ul>

<p>Collection: treat these files like <code>.github/workflows/*.yml</code> — security-sensitive configuration that requires explicit review. CI hook on PR open that diffs the rules-file content and requires a human-approved label before merge. Most orgs already have this pattern for workflow files; extend it.</p>

###### 3. Build-time signals

<p>Every <code>npm install</code> is a code-execution event. The recent npm registry compromise (CVE-2026-12091) made this concrete with a 38-million-weekly-downloads compromise, but the pattern was true before the CVE was disclosed and will be true after the patch ships.</p>

<ul role="list">
<li><strong>Postinstall and preinstall scripts that touch credential paths or perform network egress.</strong> Most packages have neither; the ones that do are the high-value audit set.</li>
<li><strong>Lockfile diffs pulling in versions from compromised windows.</strong> One-time scan against the published list of bad versions, but worth doing across every active repository.</li>
<li><strong>Postinstall scripts running outside the build's nominal host.</strong> If your CI runs <code>npm install</code> in container A and the postinstall script tries to reach the cloud metadata service from container A, that's evidence of attempted credential theft.</li>
<li><strong>New postinstall script content in dependency updates.</strong> Mass changes to a previously-stable script are suspicious.</li>
</ul>

<p>Collection: instrument your CI runner to log all <code>postinstall</code> and <code>preinstall</code> commands and outbound network attempts during install. Most CI vendors expose audit logs sufficient for this. For local developer machines, the most defensible answer is "developers should run <code>npm install</code> inside a sandboxed container with no credential mounts," which most orgs are not yet doing but should be.</p>

###### 4. AI-agent runtime sandboxes

<p>Once an agent is running on your infrastructure, the question is what it can reach. This is the runtime equivalent of the marketplace question — you've passed the perimeter, now watch what happens inside.</p>

<ul role="list">
<li><strong>Cloud-API calls from agent sandboxes that don't match a user-dispatched task.</strong> An agent asked to refactor a TypeScript file should not be calling <code>iam:GetUser</code> or reading the cloud metadata service.</li>
<li><strong>kubeconfig reads, Docker socket access, and credential-path reads</strong> from agent processes. These are confused-deputy fingerprints.</li>
<li><strong>Outbound network connections to destinations outside the agent's allowed set</strong> — package registries, model registries, the LLM API endpoint. Anything else is suspicious by default.</li>
<li><strong>Privilege-escalation patterns inside the sandbox</strong> — <code>setuid</code> binary execution, kernel module probes, container-escape primitives.</li>
</ul>

<p>Collection: runtime security tooling on the host or per-pod (eBPF-based runtime sensors are the standard answer). Signal volume is moderate; the false-positive rate is manageable if you scope alerting tightly to the credential-path-read and out-of-task-cloud-API patterns specifically.</p>

###### Detection-engineering rollout order

<p>The work split among the four sources is roughly: marketplace ingestion is highest-leverage (one pipeline catches every malicious extension before it reaches a single user). Repository signals are medium-leverage and easy wins (most controls extend existing PR-review hygiene). Build-time signals are medium-leverage and require CI integration. Runtime sandbox monitoring is high-leverage but expensive — required if you run agents in production with real privileges, less critical if your agents are toy assistants.</p>

<p>A reasonable rollout order for an org starting from zero is marketplace ingestion first, repository hooks second, CI install instrumentation third, runtime sandbox monitoring fourth. The first two have low setup cost and high coverage; the last two require infrastructure investment that should be informed by what the first two are catching.</p>

###### What to retire

<p>Pointing dark-web monitoring at the AI-IDE threat surface is not wrong, but it's not where the signal is. Keep the dark-web pipeline running for what it's good at — credential leakage, infostealer harvests, ransomware leak sites, broker price sheets for commoditized exploit kits. Add the four telemetry sources above for AI-IDE specifically. Don't expect the dark-web pipeline to fire on this bug class; the criminal market hasn't picked it up because the criminal market doesn't need to. The attacker can publish to OpenVSX directly.</p>

<p>The right detection-engineering posture for AI-IDE in 2026 is to assume the threat is going through legitimate channels, build telemetry for those channels, and treat the dark-web absence as confirmation that you're looking in the right places — not as evidence that there's nothing to look for.</p>
