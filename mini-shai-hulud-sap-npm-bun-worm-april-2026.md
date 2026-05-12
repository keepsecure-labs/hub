---
title: "Mini Shai-Hulud: SAP CAP npm compromise confirms Bun v1.3.13 cluster link and introduces MCP credential targeting"
date: 2026-05-12
heroImage: "/art/mini-shai-hulud-sap-npm-bun-worm-april-2026.png"
description: "Four SAP Cloud Application Programming Model npm packages were compromised on April 29, 2026 with a Bun v1.3.13 credential stealer. This is Mini Shai-Hulud — named by the operator in the exfil repository description. Two findings make this the pivotal event in the npm worm cluster: (1) Bun v1.3.13 pin is confirmed in primary-source analysis, closing the linking-artifact gap in the cluster article; (2) the OIDC tokens stolen from mbt@1.2.48 on April 29 were used to publish the malicious intercom-client@7.0.4 on April 30 — Mini Shai-Hulud is causally upstream of the Shai-Hulud intercom-client compromise."
slug: mini-shai-hulud-sap-npm-bun-worm-april-2026
type: threat-intel
campaign: "Mini Shai-Hulud / Team PCP"
firstObserved: 2026-04-29
targetSectors:
  - Enterprise software development
  - SAP ecosystem
  - Software development
targetRegions:
  - Global (Russia excluded by CI detection and locale check)
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - supply chain
  - Mini Shai-Hulud
  - TeamPCP
  - Bun runtime
  - SAP CAP
  - MCP credentials
  - GitHub dead-drop
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "npm — @cap-js/sqlite v2.2.2, @cap-js/postgres v2.2.2, @cap-js/db-service v2.10.1, mbt v1.2.48"
    c2_primary: "Public GitHub repositories with description 'A Mini Shai-Hulud has Appeared'; AES-256-GCM encrypted result files"
    c2_fallback: null
    exfil_channel: "AES-256-GCM encrypted files committed to attacker-created public GitHub repos; RSA-OAEP-wrapped session keys"
    c2_resilience_tier: 4
  capability:
    initial_access: npm-preinstall-hook
    execution: bun-runtime-loader
    evasion: Bun-runtime + Russian-locale-exit + CI-detection + daemon-fork
    persistence: daemonizes-on-non-CI
  victim:
    direct: "SAP CAP developers and CI/CD pipelines installing affected package versions"
    blast_radius: "SAP CAP ecosystem; OIDC tokens stolen from mbt@1.2.48 causally enabled intercom-client@7.0.4 compromise on April 30"
sources:
  - title: "Aikido Security: Mini Shai-Hulud has Appeared"
    url: "https://www.aikido.dev/blog/mini-shai-hulud-has-appeared"
  - title: "Datadog Security Research: Malicious SAP npm packages card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "Shai-Hulud worm analysis — this hub"
    url: "https://keepsecure.io/hub/shai-hulud-npm-worm-intercom-client-2026"
  - title: "npm supply-chain worm cluster analysis — this hub"
    url: "https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026"
---

<p>Four SAP Cloud Application Programming Model npm packages were published with malicious versions on April 29, 2026: <code>@cap-js/sqlite</code> v2.2.2, <code>@cap-js/postgres</code> v2.2.2, <code>@cap-js/db-service</code> v2.10.1, and <code>mbt</code> v1.2.48. The operator named the campaign in the exfil repository description: <em>"A Mini Shai-Hulud has Appeared."</em> Aikido Security published the primary analysis the same day.</p>

<p>This finding documents Mini Shai-Hulud as a standalone event, not as a footnote in the intercom-client story. The intercom-client compromise (<a href="https://keepsecure.io/hub/shai-hulud-npm-worm-intercom-client-2026">Shai-Hulud, April 30</a>) is covered separately. The two events are causally linked: GitHub Actions OIDC tokens stolen from <code>mbt@1.2.48</code> on April 29 were used to publish the malicious <code>intercom-client@7.0.4</code> on April 30. Mini Shai-Hulud is the upstream event.</p>

<p>Two things make Mini Shai-Hulud the pivotal finding in the cluster:</p>

<ol>
<li><strong>Bun v1.3.13 is confirmed.</strong> Aikido's primary analysis explicitly identifies Bun v1.3.13 in <code>setup.mjs</code>. This closes the open question in the <a href="https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026">npm worm cluster article</a>: the v1.3.13 pin is a confirmed Layer 1 linking artifact connecting Mini Shai-Hulud, Shai-Hulud, and the TeamPCP toolkit.</li>
<li><strong>The propagation marker is documented.</strong> <code>OhNoWhatsGoingOnWithGitHub</code> — the string the worm uses to locate previously-stolen tokens in GitHub commit messages — is a detection artifact not previously recorded in any hub finding. This is how the worm finds credentials left by earlier victims to extend its reach.</li>
</ol>

###### What we know

<ul role="list">
<li><strong>April 29, 2026</strong> — four SAP CAP npm packages published with malicious versions. Aikido Security primary analysis published the same day.</li>
<li><strong>Naming:</strong> exfil repositories carry the description <em>"A Mini Shai-Hulud has Appeared"</em> — operator self-attribution consistent with the Dune-themed campaign branding across the cluster.</li>
<li><strong>Causal link to Shai-Hulud:</strong> Datadog Security Research notes that <code>mbt@1.2.48</code> and <code>@cap-js/sqlite@2.2.2</code> are the earlier victims whose stolen OIDC tokens enabled the intercom-client hijack on April 30. The worm propagated from SAP CAP to Intercom in under 24 hours.</li>
</ul>

###### Why SAP CAP was targeted

<p>SAP Cloud Application Programming Model is an enterprise Node.js/Java framework used in organizations running SAP Business Technology Platform. The ecosystem profile differs from the ML/AI credential surface targeted in litellm and lightning:</p>

<ul role="list">
<li><strong>Enterprise cloud credentials with broad scope.</strong> SAP CAP deployments on BTP hold credentials for AWS, Azure, and GCP service integrations alongside SAP-specific service bindings. CI/CD pipelines for SAP projects carry the same credential surface as any enterprise cloud-connected pipeline.</li>
<li><strong>npm publish tokens in enterprise CI/CD.</strong> SAP CAP projects use npm for package management. Development pipelines hold npm tokens with publish scope — the exact token class the worm uses for propagation.</li>
<li><strong>GitHub Actions OIDC tokens.</strong> SAP CAP projects hosted on GitHub using Actions-based CI/CD produce short-lived OIDC tokens that can be exchanged for npm publish credentials. The worm specifically targets these tokens: <code>mbt@1.2.48</code>'s stolen OIDC token was used to publish <code>intercom-client@7.0.4</code>.</li>
<li><strong>AI/ML tooling now in enterprise contexts.</strong> The collection scope includes MCP server configuration files and Claude API tokens — reflecting that the operator updated their credential sweep to target AI engineering tooling now present in enterprise developer environments.</li>
</ul>

###### Payload mechanics

<p>The following analysis is from Aikido Security's primary report, as characterized by Datadog Security Research.</p>

<p><strong>Stage 1 — <code>setup.mjs</code>:</strong> The bootstrapper runs via a <code>preinstall</code> hook, detects OS and architecture, downloads <strong>Bun v1.3.13</strong> from GitHub, and invokes <code>execution.js</code> under the Bun runtime. The explicit version pin <code>v1.3.13</code> is the same pin used in the Shai-Hulud <code>intercom-client@7.0.4</code> payload — confirmed Layer 1 linking artifact.</p>

<p><strong>Stage 2 — <code>execution.js</code>:</strong> An 11.7 MB obfuscated JavaScript payload using a custom string scrambling layer labeled <strong><code>ctf-scramble-v2</code></strong>. This label is a new Layer 1 artifact: it is the operator's internal identifier for their obfuscation layer. The payload:</p>

<ul role="list">
<li>Checks for Russian locale settings — exits if detected</li>
<li>Detects whether it is running in CI — daemonizes itself on non-CI machines for persistent harvesting</li>
<li>Forks a detached child process with <code>__DAEMONIZED=1</code> to break process-tree correlation with <code>npm install</code></li>
</ul>

<p><strong>Collection scope</strong> — the payload harvests a broader credential surface than earlier TeamPCP PyPI victims:</p>

<ul role="list">
<li>GitHub tokens (including output from <code>gh auth token</code>) and npm tokens from <code>.npmrc</code></li>
<li>GitHub Actions secrets — an embedded Python helper reads <code>/proc</code> memory of the <code>Runner.Worker</code> process to extract secrets that are not written to disk</li>
<li>AWS STS identity, Secrets Manager secrets, and SSM Parameter Store parameters</li>
<li>Azure subscriptions, Key Vault names, and Key Vault secret values</li>
<li>GCP project identity and Secret Manager values</li>
<li>Kubernetes service account tokens</li>
<li><strong>MCP server configuration files and Claude API tokens</strong> — new collection target in this payload; the operator has updated the credential sweep to include AI development tooling</li>
<li>Azure token caches, GCP token databases, Signal configuration, Electrum wallets, VPN configuration files</li>
</ul>

<blockquote>The GitHub Actions Runner.Worker memory read deserves specific attention. GitHub Actions secrets are designed to never appear in logs — they are redacted. But the runner process holds them in memory. A Python helper that reads <code>/proc/[pid]/mem</code> of the runner process can extract secrets that the GitHub Actions security model explicitly assumes are not accessible. This is not a GitHub vulnerability — it is a privileged code execution on the runner extracting in-memory data. The defense is preventing the malicious code from executing, not hardening secret redaction.</blockquote>

<p><strong>Exfiltration:</strong> The malware creates public GitHub repositories with randomized names and the description <em>"A Mini Shai-Hulud has Appeared"</em>. Stolen data is written as AES-256-GCM encrypted files under <code>results/results-&lt;timestamp&gt;-&lt;counter&gt;.json</code>, with the AES session key wrapped under an embedded RSA public key. All exfil traffic goes to <code>api.github.com</code>.</p>

<p><strong>Propagation mechanism:</strong> The worm propagates via two paths:</p>

<ol>
<li><strong>GitHub commit dead-drop discovery.</strong> The malware searches GitHub commit messages for the string <strong><code>OhNoWhatsGoingOnWithGitHub</code></strong>. Matching commits contain base64-encoded GitHub tokens left by prior victims. The worm decodes and uses these tokens to access additional repositories — this is how the worm grows its token pool without direct C2 communication.</li>
<li><strong>Package tarball injection.</strong> When the worm detects a GitHub Actions release workflow for <code>cap-js/cds-dbs</code>, it modifies the package tarball to inject a malicious <code>postinstall</code> hook, increments the patch version, and repacks. It also pushes files into repositories under <code>.vscode/</code> and <code>.claude/</code> paths using commit messages titled <code>chore: update dependencies</code> — designed to blend into normal dependency maintenance commits.</li>
</ol>

###### Bun v1.3.13: cluster linking artifact confirmed

<p><strong>Observation:</strong> Aikido's primary analysis of Mini Shai-Hulud's <code>setup.mjs</code> identifies Bun v1.3.13 as the pinned version. The Shai-Hulud <code>intercom-client@7.0.4</code> payload uses the same Bun v1.3.13 pin (confirmed in StepSecurity's primary analysis). The TeamPCP npm cluster analysis identified this pin as the proposed linking artifact but noted it required primary-source confirmation.</p>

<p><strong>Mechanism:</strong> Pinning to a specific Bun point release is a deliberate build choice. v1.3.13 is not an LTS release, a current stable release, or a conventionally-adopted version. Independent codebases would not pin to the same non-current point release without coordinating on the same build system. The pin is reproducible only if both payloads originated from the same codebase — or from a codebase explicitly copying the pin.</p>

<p><strong>Inference:</strong> Mini Shai-Hulud and Shai-Hulud share a build-time codebase. They are the same toolkit, not similar toolkits. The npm worm cluster linking artifact is confirmed at Layer 1.</p>

<p><strong>Confidence bound:</strong> This would weaken if v1.3.13 were shown to be a widely-adopted convention in the npm malware ecosystem at the time — i.e., if other unrelated actors were independently pinning to this version. No evidence of that pattern exists.</p>

###### Indicators of compromise

<p><strong>Compromised package versions:</strong></p>
<ul role="list">
<li><code>@cap-js/sqlite</code> v2.2.2</li>
<li><code>@cap-js/postgres</code> v2.2.2</li>
<li><code>@cap-js/db-service</code> v2.10.1</li>
<li><code>mbt</code> v1.2.48</li>
</ul>

<p><strong>Filesystem artifacts:</strong></p>
<ul role="list">
<li><code>setup.mjs</code> in package root — bootstrapper; downloads Bun v1.3.13</li>
<li><code>execution.js</code> in package root — 11.7 MB obfuscated <code>ctf-scramble-v2</code> payload</li>
<li>Bun v1.3.13 binary in a temp or hidden path</li>
</ul>

<p><strong>Network indicators:</strong></p>
<ul role="list">
<li>Outbound connection to GitHub API (<code>api.github.com</code>) from npm install context — repository creation and commit operations not expected from package installation</li>
<li>Bun runtime download from GitHub during <code>npm install</code></li>
</ul>

<p><strong>GitHub artifacts:</strong></p>
<ul role="list">
<li>Public repositories with description <strong><em>"A Mini Shai-Hulud has Appeared"</em></strong> — operator self-attribution; any such repository in your GitHub org or fork network is evidence of worm activity</li>
<li>GitHub commit messages containing <strong><code>OhNoWhatsGoingOnWithGitHub</code></strong> — the propagation token marker; search your GitHub audit logs for this string</li>
<li>Commits with message <code>chore: update dependencies</code> adding files under <code>.vscode/</code> or <code>.claude/</code> paths — worm persistence injection</li>
</ul>

<p><strong>Campaign self-attribution:</strong></p>
<ul role="list">
<li><em>"A Mini Shai-Hulud has Appeared"</em> — exfil repository description</li>
<li><code>ctf-scramble-v2</code> — internal obfuscation layer label in <code>execution.js</code></li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Update all four packages immediately.</strong> The malicious versions are the only compromised releases. Any environment that ran <code>npm install</code> with these versions in the dependency tree should be treated as potentially compromised.</li>
<li><strong>Search GitHub audit logs for <code>OhNoWhatsGoingOnWithGitHub</code>.</strong> This string in any commit message in your organization's repositories is a worm propagation indicator. If found, the associated commit contains a base64-encoded stolen GitHub token — rotate all tokens in that repository.</li>
<li><strong>Search for repositories with description "A Mini Shai-Hulud has Appeared".</strong> Any such repository under your organization or in your fork network was created by the worm's exfil routine.</li>
<li><strong>Rotate GitHub tokens, npm tokens, and all cloud credentials.</strong> The Runner.Worker memory extraction means secrets not written to disk may have been collected. Treat all CI/CD secrets from affected pipelines as compromised.</li>
<li><strong>Audit <code>.vscode/</code> and <code>.claude/</code> for injected files.</strong> The worm pushes files into these paths as a secondary persistence mechanism. Any unexpected files in these directories committed by automated workflows warrant investigation.</li>
<li><strong>Inspect MCP configuration and Claude API tokens.</strong> This is the first confirmed TeamPCP campaign to target MCP server configuration. If your CI/CD environment holds Claude API keys or MCP server definitions, rotate them.</li>
<li><strong>Detect Bun runtime download during npm install.</strong> Any process downloading Bun v1.3.13 from GitHub during <code>npm install</code> is a worm indicator. This behavioral rule covers Mini Shai-Hulud, Shai-Hulud, and the broader TeamPCP toolkit.</li>
</ul>

###### Attribution and cluster position

<p>Mini Shai-Hulud is the direct precursor to the Shai-Hulud intercom-client compromise. The causal chain is: Mini Shai-Hulud (April 29) steals OIDC tokens from <code>mbt@1.2.48</code> → those tokens are used to publish <code>intercom-client@7.0.4</code> (April 30, Shai-Hulud). The Bun v1.3.13 confirmation links both events to the TeamPCP toolkit. The operator named both with Dune-themed branding: Mini Shai-Hulud (SAP CAP precursor) and Shai-Hulud (intercom propagation).</p>

<p>See the <a href="https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026">npm supply-chain worm cluster analysis</a> for the full cluster timeline and cross-campaign technical analysis. The Bun v1.3.13 confirmation documented here updates the cluster article's open question on the linking artifact.</p>

###### Criminal-market signal

<p>No dark-web presence for Mini Shai-Hulud tooling, the SAP CAP packages, or the <em>"A Mini Shai-Hulud has Appeared"</em> campaign infrastructure has been observed. The operator-run credential-collection model confirmed across the cluster applies. The MCP/Claude configuration targeting is consistent with an operator who uses the credentials directly — a commodity actor would not invest in targeting AI development tooling that requires context to exploit. H2 operator-run pattern confirmed.</p>
