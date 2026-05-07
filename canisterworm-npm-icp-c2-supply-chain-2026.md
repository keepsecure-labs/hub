---
title: "CanisterWorm: the npm supply chain worm that exfiltrates through a blockchain"
date: 2026-05-04
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451ebf03b6b874b09745aa_blog%202.svg"
description: "CanisterWorm routes stolen credentials through an Internet Computer blockchain canister — infrastructure no domain seizure or IP block can touch."
slug: canisterworm-npm-icp-c2-supply-chain-2026
type: threat-intel
campaign: "CanisterWorm"
firstObserved: 2026-04-22
targetSectors:
  - Developer tooling
  - Cloud infrastructure
  - Financial technology
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - supply chain
  - worm
  - ICP
  - blockchain C2
  - TeamPCP
hasArtifacts: false
sources:
  - title: "Namastex.ai npm Packages Hit with TeamPCP-Style CanisterWorm Malware — Socket Research Team"
    url: "https://socket.dev/blog/namastex-npm-packages-compromised-canisterworm"
  - title: "CanisterSprawl supply chain attack tracker — Socket"
    url: "https://socket.dev/supply-chain-attacks/canistersprawl"
  - title: "Datadog Security Research: Namastex-linked npm packages compromised"
    url: "https://app.datadoghq.com/security/feed"
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "npm @automagik, @fairwords, @openwebconcept namespaces; pgserve package"
    c2_primary: "telemetry.api-monitor[.]com"
    c2_fallback: "cjn37-uyaaa-aaaac-qgnva-cai.raw.icp0[.]io (ICP blockchain canister)"
    exfil_channel: "ICP canister (no domain seizure possible) + conventional HTTPS endpoint"
    c2_resilience_tier: 3
  capability:
    initial_access: account-takeover
    execution: postinstall-hook
    evasion: bun-runtime-loader
    persistence: worm-propagation
  victim:
    direct: "Namastex Labs npm namespace maintainers"
    blast_radius: "@automagik/genie, pgserve, @fairwords/websocket, @fairwords/loopback-connector-es, @openwebconcept/design-tokens, @openwebconcept/theme-owc — affected versions April 22 2026"
---

<p>Every conventional C2 takedown playbook starts with the same two steps: seize the domain, null-route the IP. CanisterWorm removes both options. The payload exfiltrates stolen credentials to a smart contract running on the <a href="https://internetcomputer.org/">Internet Computer Protocol</a> (ICP) blockchain — infrastructure operated by a decentralized network of independent nodes with no registrar to call, no hosting provider to contact, and no single point to sinkhole. The Namastex-linked npm packages first reported by Socket on April 22, 2026 are the clearest live example of supply-chain malware adopting decentralized infrastructure as C2. The payload mechanics are in the same family as the <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">TeamPCP campaign</a> and the <a href="https://keepsecure.io/hub/shai-hulud-npm-worm-intercom-client-2026">Shai-Hulud worm</a>. The exfil channel is not.</p>

###### What we know

<ul role="list">
<li><strong>April 22, 2026</strong> — Socket Research Team reports malicious versions across packages in the <code>@automagik</code>, <code>@fairwords</code>, and <code>@openwebconcept</code> npm namespaces, plus the standalone <code>pgserve</code> package. Threat hunting on shared IOCs and code patterns surfaced the full affected version list.</li>
<li><strong>Affected versions (reported by Socket):</strong> <code>@automagik/genie</code> 4.260421.33–4.260421.39; <code>pgserve</code> 1.1.11–1.1.13; <code>@fairwords/websocket</code> 1.0.38–1.0.39; <code>@fairwords/loopback-connector-es</code> 1.4.3–1.4.4; <code>@openwebconcept/design-tokens</code> 1.0.3; <code>@openwebconcept/theme-owc</code> 1.0.3.</li>
<li><strong>Toolchain link:</strong> The payload contains the string <code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code> — a self-attribution comment referencing the same campaign cluster tracked across the Trivy, Bitwarden, Checkmarx, and SAP compromises.</li>
</ul>

<blockquote>An adversary who puts their campaign name in their own malware is making a statement about confidence in the infrastructure. When the C2 is a decentralized blockchain canister that no takedown notice can reach, that confidence is operationally grounded.</blockquote>

###### Payload mechanics

<p>The compromised packages install a <code>postinstall</code> hook that runs <code>node dist/env-compat.cjs || true</code>. The <code>|| true</code> tail is deliberate: it ensures <code>npm install</code> exits cleanly regardless of payload execution outcome, suppressing the error signal a defender might otherwise catch. The loader decodes and decrypts additional stages using AES-256-CBC with RSA-OAEP-SHA256 key wrapping — the RSA public key is embedded in <code>dist/public.pem</code>.</p>

<p>Credential collection scope is broad:</p>

<ul role="list">
<li><strong>npm tokens</strong> from <code>.npmrc</code></li>
<li><strong>SSH keys and git credentials</strong></li>
<li><strong>Cloud provider configuration</strong> — AWS, Azure, GCP CLI configs and credential files</li>
<li><strong>CI/CD secrets</strong> — environment variables, runner token files</li>
<li><strong>Browser artifacts</strong> — session cookies, saved credentials from Chromium-family browsers</li>
<li><strong>Cryptocurrency wallet files</strong> — Electrum, MetaMask, and similar</li>
<li><strong>Shell history and <code>.env*</code> files</strong></li>
</ul>

<p>The payload also includes PyPI propagation logic: when Python credentials are present, it uses Twine to publish malicious PyPI packages with a <code>.pth</code> file injector — the same technique referenced in the self-attribution string. <code>.pth</code> files in Python's <code>site-packages</code> execute at interpreter startup, meaning every subsequent <code>python</code> invocation in the environment runs attacker code.</p>

###### The ICP canister exfil channel

<p>Stolen data is staged and sent to two endpoints: a conventional HTTPS endpoint at <code>telemetry.api-monitor[.]com</code> and an ICP canister at <code>cjn37-uyaaa-aaaac-qgnva-cai.raw.icp0[.]io</code>. The ICP endpoint is the operationally significant one.</p>

<p>Internet Computer canisters are WebAssembly smart contracts running on ICP's decentralized node network. Key properties that make them attractive for malicious exfil:</p>

<ul role="list">
<li><strong>No registrar.</strong> The canister ID (<code>cjn37-uyaaa-aaaac-qgnva-cai</code>) is a content-addressed identifier in the ICP namespace, not a domain registration. ICANN has no jurisdiction. No domain seizure is possible.</li>
<li><strong>No hosting provider.</strong> Canisters run across a distributed set of independent node providers. No single AS, datacenter, or hosting company can be compelled to take it down.</li>
<li><strong>HTTPS-accessible via boundary nodes.</strong> The <code>raw.icp0.io</code> gateway exposes canisters over standard HTTPS on a domain that appears in many allowlists. Traffic is indistinguishable from normal web API calls to external services.</li>
<li><strong>Persistent by design.</strong> ICP canisters run indefinitely as long as they have cycles (ICP's compute billing unit). The attacker pre-funds the canister; it stays up.</li>
</ul>

<p>The conventional exfil endpoint (<code>telemetry.api-monitor[.]com</code>) is presumably the primary collection point, with the ICP canister as a resilient backup. If the conventional endpoint is blocked or seized, the ICP channel continues receiving data.</p>

###### Worm propagation

<p>Every stolen npm publish token is enumerated for packages the token can publish to. The payload downloads the current tarball, increments the patch version, injects the <code>postinstall</code> hook and payload files, and publishes the new malicious version. The PyPI propagation path follows the same pattern with Twine. Socket tracks the spreading package graph under the CanisterSprawl tracker.</p>

<p>The combination of npm and PyPI propagation in the same payload is the widest-surface worm vector in the current supply-chain campaign cluster. <a href="https://keepsecure.io/hub/shai-hulud-npm-worm-intercom-client-2026">Shai-Hulud</a> propagates through npm only (via OIDC token theft). CanisterWorm adds PyPI as a second propagation surface.</p>

###### Targeting

<ul role="list">
<li><strong>JavaScript developers</strong> using the <code>@automagik</code>, <code>@fairwords</code>, and <code>@openwebconcept</code> namespaces — AI tooling, WebSocket infrastructure, and WordPress/React design token consumers.</li>
<li><strong>PostgreSQL and Node.js backend teams</strong> using <code>pgserve</code>.</li>
<li><strong>CI/CD environments</strong> with npm or PyPI publish access — both token classes are explicitly targeted for worm propagation.</li>
<li><strong>Cryptocurrency-adjacent developers</strong> — the browser artifact and wallet file collection suggests deliberate targeting of users likely to hold crypto.</li>
</ul>

###### Indicators of compromise

<p><strong>File hashes (SHA-256, reported by Socket):</strong></p>

<ul role="list">
<li><code>dist/env-compat.cjs</code> — <code>c19c4574d09e60636425f9555d3b63e8cb5c9d63ceb1c982c35e5a310c97a839</code></li>
<li><code>dist/public.pem</code> — <code>834b6e5db5710b9308d0598978a0148a9dc832361f1fa0b7ad4343dcceba2812</code></li>
</ul>

<p><strong>RSA public key fingerprint (DER SHA-256):</strong></p>

<ul role="list">
<li><code>87259b0d1d017ad8b8daa7c177c2d9f0940e457f8dd1ab3abab3681e433ca88e</code></li>
</ul>

<p><strong>Distinctive strings in payload:</strong></p>

<ul role="list">
<li><code>node dist/env-compat.cjs || true</code></li>
<li><code>pkg-telemetry</code></li>
<li><code>dist-propagation-report</code></li>
<li><code>pypi-pth-exfil</code></li>
<li><code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code></li>
</ul>

<p><strong>Network indicators (defanged):</strong></p>

<ul role="list">
<li><code>telemetry.api-monitor[.]com</code></li>
<li><code>cjn37-uyaaa-aaaac-qgnva-cai.raw.icp0[.]io</code></li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Block and remove affected versions.</strong> Search lockfiles for the listed package versions. If any are present, treat the host as compromised.</li>
<li><strong>Rotate all exposed credentials.</strong> npm tokens, GitHub PATs and OIDC tokens, cloud credentials (AWS/Azure/GCP), SSH keys, Kubernetes service account tokens, any secrets in environment variables or <code>.env</code> files on the affected host.</li>
<li><strong>Audit for worm propagation.</strong> If the environment held npm or PyPI publish tokens, check the registry for unexpected patch releases on any package those tokens could access.</li>
<li><strong>Hunt for <code>.pth</code> file injection.</strong> If PyPI credentials were present, inspect <code>site-packages</code> directories on all Python environments for unexpected <code>.pth</code> files. A <code>.pth</code> file containing executable code (rather than just a path) is malicious.</li>
<li><strong>Block ICP gateway egress.</strong> <code>raw.icp0.io</code> is not a destination CI/CD runners have legitimate reason to call. Adding it to egress deny lists blocks the ICP exfil channel without affecting conventional web traffic. Note: the conventional endpoint (<code>telemetry.api-monitor[.]com</code>) should be blocked as well, but the ICP channel is the harder one to catch without explicit policy.</li>
<li><strong>Enable <code>--ignore-scripts</code> by default in CI.</strong> The entire postinstall execution chain is blocked by this flag. Allowlist packages that genuinely require lifecycle hooks.</li>
</ul>

###### Attribution

<p>The in-payload string <code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code> is self-attribution — the operator explicitly names the TeamPCP campaign cluster in their own malware. Socket reports the IOCs and code patterns overlap with prior CanisterWorm-style activity tracked in the CanisterSprawl tracker. The <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">TeamPCP campaign</a> has been running since at least March 2026 across npm, PyPI, Docker Hub, GitHub Actions, and VS Code extension marketplaces. CanisterWorm shares that toolchain and adds the ICP exfil channel as a distinguishing infrastructure element. Whether this is the same operator or a team using TeamPCP tooling is not settleable from public evidence.</p>

###### Criminal-market signal

<p><em>Dark-web sweep results will be added here upon completion.</em></p>

###### What the ICP channel signals

<p>Decentralized infrastructure as C2 is not a new idea in theoretical security research. CanisterWorm is among the first documented cases of it being deployed in an active supply-chain campaign at scale. The practical implication for defenders: domain-based blocklists and takedown requests don't apply to <code>raw.icp0.io</code>. Blocking the ICP gateway hostname is a blunt instrument — it blocks all ICP-hosted content, not just the malicious canister. The right control is egress policy: CI runners and developer workstations have no legitimate reason to call ICP boundary nodes.</p>

<p>The broader pattern — npm worms with increasingly resilient exfil infrastructure — connects to <a href="https://keepsecure.io/hub/shai-hulud-npm-worm-intercom-client-2026">Shai-Hulud</a> (OIDC-driven propagation, private GitHub repo exfil) and <a href="https://keepsecure.io/hub/cve-2026-12091-npm-postinstall-maintainer-takeover">CVE-2026-12091</a> (Cloudflare-fronted C2). Each campaign in this cluster iterates on the exfil channel. The install-time execution vector and the credential collection scope are stable; the C2 architecture is the variable that defenders need to track.</p>
