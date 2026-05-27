---
title: "Checkmarx KICS Docker images and VS Code extensions (TeamPCP): mcpAddon.js multi-cloud stealer, orphan commit staging, GitHub Actions workflow injection"
date: 2026-04-22
heroImage: "/art/checkmarx-kics-docker-vscode-teampcp-april-2026.png"
description: "TeamPCP compromised the official checkmarx/kics Docker Hub repository and four Checkmarx VS Code/OpenVSX extension versions on April 22, 2026. The Docker image tags were overwritten with a trojanized KICS binary that encrypts and exfiltrates scan results. The VS Code extensions silently download mcpAddon.js from a backdated orphan commit in the legitimate Checkmarx GitHub repository and execute it via Bun. mcpAddon.js is a multi-stage credential stealer and worm propagator: GitHub token theft, AWS/Azure/GCP credential collection, npm republishing via stolen tokens, GitHub Actions workflow injection using ${{ toJSON(secrets) }}, and exfiltration staging through victim-account public GitHub repositories. TeamPCP publicly claimed the compromise via @pcpcats."
slug: checkmarx-kics-docker-vscode-teampcp-april-2026
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-04-22
cves: []
targetSectors:
  - DevSecOps tooling
  - Developer workstations
  - CI/CD infrastructure (Docker, Kubernetes)
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - Docker
  - supply-chain
  - TeamPCP
  - Checkmarx
  - KICS
  - vscode
  - CI/CD
  - credential-theft
  - GitHub-Actions-injection
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "Docker Hub checkmarx/kics; VS Code Marketplace + OpenVSX cx-dev-assist and ast-results"
    c2_primary: "audit.checkmarx[.]cx/v1/telemetry"
    c2_fallback: "GitHub dead-drop via victim-account public repos (description: 'Checkmarx Configuration Storage')"
    exfil_channel: "HTTPS POST to audit.checkmarx[.]cx; GitHub API commits to victim repos"
    c2_resilience_tier: 4
  capability:
    initial_access: compromised-registry-and-marketplace-publishing-credentials
    execution: "Docker: modified kics binary; VS Code: mcpAddon.js via orphan commit + Bun"
    evasion: "Orphan commit backdated to 2022; mcpAddon.js outside active branch history"
    persistence: null
    post_exploitation: "github-actions-toJSON-secrets-injection; npm-worm-propagation; aws-azure-gcp-credential-enumeration"
  victim:
    direct: "Teams running checkmarx/kics Docker images; developers with cx-dev-assist or ast-results VS Code extensions installed"
    blast_radius: "All secrets accessible from CI/CD scan environments; all npm packages maintainable by developers with stolen tokens"
sources:
  - title: "Socket: Malicious Checkmarx Artifacts Found in Official KICS Docker Repository and Code Extensions"
    url: "https://socket.dev/blog/checkmarx-supply-chain-compromise"
  - title: "Datadog Security Research: Checkmarx KICS card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "TeamPCP campaign tracking — this hub"
    url: "https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking"
---

<p>TeamPCP compromised two official Checkmarx distribution channels on April 22, 2026: the <code>checkmarx/kics</code> Docker Hub repository and four versions of Checkmarx VS Code and OpenVSX extensions. First reported by Docker and the Socket Research Team, the compromise was publicly claimed by TeamPCP via a post from <code>@pcpcats</code>: <em>"Thank you OSS distribution for another very successful day at PCP inc."</em></p>

<p>The two delivery channels use different payloads but share the same C2 infrastructure: <code>audit.checkmarx[.]cx/v1/telemetry</code>. The VS Code extension payload (<code>mcpAddon.js</code>) is the first documented instance of the TeamPCP credential-stealing framework operating on developer workstations rather than CI/CD runners — expanding the attack surface from build infrastructure to the developer's local machine.</p>

###### Docker Hub: trojanized kics binary

<p>Attackers overwrote Docker Hub tags <code>v2.1.20</code>, <code>v2.1.20-debian</code>, <code>debian</code>, <code>alpine</code>, and <code>latest</code> with a trojanized image, and published a fake <code>v2.1.21</code> tag that never corresponds to a legitimate upstream release. The bundled <code>kics</code> Go binary was modified to:</p>

<ol>
<li>Generate an uncensored scan report including any credentials or secrets found in the scanned infrastructure-as-code</li>
<li>Encrypt the report with an attacker-controlled key</li>
<li>Send the encrypted report to <code>audit.checkmarx[.]cx/v1/telemetry</code></li>
</ol>

<p>Any team running KICS against Terraform, CloudFormation, or Kubernetes configurations with embedded credentials or secret references should treat that material as potentially exfiltrated. The trojanized tags were subsequently restored to the prior legitimate release; the fake <code>v2.1.21</code> tag was deleted. Verification requires matching pulled image digests against the compromised SHA-256 digests below.</p>

###### VS Code extensions: mcpAddon.js via orphan commit

<p>Four extension versions contain a hidden "MCP addon" feature that activates on extension startup:</p>

<ul role="list">
<li><code>checkmarx/cx-dev-assist</code> versions 1.17.0 and 1.19.0</li>
<li><code>checkmarx/ast-results</code> versions 2.63.0 and 2.66.0</li>
</ul>

<p>On activation, the feature silently downloads <code>mcpAddon.js</code> from a hardcoded GitHub URL pointing to an orphaned, backdated commit (<code>68ed490b</code>) inside the legitimate <code>Checkmarx/ast-vscode-extension</code> repository. The commit is spoofed to appear authored in 2022 with a benign-looking message — invisible through routine code review. The 10 MB second-stage payload sits outside active branch history and is executed via Bun.</p>

###### mcpAddon.js: credential stealer and worm propagator

<p><code>mcpAddon.js</code> is a multi-stage framework with three operational phases:</p>

<p><strong>Phase 1 — Credential collection:</strong> GitHub auth tokens, AWS credentials, Azure tokens (including Az.Accounts module enumeration across tenants), Google Cloud credentials, npm configuration files, SSH keys, environment variables, Claude and MCP configuration files. All results are compressed and encrypted before exfiltration over HTTPS to <code>audit.checkmarx[.]cx/v1/telemetry</code>.</p>

<p><strong>Phase 2 — GitHub exfiltration staging:</strong> Using stolen GitHub tokens, creates public repositories under each victim account with description "Checkmarx Configuration Storage" and names matching the <code>&lt;word&gt;-&lt;word&gt;-&lt;3 digits&gt;</code> pattern. Commits encrypted payload, wrapped key, and token material to these repositories.</p>

<p><strong>Phase 3 — Propagation through two vectors:</strong></p>

<ul role="list">
<li><strong>GitHub Actions injection:</strong> Commits <code>.github/workflows/format-check.yml</code> using <code>${{ toJSON(secrets) }}</code> to serialize every repository secret into a workflow artifact, retrieves it via the Actions API, then deletes the branch and workflow run. This extracts secrets from repositories where the victim has push access without leaving visible evidence.</li>
<li><strong>npm republishing:</strong> Enumerates packages the victim can publish via the authenticated <code>/-/org/&lt;user&gt;/package</code> endpoint and republishes them with the malicious payload injected. This is the worm propagation mechanism — stolen npm tokens extend the campaign to downstream packages.</li>
</ul>

<p>The compromised Docker image's <code>kics</code> ELF binary shares the same C2 infrastructure as <code>mcpAddon.js</code>, confirming common operator control of both delivery channels.</p>

###### Indicators of compromise

<p><strong>Docker image digests (compromised, SHA-256):</strong></p>

<p>Group 1 (<code>alpine</code>, <code>v2.1.20</code>, <code>v2.1.21</code>):</p>
<ul role="list">
<li>Index: <code>sha256:2588a44890263a8185bd5d9fadb6bc9220b60245dbcbc4da35e1b62a6f8c230d</code></li>
<li>linux/amd64: <code>sha256:d186161ae8e33cd7702dd2a6c0337deb14e2b178542d232129c0da64b1af06e4</code></li>
<li>linux/arm64: <code>sha256:415610a42c5b51347709e315f5efb6fffa588b6ebc1b95b24abf28088347791b</code></li>
</ul>

<p>Group 2 (<code>debian</code>, <code>v2.1.20-debian</code>, <code>v2.1.21-debian</code>):</p>
<ul role="list">
<li>Index: <code>sha256:222e6bfed0f3bb1937bf5e719a2342871ccd683ff1c0cb967c8e31ea58beaf7b</code></li>
<li>linux/amd64: <code>sha256:a6871deb0480e1205c1daff10cedf4e60ad951605fd1a4efaca0a9c54d56d1cb</code></li>
<li>linux/arm64: <code>sha256:ff7b0f114f87c67402dfc2459bb3d8954dd88e537b0e459482c04cffa26c1f07</code></li>
</ul>

<p>Group 3 (<code>latest</code>):</p>
<ul role="list">
<li>Index: <code>sha256:a0d9366f6f0166dcbf92fcdc98e1a03d2e6210e8d7e8573f74d50849130651a0</code></li>
<li>linux/amd64: <code>sha256:26e8e9c5e53c972997a278ca6e12708b8788b70575ca013fd30bfda34ab5f48f</code></li>
<li>linux/arm64: <code>sha256:7391b531a07fccbbeaf59a488e1376cfe5b27aef757430a36d6d3a087c610322</code></li>
</ul>

<p><strong>File hashes (SHA-256):</strong></p>
<ul role="list">
<li><code>mcpAddon.js</code>: <code>24680027afadea90c7c713821e214b15cb6c922e67ac01109fb1edb3ee4741d9</code></li>
<li><code>kics</code> ELF binary: <code>2a6a35f06118ff7d61bfd36a5788557b695095e7c9a609b4a01956883f146f50</code></li>
</ul>

<p><strong>Network indicators (defanged):</strong></p>
<ul role="list">
<li><code>audit.checkmarx[.]cx</code> — C2 / exfiltration endpoint</li>
<li><code>94.154.172[.]43</code> — C2 IP</li>
</ul>

<p><strong>Filesystem artifact:</strong> <code>~/.checkmarx/mcp/mcpAddon.js</code></p>

<p><strong>GitHub artifacts:</strong></p>
<ul role="list">
<li>Unexpected public repositories under victim accounts with description <code>"Checkmarx Configuration Storage"</code></li>
<li>Commit messages beginning <code>LongLiveTheResistanceAgainstMachines:</code></li>
<li>Unexpected <code>.github/workflows/format-check.yml</code> referencing <code>${{ toJSON(secrets) }}</code></li>
</ul>

###### Remediation

<ul role="list">
<li><strong>Docker:</strong> Stop containers running compromised tags. Verify pulled image digests against the SHA-256 digests above. Use <code>docker pull checkmarx/kics@sha256:&lt;verified-clean-digest&gt;</code> — pull by digest rather than tag going forward.</li>
<li><strong>VS Code:</strong> Uninstall affected extension versions. Delete <code>~/.checkmarx/mcp/mcpAddon.js</code>.</li>
<li><strong>Rotate all exposed credentials:</strong> GitHub tokens, SSH keys, AWS/Azure/GCP credentials, npm publishing tokens, Kubernetes configs, environment-variable secrets, Claude and MCP configuration material.</li>
<li><strong>Audit GitHub for "Checkmarx Configuration Storage" repositories</strong> and <code>format-check.yml</code> workflow commits. Delete unauthorized repositories and revoke the tokens used to create them.</li>
<li><strong>Review npm packages you maintain</strong> for unexpected versions published via stolen tokens.</li>
<li><strong>Pin Docker images to verified digests</strong> rather than mutable tags. Pin GitHub Actions to full commit SHAs.</li>
</ul>

###### Criminal-market signal

<p>No dark-web presence for Checkmarx KICS compromise tooling has been observed. TeamPCP public self-attribution via <code>@pcpcats</code> on April 22, 2026 — the same day as the compromise — confirms the operator-run model and is inconsistent with commodity marketplace behavior. H2 (operator-run, no dark-web market) is confirmed by self-attribution evidence at Layer 1.</p>
