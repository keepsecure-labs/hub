---
title: "Shai-Hulud closes the loop: how the worm reached intercom-client in 24 hours"
date: 2026-05-04
heroImage: "/art/shai-hulud-npm-worm-intercom-client-2026.png"
description: "The Shai-Hulud worm closed its loop in 24 hours: OIDC tokens from April 29 npm victims published intercom-client@7.0.4 the next day."
slug: shai-hulud-npm-worm-intercom-client-2026
type: threat-intel
campaign: "Shai-Hulud"
firstObserved: 2026-04-29
targetSectors:
  - Developer tooling
  - SaaS
  - Customer success platforms
  - Cloud infrastructure
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
  - OIDC
  - credential theft
  - TeamPCP
hasArtifacts: false
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "npm OIDC publisher via stolen GitHub Actions tokens from mbt, @cap-js/sqlite"
    c2_primary: "api.github[.]com (private repo creation under victim account)"
    c2_fallback: null
    exfil_channel: "AES-256-GCM encrypted files in private GitHub repos"
    c2_resilience_tier: 4
  capability:
    initial_access: oidc-token-theft
    execution: preinstall-hook
    evasion: bun-runtime-loader
    persistence: worm-propagation
  victim:
    direct: "Intercom (intercom-client npm package maintainer)"
    blast_radius: "~2M weekly downloads, versions 7.0.4 only (April 30 window)"
sources:
  - title: "Shai-Hulud worm pivots to multi-cloud: intercom-client hijacked — StepSecurity"
    url: "https://www.stepsecurity.io/blog/shai-hulud-worm-pivots-to-multi-cloud-intercom-client-hijacked"
  - title: "GitHub issue: intercom-client@7.0.4 contains malicious code — StepSecurity"
    url: "https://github.com/intercom/intercom-node/issues/518"
  - title: "Mini Shai-Hulud has appeared — Aikido"
    url: "https://www.aikido.dev/blog/mini-shai-hulud-has-appeared"
  - title: "Datadog Security Research: Shai-Hulud worm compromises intercom-client"
    url: "https://app.datadoghq.com/security/feed"
---

<p>The worm's propagation loop closed in 24 hours. On April 29, 2026, the Shai-Hulud campaign compromised four SAP CAP npm packages and used those victims' CI/CD pipelines to steal GitHub Actions OIDC publishing tokens. By 14:41 UTC on April 30, those stolen tokens published <code>intercom-client@7.0.4</code> — a package with roughly two million weekly downloads — as the worm's next infected host. Each victim becomes the next launchpad; each stolen npm publish token extends the worm's reach to every package that token can access. A self-propagating npm supply chain attack is no longer theoretical. <a href="https://keepsecure.io/hub/bun-runtime-supply-chain-stealer-april-2026">The April 29 wave we tracked as Mini Shai-Hulud</a> wasn't the campaign's conclusion — it was its first propagation step.</p>

###### What we know

<ul role="list">
<li><strong>April 29, ~04:00 UTC</strong> — <code>mbt@1.2.48</code> and <code>@cap-js/sqlite@2.2.2</code> published with Bun-based credential stealer payloads. GitHub Actions OIDC publishing tokens from these packages' CI pipelines are stolen by the payload as it executes on compromised environments.</li>
<li><strong>April 30, 14:41 UTC</strong> — <code>intercom-client@7.0.4</code> published via npm OIDC publisher <code>npm-oidc-no-reply@github.com</code> using a token stolen from the April 29 victims. Package size jumps from 6 MB to 17.8 MB. SLSA v1 provenance attestations present in <code>7.0.3</code> are absent in <code>7.0.4</code>. A <code>preinstall</code> hook appears for the first time in the package's history.</li>
<li><strong>April 30</strong> — StepSecurity identifies and reports the compromise. The malicious version is removed from the npm registry. Safe version: <code>intercom-client@7.0.3</code>.</li>
</ul>

<blockquote>The SLSA attestation gap is the detection signal that mattered here. StepSecurity's tooling flagged the attestation absence before payload analysis confirmed the compromise. A package that has shipped provenance for every prior release dropping it on a patch bump is a high-confidence indicator — worth automating into CI gates.</blockquote>

###### How the worm propagates

<p>The payload runs in four stages. Stages 1–3 are the credential theft; Stage 4 is what makes this a worm rather than a stealer.</p>

<ul role="list">
<li><strong>Stage 1 — Preinstall hook.</strong> A new <code>preinstall</code> script in <code>package.json</code> runs <code>node setup.mjs</code> before any install logic executes.</li>
<li><strong>Stage 2 — Bun runtime loader.</strong> <code>setup.mjs</code> detects OS and architecture, downloads Bun v1.3.13 from GitHub releases, and uses it to execute <code>router_runtime.js</code> — an 11.7 MB single-line obfuscated file (zero newlines, structural obfuscation). Running under Bun instead of Node evades EDR and SIEM rules tuned for suspicious <code>node</code> child processes spawned during package install. The payload daemonizes itself by forking a detached child process with <code>__DAEMONIZED=1</code>, breaking process-tree correlation between <code>npm install</code> and the credential theft that follows.</li>
<li><strong>Stage 3 — Multi-cloud credential sweep.</strong> The payload queries the AWS Instance Metadata Service at <code>169.254.169[.]254</code> for IAM role credentials; queries <code>metadata.google[.]internal</code> for GCP service account tokens; scans for Azure storage connection strings (<code>AccountKey</code>), client secrets, and access keys; extracts PEM-encoded private keys and common secret variable names. All stolen material is exfiltrated to a <em>private</em> repository created under the victim's own GitHub account — all traffic to <code>api.github.com</code>, which is allowlisted in virtually every corporate firewall and CI/CD egress policy.</li>
<li><strong>Stage 4 — Worm propagation.</strong> Every stolen npm publish token is used to enumerate packages the token can publish to, increment the patch version, inject the <code>preinstall</code> hook and payload files into the tarball, and publish a new malicious version. This is the step that turned the April 29 SAP victims into the April 30 intercom-client launchpad.</li>
</ul>

###### What evolved from April 29

<p>Two changes distinguish the intercom-client payload from the Mini Shai-Hulud wave, both in the direction of operational quietness:</p>

<ul role="list">
<li><strong>Exfiltration moved from public to private repos.</strong> The April 29 payloads created public GitHub repositories with the description <code>A Mini Shai-Hulud has Appeared</code> — a detectable public signal. The intercom-client payload creates <em>private</em> repositories. Repository-monitoring approaches that worked against the April 29 wave no longer apply. The OPSEC iteration happened within a single propagation cycle, within 24 hours.</li>
<li><strong>Multi-cloud credential sweep expanded.</strong> The April 29 campaign focused primarily on GitHub tokens, npm publish tokens, and GitHub Actions secrets. The intercom-client payload adds explicit AWS IMDS queries, GCP metadata service lookups, and Azure-specific connection string patterns. The target surface grew with the target audience: intercom-client users are more likely to be cloud-native SaaS operators than SAP CAP developers.</li>
</ul>

###### Targeting

<ul role="list">
<li><strong>B2B SaaS and customer success teams</strong> — intercom-client is the official Intercom Node.js SDK, used across customer support, onboarding, and CX platforms. Roughly two million weekly downloads. Any team that installed it via <code>npm install</code> or <code>npm ci</code> during the April 30 window without a lockfile pinning <code>7.0.3</code> ran the payload.</li>
<li><strong>CI/CD runners with npm publish access</strong> — environments that installed the malicious version and held npm publish tokens for other packages are the worm's next propagation targets. Every package those tokens can publish is the next candidate for a malicious patch release.</li>
<li><strong>Environments with cloud credential access</strong> — AWS EC2 instances without IMDSv2 enforcement, GCP workloads with reachable metadata, Azure environments with connection strings in environment variables.</li>
</ul>

###### Indicators of compromise

<p><strong>Compromised package:</strong></p>

<ul role="list">
<li><code>intercom-client@7.0.4</code> — malicious. Safe version: <code>intercom-client@7.0.3</code>.</li>
</ul>

<p><strong>File hashes (SHA-256):</strong></p>

<ul role="list">
<li><code>setup.mjs</code> — <code>fe64699649591948d6f960705caac86fe99600bf76e3eae29b4517705a58f0e2</code></li>
<li><code>router_runtime.js</code> — <code>5ae8b2343e97cc3b2c945ec34318b63f27fa2db1e3d8fbaa78c298aa63db52ed</code></li>
</ul>

<p><strong>Package-level detection signals:</strong></p>

<ul role="list">
<li>Package size: 6 MB → 17.8 MB in a single patch bump</li>
<li>SLSA v1 provenance attestations present in <code>7.0.3</code>, absent in <code>7.0.4</code></li>
<li><code>preinstall</code> hook introduced for the first time in the package's publish history</li>
<li><code>router_runtime.js</code> is a single 11.7 MB line with zero newlines</li>
</ul>

<p><strong>Behavioral signals:</strong></p>

<ul role="list">
<li>Bun v1.3.13 download from <code>github.com/oven-sh/bun/releases</code> during <code>npm install</code> on a host that doesn't legitimately use Bun</li>
<li>Private GitHub repository creation by CI service accounts in the window immediately following an install step</li>
<li>Detached child processes with <code>__DAEMONIZED=1</code> environment variable surviving beyond <code>npm install</code> exit</li>
<li>Outbound connections to <code>169.254.169[.]254</code> or <code>metadata.google[.]internal</code> from a process not normally querying instance metadata</li>
</ul>

<p><strong>Related April 29 packages (same campaign):</strong></p>

<ul role="list">
<li><code>mbt@1.2.48</code></li>
<li><code>@cap-js/sqlite@2.2.2</code></li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Pin to the safe version.</strong> Downgrade to <code>intercom-client@7.0.3</code> and run <code>npm cache clean --force</code>. Verify lockfiles across every active repository.</li>
<li><strong>Rotate all exposed credentials.</strong> If <code>intercom-client@7.0.4</code> was installed in any environment, treat all credentials reachable from that environment as compromised: AWS IAM credentials, GCP service account tokens, Azure connection strings and client secrets, GitHub tokens (PAT, OAuth, and OIDC), npm publish tokens, SSH keys, and any API keys in environment variables or config files.</li>
<li><strong>Audit for worm propagation.</strong> If the compromised environment held npm publish access to other packages, check the npm registry for unexpected patch bumps. Compare published tarballs to git tags for any package the environment's tokens could access.</li>
<li><strong>Review GitHub audit logs</strong> for private repository creation by service accounts in windows immediately following CI install steps. Cross-reference with install logs from April 30 onward.</li>
<li><strong>Enforce IMDSv2 on all EC2 instances and container workloads.</strong> Require IMDSv2 (hop limit = 1) to prevent unauthenticated IMDS credential queries. This blocks Stage 3 AWS harvesting without payload analysis.</li>
<li><strong>Enable <code>npm install --ignore-scripts</code> by default in CI.</strong> Explicitly allowlist packages that require lifecycle hooks. This is the structural defense that blocks the entire preinstall vector — <a href="https://keepsecure.io/hub/cve-2026-12091-npm-postinstall-maintainer-takeover">CVE-2026-12091</a>, Mini Shai-Hulud, and intercom-client all run through it.</li>
<li><strong>Gate on SLSA attestations.</strong> Any patch release that drops provenance should trigger investigation before install. Automate this check in CI rather than treating it as a manual review item.</li>
</ul>

###### Attribution

<p>The <code>router_runtime.js</code> payload carries the toolchain fingerprint tracked across the broader <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">TeamPCP campaign</a>: the <code>__decodeScrambled</code> PBKDF2 obfuscation cipher appears 232 times, the same count and structure as payloads in the Trivy, Bitwarden, Checkmarx, xinference, and April 29 SAP/Lightning compromises. That's shared tooling. Whether it means a single operator, a toolchain sold or shared between groups, or a fork is not settleable from public evidence. We use "Shai-Hulud" as the campaign name for the self-propagating npm worm and treat the TeamPCP toolchain overlap as a linking indicator, not a hard attribution.</p>

###### Criminal-market signal

<p>We ran comprehensive dark-web sweeps across 19+ live onion search engines for both <code>shai-hulud worm</code> and <code>intercom-client npm</code> on May 4, 2026 — five days after the intercom-client compromise was disclosed. Both returned clean negatives.</p>

<ul role="list">
<li><strong><code>shai-hulud worm</code></strong> — 23 pages analyzed: 5 exploit forums, 16 marketplaces, 2 other. Zero findings. No broker pricing, no actor discussion, no IOC references.</li>
<li><strong><code>intercom-client npm</code></strong> — 76 pages analyzed: 13 exploit forums, 59 marketplaces, 3 news mirrors. Zero findings on the supply-chain compromise specifically. The unrelated CVE mentions surfaced in results (CVE-2023-38545, CVE-2024-3094, CVE-2021-44228) confirm the forums are active and indexed — the clean negative is genuine, not a search failure.</li>
</ul>

<p>This matches the pattern established across every Shai-Hulud-adjacent campaign we've swept. The Mini Shai-Hulud wave (19 pages, 2 exploit forums, 16 marketplaces) returned clean in April. The TeamPCP campaign (18 pages, 2 exploit forums, 15 marketplaces) returned clean in March. Three sweeps, three clean negatives, same campaign cluster.</p>

<blockquote>The contrast with <a href="https://keepsecure.io/hub/copyfail-time-to-criminalization-seven-days">CopyFail</a> is instructive. CVE-2026-31431 crossed to a carding forum's Exploits section in seven days. Shai-Hulud, after five days and 76 pages of active forum coverage, has no criminal-market presence. The difference is the business model: CopyFail is a reusable primitive that any ransomware operator can compose with any foothold. Shai-Hulud is the operator's own delivery infrastructure — they're not selling the capability, they're using it. There's no market because the attacker and the tool are the same entity.</blockquote>

<p>This shapes where defenders should look. The Shai-Hulud threat signal lives in SLSA attestation gaps, npm publish audit logs, package size anomalies, and CI egress to GitHub releases endpoints — not in .onion marketplaces. The news mirrors that did appear in the intercom-client sweep were clearnet security coverage reaching Tor relay nodes; the exploit forums returned nothing. Monitor the registry, not the forum.</p>

###### What the 24-hour loop means

<p>The propagation velocity is the finding. The worm iterated its own OPSEC — moving from public to private exfil repos — within a single propagation cycle. The multi-cloud sweep expanded to match the new victim profile. Neither change required human intervention; they were already in the payload, triggered by which environment the worm landed in next. That's an automated adversarial adaptation loop running faster than most incident response timelines.</p>

<p>The structural defense is the same one that applied to <a href="https://keepsecure.io/hub/cve-2026-12091-npm-postinstall-maintainer-takeover">CVE-2026-12091</a> and Mini Shai-Hulud: <code>--ignore-scripts</code> by default, lockfile pinning, IMDSv2 enforcement, SLSA provenance as a publish-time gate. None of that is new. What changed is the timeline defenders need to operate on: 24 hours from initial compromise to two-million-download package infected is not a CVE-cadence problem. It's an automated response problem.</p>

<p>SLSA attestation gaps, Bun egress from CI runners that don't use it, and unexpected patch bumps on packages your tokens can publish are the signals worth automating. The worm's five-day-old presence across 76 dark-web pages left no trace. Its presence in the npm registry left size metadata, a missing provenance attestation, and a new preinstall hook — all detectable before payload execution.</p>
