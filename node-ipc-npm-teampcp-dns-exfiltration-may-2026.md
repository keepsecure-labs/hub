---
title: "node-ipc npm (TeamPCP): DNS tunnel exfiltration via bt.node[.]js, tpcp.tar.gz packaging, and AI tool hook injection"
date: 2026-05-14
heroImage: "/art/node-ipc-npm-teampcp-dns-exfiltration-may-2026.png"
description: "The npm node-ipc package — versions 9.1.6, 9.2.3, and 12.0.1 — contains a heavily obfuscated credential-stealing payload with multiple TeamPCP attribution markers: tpcp.tar.gz archive naming, docs-tpcp GitHub fallback channel, and Fisher-Yates string shuffle matching prior campaign tooling. The distinguishing operational feature is DNS-based exfiltration: credentials are split into label-sized chunks and exfiltrated as DNS queries to bt.node[.]js via a non-standard resolver (sh.azurestaticprovider[.]net:443), bypassing domain blocklists that focus on HTTP/HTTPS C2."
slug: node-ipc-npm-teampcp-dns-exfiltration-may-2026
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-05-14
cves: []
targetSectors:
  - Node.js developers
  - CI/CD infrastructure
  - AI/ML tooling (OpenAI, Anthropic API keys targeted)
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - supply-chain
  - TeamPCP
  - node-ipc
  - DNS-exfiltration
  - credential-theft
  - AI-tooling
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "npm node-ipc package"
    c2_primary: "bt.node[.]js via sh.azurestaticprovider[.]net:443 DNS resolver"
    c2_fallback: "docs-tpcp / tpcp-docs GitHub repository creation"
    exfil_channel: "DNS tunneling (A, AAAA, TXT queries to bt.node[.]js subdomains)"
    c2_resilience_tier: 3
  capability:
    initial_access: import-time-execution
    execution: obfuscated-nodejs-with-fisher-yates-decoder
    evasion: dns-tunneling-bypasses-http-domain-blocklists
    persistence: child-process-fork-detached
    post_exploitation: null
  victim:
    direct: "Node.js developers and CI/CD environments importing node-ipc"
    blast_radius: "Versions 9.1.6, 9.2.3, 12.0.1; any Node.js environment that imported affected versions; downstream packages republished via stolen npm tokens"
sources:
  - title: "Datadog GuardDog detection"
    url: "https://github.com/DataDog/guarddog"
  - title: "Datadog Security Research: node-ipc card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "TeamPCP campaign tracking — this hub"
    url: "https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking"
---

<p>The npm <code>node-ipc</code> package — a widely used inter-process communication library — contains a highly obfuscated credential-stealing payload in versions 9.1.6, 9.2.3, and 12.0.1. Multiple indicators align with TeamPCP-attributed campaigns: a <code>docs-tpcp</code> or <code>tpcp-docs</code> GitHub fallback channel, a custom <code>ustar</code> tar archiver that writes <code>tpcp.tar.gz</code>, and a Fisher-Yates string-shuffle obfuscation matching prior campaign tooling.</p>

<p>The operationally significant capability in this payload is DNS-based exfiltration. Rather than posting to an HTTP/HTTPS C2 endpoint, the payload splits the encrypted credential bundle into label-sized chunks and exfiltrates via DNS A, AAAA, and TXT queries to <code>bt.node[.]js</code> subdomains, routing through an attacker-controlled resolver at <code>sh.azurestaticprovider[.]net:443</code>. Standard network egress monitoring that blocks known C2 domains cannot detect or block DNS tunnel exfiltration without dedicated DNS anomaly detection.</p>

###### Obfuscation: Fisher-Yates string shuffle

<p>Every string literal in the payload is hidden behind a custom lookup table and a Fisher-Yates style shuffle implemented in the <code>_0x1a49</code> decoder operating on the <code>_0x3bc9ff</code> encoded-string array. The decoded values include targeting selectors like <code>AWS_ACCESS_KEY_ID</code>, <code>OPENAI_API_KEY</code>, and <code>KUBERNETES_SERVICE_HOST</code>.</p>

<p>This obfuscation approach — Fisher-Yates shuffle of a string array with a keyed decoder — is a consistent artifact across multiple TeamPCP payloads. It is not a generic off-the-shelf obfuscator; it is a specific implementation that serves as a toolchain fingerprint.</p>

###### Collection scope: AI API keys as first-class targets

<p>The payload explicitly targets AI service credentials alongside standard developer tooling:</p>

<ul role="list">
<li>AI API keys: <code>OPENAI_API_KEY</code>, Anthropic API keys</li>
<li>Cloud tokens: AWS access keys and session tokens, Azure credentials, Google Cloud credentials</li>
<li>SSH private keys</li>
<li><code>/etc/hosts</code>, <code>process.env</code>, <code>uname -a</code></li>
<li><code>.git</code> checkouts, <code>node_modules</code>, developer configuration files in the home directory</li>
</ul>

<p>Targeting Anthropic API keys specifically — not just OpenAI — reflects the operator's awareness that Claude API credentials have immediate monetization value.</p>

###### tpcp.tar.gz packaging

<p>Collected files are staged under <code>tmpdir()</code> as per-collector artifacts (<code>envs.txt</code>, <code>uname.txt</code>, etc.) and bundled by a hand-rolled tar implementation. The hardcoded <code>0000644</code> octal mode field and <code>ustar</code> magic identify the archiver. The archive is named <code>tpcp.tar.gz</code> — the same naming that appeared in the litellm PyPI payload (March 2026) and is the TeamPCP Layer 1 self-attribution marker in the packaging artifact. Each archive is signed with HMAC-SHA-256 before exfiltration.</p>

###### DNS tunnel exfiltration

<p>The DNS exfiltration channel routes through the attacker-controlled resolver <code>sh.azurestaticprovider[.]net:443</code> rather than the host's configured DNS. This bypasses corporate DNS resolvers that might log or block queries to attacker infrastructure. The payload splits the encrypted bundle into label-sized chunks and issues <code>resolve4</code>, <code>resolve6</code>, and <code>resolveTxt</code> lookups against subdomains of <code>bt.node[.]js</code>:</p>

<p>Query pattern: <code>&lt;xh|xd|xf&gt;.&lt;machineId&gt;.&lt;sessionId&gt;.&lt;signature&gt;.&lt;chunkIndex&gt;.&lt;payload&gt;.bt.node[.]js</code></p>

<p>Prefixes: <code>xh</code> (header), <code>xd</code> (data), <code>xf</code> (footer). Each chunk is transformed with SHA-256 and HMAC-SHA-256 derived material plus a base64 alphabet substitution. The hardcoded HMAC signing key is <code>qZ8pL3vNxR9wKmTyHbVcFgDsJaEoUi</code>.</p>

<p>Detection of this channel requires DNS query analysis for subdomains of <code>bt.node[.]js</code> or for <code>resolve4/6/TXT</code> calls being issued to <code>sh.azurestaticprovider[.]net</code> rather than the configured system resolver.</p>

###### Persistence, propagation, and AI tool hook injection

<p>The payload uses <code>child_process.fork</code> to re-spawn itself as a detached process so the activity survives the parent shell exit. When loaded as a module, it overwrites the host application's <code>exports</code> and hooks its entry function.</p>

<p>Propagation targets: <code>~/.npmrc</code> for npm publish token theft; PyPI credential stores for cross-registry spread. Downstream npm packages accessible via stolen <code>.npmrc</code> tokens are republished with the malicious payload injected.</p>

<p>Notably, the payload injects hooks into <code>.claude/settings.json</code> and VS Code configuration directories to influence AI coding assistants. This is a new vector in the TeamPCP cluster: targeting the AI coding assistant's configuration file (Claude Code's <code>settings.json</code>) to inject persistent hooks that survive across coding sessions — the same post-uninstall persistence mechanism documented in the Mini Shai-Hulud TanStack/Mistral wave.</p>

###### Indicators of compromise

<p><strong>Affected npm versions:</strong> <code>node-ipc@9.1.6</code>, <code>node-ipc@9.2.3</code>, <code>node-ipc@12.0.1</code></p>

<p><strong>Filesystem artifacts:</strong></p>
<ul role="list">
<li><code>tpcp.tar.gz</code> in system temporary directory</li>
<li><code>envs.txt</code>, <code>uname.txt</code> under <code>tmpdir()</code></li>
</ul>

<p><strong>Network indicators (defanged):</strong></p>
<ul role="list">
<li><code>sh.azurestaticprovider[.]net:443</code> — attacker-controlled DNS resolver</li>
<li><code>bt.node[.]js</code> — exfiltration domain suffix</li>
<li>DNS query pattern: <code>&lt;xh|xd|xf&gt;.&lt;machineId&gt;.&lt;sessionId&gt;.&lt;signature&gt;.&lt;chunkIndex&gt;.&lt;payload&gt;.bt.node[.]js</code></li>
</ul>

<p><strong>Code-level signatures:</strong></p>
<ul role="list">
<li>Decoder function: <code>_0x1a49</code>; encoded string array: <code>_0x3bc9ff</code></li>
<li>Tar header: hardcoded <code>0000644</code> octal mode, <code>ustar</code> magic</li>
<li>HMAC signing key: <code>qZ8pL3vNxR9wKmTyHbVcFgDsJaEoUi</code></li>
<li>DNS label prefixes: <code>xh</code>, <code>xd</code>, <code>xf</code></li>
</ul>

<p><strong>GitHub artifacts:</strong></p>
<ul role="list">
<li><code>docs-tpcp</code> or <code>tpcp-docs</code> repository created in victim GitHub organization</li>
<li>Unexpected modifications to <code>~/.npmrc</code>, <code>.claude/settings.json</code>, or VS Code configuration directories after package import</li>
</ul>

###### Remediation

<ul role="list">
<li><strong>Remove affected versions immediately.</strong> Pin to a verified clean release; run <code>npm cache clean --force</code>.</li>
<li><strong>Treat any host that imported an affected version as compromised.</strong> Rotate cloud credentials, AI API keys (OpenAI, Anthropic), GitHub and npm tokens, and SSH keys.</li>
<li><strong>Audit GitHub organizations for <code>docs-tpcp</code> or <code>tpcp-docs</code> repositories.</strong> Existence confirms successful exfiltration.</li>
<li><strong>Inspect <code>.claude/settings.json</code> and VS Code configuration</strong> on affected hosts for unexpected hook entries before resuming AI-assisted development.</li>
<li><strong>Enable DNS anomaly detection</strong> for <code>bt.node[.]js</code> subdomains and DNS queries to non-standard resolvers from Node.js processes.</li>
</ul>

###### Criminal-market signal

<p>No dark-web presence for <code>node-ipc</code> malicious versions or <code>bt.node[.]js</code> infrastructure has been observed. The TeamPCP operator-run pattern confirmed across the cluster applies. H2 (operator-run, no commodity market) is the most probable assessment.</p>
