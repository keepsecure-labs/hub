---
title: "The Bun runtime is becoming the malware delivery vehicle of 2026"
date: 2026-04-30
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451ec9f48304bdea104c6a_blog%203.svg"
description: "Two supply-chain compromises in 48 hours both fetch Bun and run an obfuscated credential stealer. Lightning PyPI and four SAP CAP npm packages, both Team PCP."
slug: bun-runtime-supply-chain-stealer-april-2026
type: threat-intel
campaign: "Mini Shai-Hulud"
firstObserved: 2026-04-29
targetSectors:
  - Software development
  - Cloud infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - supply chain
  - npm
  - PyPI
  - Bun
  - credential theft
  - Team PCP
sources:
  - title: "Mini Shai-Hulud has appeared — Aikido"
    url: "https://www.aikido.dev/blog/mini-shai-hulud-has-appeared"
  - title: "Lightning PyPI package compromised — Socket"
    url: "https://socket.dev/blog/lightning-pypi-package-compromised"
  - title: "Datadog Security Research: Lightning PyPI compromised by Bun-based credential stealer"
    url: "https://app.datadoghq.com/security/feed"
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "npm @cap-js/* and mbt packages; PyPI lightning package"
    c2_primary: "GitHub public repos (description: 'A Mini Shai-Hulud has Appeared')"
    c2_fallback: "GitHub commit dead-drop: OhNoWhatsGoingOnWithGitHub magic string"
    exfil_channel: "AES-256-GCM encrypted result files in public GitHub repos"
    c2_resilience_tier: 4
  capability:
    initial_access: account-takeover
    execution: preinstall-hook
    evasion: bun-runtime-loader
    persistence: worm-propagation
  victim:
    direct: "SAP CAP npm maintainers; Lightning AI (PyPI)"
    blast_radius: "@cap-js/sqlite 2.2.2, @cap-js/postgres 2.2.2, @cap-js/db-service 2.10.1, mbt 1.2.48; lightning 2.6.2-2.6.3"
---

<p>In a 48-hour window between April 29 and April 30, 2026, two unrelated-looking supply-chain compromises landed on different package ecosystems with the same novel runtime evasion: both fetch the <a href="https://github.com/oven-sh/bun">Bun</a> JavaScript runtime from GitHub at install time, and both use it to execute an 11-MB obfuscated credential stealer. The npm compromise is four SAP Cloud Application Programming Model packages first reported by Aikido under the name "Mini Shai-Hulud." The PyPI compromise is the popular <code>lightning</code> package, first reported by Socket. A group calling itself Team PCP claimed responsibility for the PyPI side through a Tor onion site linked from a GitHub issue. The shared TTP — Bun as the second-stage runtime — is the headline. Defenders' static analysis tooling is built for Node.js and Python; Bun is neither.</p>

###### What we know

<p>The npm compromise affected four packages with malicious versions published April 29, 2026:</p>

<ul role="list">
<li><strong><code>@cap-js/sqlite</code> v2.2.2</strong></li>
<li><strong><code>@cap-js/postgres</code> v2.2.2</strong></li>
<li><strong><code>@cap-js/db-service</code> v2.10.1</strong></li>
<li><strong><code>mbt</code> v1.2.48</strong></li>
</ul>

<p>The PyPI compromise affected <code>lightning</code> versions <strong>2.6.2</strong> and <strong>2.6.3</strong> on April 30, 2026. Socket reports the package receives several hundred thousand downloads per day from Python machine-learning environments. Version 2.6.1, published January 30, 2026, is reported clean.</p>

<p>The npm payload chain: a <code>preinstall</code> hook in <code>package.json</code> runs <code>node setup.mjs</code>, which detects OS and architecture, downloads <strong>Bun v1.3.13</strong> from GitHub releases, and uses Bun to execute an 11.7-MB obfuscated <code>execution.js</code>. The PyPI payload chain is structurally identical: a <code>start.py</code> bootstrapper added to the package detects host architecture, fetches Bun, and launches a hidden <code>_runtime/router_runtime.js</code> daemon with suppressed output.</p>

<p>The credential-harvest scope is broad — every secret reachable from a developer or CI environment:</p>

<ul role="list">
<li><strong>GitHub tokens</strong> including <code>gh auth token</code> output and Actions tokens.</li>
<li><strong>npm publish tokens</strong> from <code>.npmrc</code>.</li>
<li><strong>GitHub Actions secrets</strong>, extracted by an embedded Python helper that reads <code>/proc</code> memory of the <code>Runner.Worker</code> process.</li>
<li><strong>Cloud credentials</strong> across AWS STS / Secrets Manager / SSM, Azure subscriptions / Key Vault, GCP project identity / Secret Manager.</li>
<li><strong>Kubernetes service account tokens</strong>.</li>
<li><strong>Claude and MCP configuration files</strong>, Azure token caches, GCP token databases, Signal configuration, Electrum wallets, VPN configuration.</li>
</ul>

<p>Exfiltration on the npm side uses GitHub itself as the transport. The malware creates public repositories with randomized names and the description <code>A Mini Shai-Hulud has Appeared</code>, then writes AES-256-GCM encrypted result files (with RSA-wrapped keys) under <code>results/results-&lt;timestamp&gt;-&lt;counter&gt;.json</code>. A propagation channel uses a GitHub commit dead-drop: the malware searches commits for the string <code>OhNoWhatsGoingOnWithGitHub</code>, decodes matching commit messages as base64-encoded GitHub tokens, and uses them to access additional repositories.</p>

<blockquote>The dead-drop pattern is novel. Searching public GitHub commits for a magic string lets the operator distribute new tokens without a fixed C2 domain — every git push by anyone, anywhere, becomes potential infrastructure.</blockquote>

###### Why Bun matters

<p>Picking Bun as the second-stage runtime is the most interesting technical choice. It accomplishes several things simultaneously:</p>

<ul role="list">
<li><strong>Static analysis evasion.</strong> Most npm-ecosystem static-analysis tooling expects Node.js semantics. Bun has different module resolution, different runtime APIs in some cases, and many sandboxes don't model it correctly. PyPI-ecosystem tooling expects Python; running JavaScript via Bun from a Python package is even further off the analysis path.</li>
<li><strong>No interpreter dependency on the victim.</strong> Bun is a single static binary. The malware doesn't need a Node.js or Python runtime that fits its expectations — it brings its own.</li>
<li><strong>Faster than Node for execution.</strong> An 11-MB obfuscated bundle that has to deobfuscate, run a string-scrambling layer, and execute hundreds of credential-collection routines wants the runtime startup overhead to be small enough to complete during <code>npm install</code> without obvious delay. Bun's startup is faster than Node.</li>
</ul>

<p>The obfuscation layer in the npm sample is labeled <code>ctf-scramble-v2</code>. The payload exits on Russian locale settings, daemonizes itself on non-CI machines for persistent harvesting, and detects CI to alter its behavior accordingly.</p>

###### Worm-style propagation through GitHub Actions

<p>Beyond credential theft, the npm payload includes a worm component. When it detects a GitHub Actions release workflow for <code>cap-js/cds-dbs</code>, it can modify package tarballs to inject itself, increment the patch version, and repack the tarball — propagating to whoever installs the next release. It also pushes files into repositories under <code>.vscode/</code> and <code>.claude/</code> paths using commit messages titled <code>chore: update dependencies</code> authored by <code>claude &lt;claude@users.noreply.github.com&gt;</code>. The choice of impersonating Claude commits is opportunistic: any reviewer who sees a Claude-authored commit may dismiss it as agent-generated maintenance.</p>

###### Targeting

<ul role="list">
<li><strong>Software developers using SAP CAP</strong> — anyone whose <code>package.json</code> resolved to the affected versions of <code>@cap-js/sqlite</code>, <code>@cap-js/postgres</code>, <code>@cap-js/db-service</code>, or <code>mbt</code> in the April 29 window.</li>
<li><strong>Python ML developers using <code>lightning</code></strong> — versions 2.6.2 and 2.6.3 from April 30 onward, until pinned to 2.6.1 or to a maintainer-confirmed clean release.</li>
<li><strong>CI/CD runners</strong> running <code>npm install</code> or <code>pip install</code> against either ecosystem during the affected windows. Embedded <code>/proc</code> memory dumping of <code>Runner.Worker</code> means GitHub Actions secrets are explicitly in scope.</li>
<li><strong>Developer workstations</strong> with broad cloud-credential access — AWS, Azure, GCP CLI configurations, Kubernetes contexts, SSH keys.</li>
</ul>

###### TTPs and infrastructure

<ul role="list">
<li><strong>Initial access</strong> — package version takeover (npm + PyPI). The reported vector for the PyPI compromise is consistent with credentials previously stolen from earlier supply-chain incidents, used to publish.</li>
<li><strong>Execution</strong> — preinstall lifecycle hook (npm) or import-time bootstrapper (PyPI) that fetches Bun and runs the second stage.</li>
<li><strong>Persistence</strong> — daemonization on non-CI hosts; immediate exit on CI to maximize secret collection without leaving forensic timestamps.</li>
<li><strong>Defense evasion</strong> — Bun runtime to bypass Node/Python-focused analysis; <code>ctf-scramble-v2</code> string obfuscation; Russian-locale exit.</li>
<li><strong>Command and control</strong> — GitHub itself, via attacker-created public repositories with the Mini Shai-Hulud signature description and AES-256-GCM encrypted result files.</li>
<li><strong>Lateral movement</strong> — GitHub commit dead-drop using the magic string <code>OhNoWhatsGoingOnWithGitHub</code> to discover new tokens; tarball injection in detected GitHub Actions release workflows.</li>
</ul>

###### Indicators of compromise

<p><strong>Affected packages and versions:</strong></p>

<ul role="list">
<li><code>@cap-js/sqlite</code> 2.2.2</li>
<li><code>@cap-js/postgres</code> 2.2.2</li>
<li><code>@cap-js/db-service</code> 2.10.1</li>
<li><code>mbt</code> 1.2.48</li>
<li><code>lightning</code> 2.6.2 and 2.6.3</li>
</ul>

<p><strong>SHA-256 hashes (from <code>@cap-js/sqlite@2.2.2</code>):</strong></p>

<ul role="list">
<li><code>setup.mjs</code> — <code>4066781fa830224c8bbcc3aa005a396657f9c8f9016f9a64ad44a9d7f5f45e34</code></li>
<li><code>execution.js</code> — <code>6f933d00b7d05678eb43c90963a80b8947c4ae6830182f89df31da9f568fea95</code></li>
</ul>

<p><strong>Filesystem and behavioral artifacts:</strong></p>

<ul role="list">
<li>Package contains <code>setup.mjs</code> + <code>execution.js</code> (npm) or <code>start.py</code> + <code>_runtime/router_runtime.js</code> (PyPI).</li>
<li>Bun v1.3.13 download from GitHub releases at install time (egress to <code>github.com/oven-sh/bun/releases</code> from a CI runner that doesn't normally pull Bun).</li>
<li>Public GitHub repositories created with description <code>A Mini Shai-Hulud has Appeared</code> under the victim's account.</li>
<li>Commits authored by <code>claude &lt;claude@users.noreply.github.com&gt;</code> with subject <code>chore: update dependencies</code> touching <code>.vscode/</code> or <code>.claude/</code> paths.</li>
<li>GitHub commit messages containing the literal string <code>OhNoWhatsGoingOnWithGitHub</code>.</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Pin to known-clean versions</strong> immediately. <code>lightning</code> 2.6.1; for the SAP CAP packages, the immediately prior versions per maintainer guidance.</li>
<li><strong>Audit lockfiles</strong> across every active repository. Any resolution to the listed versions during the disclosure window means the install ran the payload.</li>
<li><strong>Treat any developer or CI host that ran the affected versions as fully compromised.</strong> Rotate GitHub tokens, npm publish tokens, AWS / Azure / GCP credentials, Kubernetes service account tokens, SSH keys, and any secret reachable from the host's environment. The malware enumerates broadly; assume everything reachable is exfiltrated.</li>
<li><strong>Search GitHub audit logs</strong> for repository creation by your service accounts with descriptions matching <code>Mini Shai-Hulud</code>, and for commits authored under <code>claude@users.noreply.github.com</code> that you didn't make.</li>
<li><strong>Detect Bun fetches</strong> on hosts that have no business running Bun. Outbound egress to <code>github.com/oven-sh/bun/releases</code> from a CI runner mid-install is a strong signal.</li>
<li><strong>Block <code>preinstall</code> and <code>postinstall</code> by default</strong> in CI via <code>npm ci --ignore-scripts</code>. Allowlist scripts that genuinely need to run.</li>
</ul>

###### Attribution discipline

<p>The PyPI compromise is claimed by a group self-identifying as <strong>Team PCP</strong> on a Tor onion site linked from a GitHub issue on the Lightning project. Socket reports tooling overlap with Shai-Hulud and Mini Shai-Hulud npm campaigns. Claimed Team PCP connections to LAPSUS$ are unverified.</p>

<p>For tracking purposes we use the campaign name <strong>Mini Shai-Hulud</strong> (Aikido's term for the npm cluster) and treat <strong>Team PCP</strong> as a self-applied label, not a confirmed actor identity. Tool-overlap-based attribution to a single operator group is consistent with the public reporting but should not be promoted to confident attribution without further evidence.</p>

###### What this signals

<p>Bun-as-malware-runtime is the durable takeaway. Defenders' static-analysis pipelines are still organized around Node.js for the npm ecosystem and Python for PyPI. A second-stage payload running under a third runtime evades the lane-specific tooling on either side. Expect more of this pattern. Detection engineering should add Bun-egress monitoring (and Deno, and WASM-based runtimes generally) as a category — not because Bun is itself malicious, but because its appearance during package installation in environments that don't legitimately use it is now a strong adversary signal. The Lightning + SAP cluster is the first time this TTP has appeared in two distinct ecosystems within 48 hours; it won't be the last.</p>

<p>This compromise sits in the <a href="https://keepsecure.io/hub/cve-2026-12091-npm-postinstall-maintainer-takeover">same npm-supply-chain pattern</a> documented in CVE-2026-12091 — postinstall lifecycle hooks running attacker-controlled code at every install — but with a more sophisticated runtime-evasion layer on top. The structural defense is identical: <code>--ignore-scripts</code> by default, sandboxed install boundaries, capability-narrowed CI tokens. Pattern-by-pattern patching of individual incidents loses to the structural fix.</p>

<p>One last note on the threat-intelligence shape. As of publication, the Mini Shai-Hulud cluster has no observable presence in commodity criminal markets — no broker pricing, no exploit-kit packaging, no forum trade volume. That tracks with how this kind of bug class actually monetizes: the operator IS the toolchain author, distributing through legitimate package registries directly rather than selling through .onion forums. The defensive lesson is the same one <a href="https://keepsecure.io/hub/ai-ide-marketplace-security-telemetry">we made earlier for the AI-IDE marketplace surface</a> — when the attacker can publish to npm or PyPI under a plausible name, dark-web monitoring is the wrong instrument. Marketplace-side telemetry, lockfile diffs, and CI install instrumentation are.</p>

<p><strong>Update, May 4, 2026:</strong> The April 29 packages were the worm's first propagation step, not its conclusion. OIDC tokens stolen from the mbt and @cap-js/sqlite pipelines were used to publish <code>intercom-client@7.0.4</code> the following day — ~2M weekly downloads, private-repo exfiltration, expanded multi-cloud credential sweep. <a href="https://keepsecure.io/hub/shai-hulud-npm-worm-intercom-client-2026">Full analysis of the Shai-Hulud propagation loop.</a></p>
