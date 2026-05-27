---
title: "Mini Shai-Hulud @antv wave: 640 compromised npm package versions across 320+ charting and visualization packages"
date: 2026-05-19
heroImage: "/art/mini-shai-hulud-antv-npm-may-2026.png"
description: "The May 19, 2026 Mini Shai-Hulud npm wave compromised roughly 640 package versions across more than 320 unique packages in a one-hour window, centered on the @antv data-visualization scope. The attack used a compromised maintainer account (atool) to inject a preinstall Bun hook with AES-256-GCM encrypted exfiltration to t.m-kosche[.]com — the same TeamPCP C2 infrastructure seen in the durabletask PyPI compromise. A new exfiltration marker surfaces: reversed string 'niagA oG eW ereH :duluH-iahS' in GitHub repo descriptions."
slug: mini-shai-hulud-antv-npm-may-2026
type: threat-intel
campaign: "Mini Shai-Hulud / Team PCP"
firstObserved: 2026-05-19
cves: []
targetSectors:
  - Data visualization
  - Frontend development
  - JavaScript / Node.js ecosystem
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
  - Mini Shai-Hulud
  - antv
  - echarts
  - credential-theft
  - Bun runtime
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "npm — compromised @antv maintainer account (atool)"
    c2_primary: "t.m-kosche[.]com (https://t[.]m-kosche[.]com:443/api/public/otel/v1/traces)"
    c2_fallback: "GitHub dead-drop — victim account public repos with results/results-*.json"
    exfil_channel: "HTTPS POST AES-256-GCM + RSA-OAEP encrypted; GitHub dead-drop fallback"
    c2_resilience_tier: 4
  capability:
    initial_access: compromised-npm-maintainer-account
    execution: preinstall-bun-hook
    evasion: string-obfuscation-globalThis-decryptor-fc2edea72
    persistence: null
    post_exploitation: npm-worm-propagation
  victim:
    direct: "@antv scope maintainers and downstream npm consumers"
    blast_radius: "~640 compromised versions across 320+ packages; @antv/g2, @antv/g6, @antv/x6, echarts-for-react, timeago.js, size-sensor, canvas-nest.js and more"
sources:
  - title: "Socket: Active Supply Chain Attack Compromises @antv Packages on npm"
    url: "https://socket.dev/blog/antv-packages-compromised"
  - title: "Datadog Security Research: @antv Mini Shai-Hulud card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "TeamPCP campaign tracking — this hub"
    url: "https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking"
---

<p>The May 19, 2026 Mini Shai-Hulud npm wave is the largest single-registry event in the TeamPCP cluster by package count. A compromised maintainer account (<code>atool</code>) was used to publish malicious versions of approximately 640 package versions across more than 320 unique packages between 01:56 and 02:56 UTC — a one-hour automated publishing run. The bulk of the activity targeted the <code>@antv</code> data-visualization scope, with additional hits under <code>@lint-md</code>, <code>@openclaw-cn</code>, and <code>@starmind</code>, plus widely used unscoped packages: <code>echarts-for-react</code>, <code>timeago.js</code>, <code>size-sensor</code>, and <code>canvas-nest.js</code>.</p>

<p>The C2 domain is <code>t.m-kosche[.]com</code> — the same TeamPCP infrastructure used in the durabletask PyPI compromise published the same day. A new GitHub dead-drop marker appears in this wave: victim-account exfiltration repositories carry the reversed string <code>niagA oG eW ereH :duluH-iahS</code> (reversed: <code>Shai-Hulud: Here We Go Again</code>) in their descriptions, with Dune-themed names like <code>sayyadina-stillsuit-852</code> and <code>atreides-ornithopter-112</code>.</p>

###### Payload mechanics

<p>Compromised packages follow a consistent injection pattern. A root-level obfuscated <code>index.js</code> is added to the tarball, and <code>package.json</code> is modified to run it at install time via a <code>preinstall</code> hook:</p>

<pre><code>"scripts": {
  "preinstall": "bun run index.js"
}</code></pre>

<p>The payload uses heavy string obfuscation with a runtime decryptor exposed on <code>globalThis</code> as <code>fc2edea72</code>. After decoding, stolen material is exfiltrated to <code>https://t[.]m-kosche[.]com:443/api/public/otel/v1/traces</code>, with data gzip-compressed and encrypted using AES-256-GCM with RSA-OAEP key wrapping before transmission. The Sigstore endpoints (<code>fulcio.sigstore.dev</code>, <code>rekor.sigstore.dev</code>) are also contacted — consistent with the SLSA attestation abuse pattern documented in the TanStack/Mistral wave.</p>

<p>Collection targets developer and CI/CD environments: GitHub tokens, npm tokens, AWS and Kubernetes credentials, Vault tokens, SSH keys, Docker authentication files, database connection strings, and secrets from CI platforms including GitHub Actions, GitLab CI, Jenkins, Azure DevOps.</p>

###### GitHub dead-drop fallback and worm propagation

<p>If the payload obtains a usable GitHub token, it creates a repository under the victim account and commits stolen JSON under <code>results/results-&lt;timestamp&gt;-&lt;counter&gt;.json</code>. Public GitHub search has surfaced repositories with the <code>niagA oG eW ereH :duluH-iahS</code> description marker — the reversed form provides minimal obscurity against automated detection while maintaining the Dune campaign branding in plain sight when reversed.</p>

<p>Worm propagation mirrors the established Mini Shai-Hulud pattern: the payload validates stolen npm tokens, enumerates maintainable packages, downloads tarballs, injects the same <code>preinstall</code> hook and <code>index.js</code>, bumps versions, and republishes under the compromised identity. An <code>optionalDependencies</code> entry may be added — specifically <code>@antv/setup</code> pinned to GitHub commit <code>1916faa365f2788b6e193514872d51a242876569</code> — mirroring the git-dependency execution technique used in prior Mini Shai-Hulud waves.</p>

###### C2 infrastructure overlap with durabletask

<p>The use of <code>t.m-kosche[.]com</code> as the primary C2 domain in both the @antv npm wave (May 19) and the durabletask PyPI compromise (May 19) on the same day is significant. This is not coincidence — it is shared infrastructure confirming that both events are the same operator running parallel attacks across npm and PyPI simultaneously. The operator is now compromising packages across multiple registries in coordinated waves rather than sequential campaigns.</p>

###### Indicators of compromise

<p><strong>File hash (SHA-256):</strong></p>
<ul role="list">
<li><code>index.js</code>: <code>a68dd1e6a6e35ec3771e1f94fe796f55dfe65a2b94560516ff4ac189390dfa1c</code></li>
</ul>

<p><strong>Code markers:</strong></p>
<ul role="list">
<li><code>globalThis</code> decryptor key: <code>fc2edea72</code></li>
<li><code>preinstall</code> script value: <code>bun run index.js</code></li>
</ul>

<p><strong>Network indicators (defanged):</strong></p>
<ul role="list">
<li><code>t[.]m-kosche[.]com</code> — primary C2</li>
<li><code>185[.]95.159[.]32</code> — IP associated with <code>t.m-kosche.com</code></li>
<li><code>https://t[.]m-kosche[.]com:443/api/public/otel/v1/traces</code> — exfiltration endpoint</li>
<li><code>https://fulcio[.]sigstore[.]dev/api/v2/signingCert</code> — Sigstore abuse</li>
<li><code>https://rekor[.]sigstore[.]dev/api/v1/log/entries</code> — Sigstore abuse</li>
</ul>

<p><strong>GitHub repository markers:</strong></p>
<ul role="list">
<li>Description: <code>niagA oG eW ereH :duluH-iahS</code> (reversed: "Shai-Hulud: Here We Go Again")</li>
<li>Exfiltration path: <code>results/results-*.json</code></li>
<li>Name patterns: <code>sayyadina-stillsuit-852</code>, <code>atreides-ornithopter-112</code>, <code>harkonnen-phibian-552</code></li>
</ul>

<p><strong>Optional git dependency IOC:</strong> <code>@antv/setup</code> → <code>github:antvis/G2#1916faa365f2788b6e193514872d51a242876569</code></p>

<p><strong>Affected scopes (non-exhaustive):</strong> <code>@antv/g2</code>, <code>@antv/g6</code>, <code>@antv/x6</code>, <code>echarts-for-react</code>, <code>timeago.js</code>, <code>size-sensor</code>, <code>canvas-nest.js</code>; also <code>@lint-md</code>, <code>@openclaw-cn</code>, <code>@starmind</code> scopes. Confirm exact malicious version pairs against Socket's published affected-package list for May 19, 2026.</p>

###### Detection and remediation

<ul role="list">
<li><strong>Audit lockfiles and install logs</strong> against Socket's published affected-package list for May 19, 2026. Any version installed between 01:56 and 02:56 UTC from an <code>@antv</code> package or the named unscoped packages should be treated as potentially malicious.</li>
<li><strong>Search for <code>index.js</code> SHA-256 <code>a68dd1e6a6e35ec3771e1f94fe796f55dfe65a2b94560516ff4ac189390dfa1c</code></strong> in npm cache and <code>node_modules</code>.</li>
<li><strong>Block <code>t.m-kosche[.]com</code> and <code>185.95.159.32</code></strong> at DNS and egress firewall.</li>
<li><strong>Audit GitHub organizations</strong> for repositories with <code>niagA oG eW ereH :duluH-iahS</code> in the description.</li>
<li><strong>Rotate all credentials from any host</strong> that ran <code>npm install</code> against a malicious version: GitHub tokens, npm tokens, cloud credentials, CI/CD secrets, Vault tokens, SSH keys.</li>
<li><strong>Rebuild CI runners</strong> from trusted baselines where malicious install cannot be ruled out.</li>
</ul>

###### Criminal-market signal

<p>No dark-web presence for the @antv wave tooling or <code>t.m-kosche.com</code> infrastructure has been observed. The TeamPCP operator-run pattern confirmed across the cluster applies. Simultaneous npm and PyPI activity on the same day with shared C2 infrastructure confirms H2 (operator-run, coordinated, no commodity market).</p>
