---
title: "One toolchain, six weeks, four campaigns: mapping the npm supply-chain worm cluster"
date: 2026-05-07
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451ed0939d3c19cc367822_blog%204.svg"
description: "TeamPCP, Mini Shai-Hulud, Shai-Hulud, and CanisterWorm are four names for one connected operation. The shared __decodeScrambled PBKDF2 fingerprint, Bun runtime evasion, and worm propagation architecture trace to a single toolchain that upgraded its C2 resilience twice and its OPSEC once — within a single propagation cycle."
slug: npm-supply-chain-worm-cluster-2026
type: threat-intel
campaign: "TeamPCP / Shai-Hulud / CanisterWorm / Mini Shai-Hulud"
firstObserved: 2026-03-20
targetSectors:
  - Software development
  - Security tooling
  - SaaS
  - Cloud infrastructure
  - Developer tooling
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - PyPI
  - supply chain
  - worm
  - TeamPCP
  - Shai-Hulud
  - CanisterWorm
  - Bun
  - cluster analysis
hasArtifacts: false
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "npm, PyPI, Docker Hub, OpenVSX, GitHub Actions marketplace"
    c2_primary: "cluster — see C2 evolution section"
    c2_fallback: "tdtqy-oyaaa-aaaae-af2dq-cai.raw.icp0[.]io (ICP canister, TeamPCP/CanisterWorm phase)"
    exfil_channel: "private GitHub repos (Shai-Hulud phase); ICP canister (CanisterWorm phase)"
    c2_resilience_tier: 4
  capability:
    initial_access: oidc-token-theft
    execution: preinstall-hook
    evasion: bun-runtime-loader
    persistence: worm-propagation
  victim:
    direct: "Aqua Security, Checkmarx, Bitwarden, Xinference, litellm, SAP CAP maintainers, Lightning AI, Intercom, Namastex Labs"
    blast_radius: "~2M weekly downloads at peak (intercom-client); all CI/CD pipelines using affected GitHub Actions by version tag"
  evolution:
    - date: "2026-03-20"
      c2_tier: 3
      note: "TeamPCP — Cloudflare Tunnel + ICP canister fallback; public exfil repos"
    - date: "2026-04-22"
      c2_tier: 3
      note: "CanisterWorm — ICP canister as primary exfil; conventional HTTPS backup"
    - date: "2026-04-29"
      c2_tier: 4
      note: "Mini Shai-Hulud — private GitHub repos, OIDC token theft; public repos still used"
    - date: "2026-04-30"
      c2_tier: 4
      note: "Shai-Hulud — switched to private repos only within 24h of Mini Shai-Hulud; OPSEC upgraded mid-cycle"
sources:
  - title: "TeamPCP attack on KICS GitHub Action — Wiz"
    url: "https://www.wiz.io/blog/teampcp-attack-kics-github-action"
  - title: "Mini Shai-Hulud has appeared — Aikido"
    url: "https://www.aikido.dev/blog/mini-shai-hulud-has-appeared"
  - title: "Shai-Hulud worm pivots to multi-cloud: intercom-client hijacked — StepSecurity"
    url: "https://www.stepsecurity.io/blog/shai-hulud-worm-pivots-to-multi-cloud-intercom-client-hijacked"
  - title: "Namastex.ai npm Packages Hit with TeamPCP-Style CanisterWorm Malware — Socket"
    url: "https://socket.dev/blog/namastex-npm-packages-compromised-canisterworm"
  - title: "Datadog Security Research: supply-chain cluster tracking"
    url: "https://app.datadoghq.com/security/feed"
---

<p>Four campaign names have circulated in security research over the past six weeks: TeamPCP, Mini Shai-Hulud, Shai-Hulud, and CanisterWorm. Each was reported by a different team — Wiz, Aikido, StepSecurity, Socket — as a distinct incident. Each had its own IOC set, its own victim list, its own timeline. Researchers linked them by TTP overlap. What has not been written in one place is the single story they compose: one toolchain, iterating on its own infrastructure twice within six weeks, and upgrading its OPSEC once within a single 24-hour propagation cycle.</p>

<p>The linking artifact is the <code>__decodeScrambled</code> PBKDF2 cipher. It appears 232 times in the Shai-Hulud payload. It appears 232 times in the TeamPCP Bitwarden payload. It appears in CanisterWorm. It appears in Mini Shai-Hulud. That count and structure is not a coincidence of independent development. It is a shared build artifact — the same obfuscation library compiled into every payload across the cluster. What the individual incident reports each called a "link to TeamPCP" is, in aggregate, a single operator running one toolchain across six weeks of continuous operation.</p>

###### The cluster timeline

<ul role="list">
<li><strong>March 20, 2026 — TeamPCP initial compromise.</strong> <code>aquasecurity/trivy-action</code> and <code>aquasecurity/setup-trivy</code> compromised via tag force-update. Credential-stealing logic injected into <code>action.yaml</code>. Exfiltration to <code>scan.aquasecurtiy[.]org</code> (typosquat). C2 architecture: primary domain typosquat → Cloudflare Tunnel fallback → ICP canister fallback. C2 resilience tier 3.</li>
<li><strong>March 23–24, 2026 — TeamPCP expands.</strong> Checkmarx KICS GitHub Action compromised. <code>litellm</code> on PyPI (versions 1.82.7, 1.82.8) infected via credentials stolen from litellm's CI/CD pipeline running the compromised Trivy Action. The worm is propagating: each victim's credentials fuel the next compromise.</li>
<li><strong>April 22, 2026 — TeamPCP peak + CanisterWorm parallel track.</strong> Checkmarx KICS Docker images overwritten; Bitwarden CLI <code>@bitwarden/cli@2026.4.0</code> compromised; Xinference PyPI infected. Separately: CanisterWorm infects Namastex-linked npm packages. CanisterWorm adds the ICP canister (<code>cjn37-uyaaa-aaaac-qgnva-cai.raw.icp0[.]io</code>) as the <em>primary</em> exfil channel rather than a fallback — a distinct capability choice that signals either a separate operator reusing the toolchain or an evolution of CanisterWorm specifically for resilient exfil.</li>
<li><strong>April 29, 2026 — Mini Shai-Hulud.</strong> Four SAP CAP npm packages (<code>@cap-js/sqlite@2.2.2</code>, <code>@cap-js/postgres@2.2.2</code>, <code>@cap-js/db-service@2.10.1</code>, <code>mbt@1.2.48</code>) and the PyPI <code>lightning</code> package (versions 2.6.2, 2.6.3) infected. First use of GitHub Actions OIDC token theft as the propagation vector. Exfil via public GitHub repositories with the description <code>A Mini Shai-Hulud has Appeared</code> — a detectable public signal. C2 tier 4 (private GitHub API as exfil channel).</li>
<li><strong>April 30, 2026 — Shai-Hulud closes the loop.</strong> OIDC tokens stolen from the April 29 SAP/Lightning victims are used to publish <code>intercom-client@7.0.4</code> — roughly 2 million weekly downloads — within 24 hours. Critical change: exfil moves from <em>public</em> GitHub repositories to <em>private</em> repositories. The detectable signal from Mini Shai-Hulud (public repo creation) was removed within one propagation cycle. The operator iterated their own OPSEC in real time.</li>
</ul>

<blockquote>The OPSEC upgrade from public to private exfil repos happened within 24 hours of Mini Shai-Hulud. The operator observed that public repository creation was a detectable signal — it was being used by StepSecurity and others to identify compromises — and removed it before the next victim was published. That is an automated adversarial adaptation loop running faster than most incident response timelines.</blockquote>

###### The shared toolchain: what links them

<p>Three technical artifacts link the campaigns at the payload level. These are not TTP overlaps of the "both used postinstall hooks" variety. They are code-level fingerprints in the compiled output.</p>

<p><strong>1. <code>__decodeScrambled</code> PBKDF2 cipher (232 occurrences).</strong> The obfuscation layer used across the cluster applies a PBKDF2-based cipher with a consistent seed (<code>0x3039</code>). The function <code>__decodeScrambled</code> appears exactly 232 times in payloads from Shai-Hulud, TeamPCP Bitwarden phase, and Mini Shai-Hulud. This count is a build artifact — the number of obfuscated strings in the payload. Identical counts across independently discovered campaigns confirm a shared code base, not just shared technique.</p>

<p><strong>2. Bun v1.3.13 as the second-stage runtime.</strong> Every payload in the cluster downloads Bun v1.3.13 from GitHub releases during <code>npm install</code> and uses it to execute the obfuscated credential stealer. The version is pinned — not "latest Bun" but specifically v1.3.13. Pinning to a specific version is a build artifact that would differ across independently-developed payloads. The consistent version across campaigns confirms they were built from the same codebase at the same point in the Bun release history.</p>

<p><strong>3. Daemonization pattern (<code>__DAEMONIZED=1</code>).</strong> The payload forks a detached child process with the environment variable <code>__DAEMONIZED=1</code> to break process-tree correlation between <code>npm install</code> and the credential theft that follows. This specific variable name and pattern appears consistently across the cluster and is absent from unrelated npm supply-chain payloads.</p>

###### C2 architecture evolution

<p>The cluster's command and control architecture changed twice across six weeks. Each change increased resilience — the operator moved up the C2 tier hierarchy in response to defenders' ability to block or seize prior infrastructure.</p>

<table>
<thead><tr><th>Phase</th><th>Date</th><th>Primary C2</th><th>Fallback</th><th>Tier</th><th>Defender countermeasure</th></tr></thead>
<tbody>
<tr><td>TeamPCP</td><td>Mar 20</td><td>Domain typosquat (<code>scan.aquasecurtiy[.]org</code>)</td><td>Cloudflare Tunnel → ICP canister</td><td>3</td><td>Domain seizure possible on primary; Tunnel domain rotatable</td></tr>
<tr><td>CanisterWorm</td><td>Apr 22</td><td>ICP canister (<code>cjn37-uyaaa-aaaac-qgnva-cai</code>)</td><td>Conventional HTTPS (<code>telemetry.api-monitor[.]com</code>)</td><td>3</td><td>ICP canister cannot be seized; blocking requires egress policy on <code>raw.icp0.io</code></td></tr>
<tr><td>Mini Shai-Hulud</td><td>Apr 29</td><td>Public GitHub repos (victim's own account)</td><td>Commit dead-drop (<code>OhNoWhatsGoingOnWithGitHub</code>)</td><td>4</td><td>Public repo creation is detectable; dead-drop requires scanning public commits</td></tr>
<tr><td>Shai-Hulud</td><td>Apr 30</td><td>Private GitHub repos (victim's own account)</td><td>None needed</td><td>4</td><td>Traffic to <code>api.github.com</code> is allowlisted virtually everywhere; private repos invisible to external monitoring</td></tr>
</tbody>
</table>

<p>The progression is not random. Each tier removes a specific defensive countermeasure. The move to private GitHub repos as the exfil channel is particularly significant: all traffic goes to <code>api.github.com</code>, which appears in every corporate firewall's allowlist and every CI/CD egress policy. The credential exfiltration is indistinguishable from a legitimate CI job pushing artifacts to GitHub. There is no C2 domain to block, no suspicious IP to sinkhole, no non-standard port.</p>

###### The propagation architecture: how the worm works

<p>Understanding this cluster requires understanding that it is not a one-time compromise pattern. It is a self-propagating system where each victim becomes the next attack launchpad.</p>

<ol>
<li><strong>Initial compromise.</strong> The operator gains CI/CD access via one of three vectors across the cluster's history: GitHub Actions tag force-update (TeamPCP); maintainer account takeover via 2FA bypass (CanisterWorm); OIDC token theft from a compromised upstream pipeline (Shai-Hulud). Each method yields npm or PyPI publish credentials for the victim.</li>
<li><strong>Payload injection.</strong> The stolen credentials are used to publish a malicious patch version of every package the token can access. The payload is injected via a <code>preinstall</code> or <code>postinstall</code> hook. The version bump is minimal (patch release) to avoid triggering lockfile review. Package size increases substantially — 6 MB to 17.8 MB in the intercom-client case — which is a reliable detection signal.</li>
<li><strong>Credential sweep.</strong> When any developer or CI runner installs the malicious version, the payload executes and sweeps: npm tokens from <code>.npmrc</code>; GitHub OIDC tokens; AWS IMDS credentials; GCP metadata service tokens; Azure connection strings; SSH keys; Kubernetes service account tokens; secrets in environment variables.</li>
<li><strong>Worm propagation.</strong> Every stolen npm token is enumerated for packages it can publish. Each reachable package becomes the next infected host. The worm expands without operator intervention.</li>
</ol>

<p>This architecture means the blast radius of any single compromise scales with the publish-token scope of the victim's CI/CD environment. The SAP CAP maintainers had tokens that could publish to <code>@cap-js/*</code> packages. Those packages' consumers included CI pipelines with tokens that could publish to <code>intercom-client</code>. The worm found the path from a niche SAP toolchain package to a 2-million-download customer success SDK in 24 hours.</p>

###### What the cluster proves about automated OPSEC

<p>The transition from public to private exfil repos between Mini Shai-Hulud (April 29) and Shai-Hulud (April 30) is the single most analytically significant observation in this cluster. It is not a planned capability that was always present. The Mini Shai-Hulud payloads created public repos. By the time the worm reached intercom-client the following day, the payloads created private repos.</p>

<p>This means the operator either:</p>
<ul role="list">
<li>Manually updated the payload between April 29 and April 30 after observing that public repository creation was being used as a detection signal, or</li>
<li>Had already built both variants into the payload, with the private-repo variant triggered by the victim environment profile (intercom-client's environment was detected as higher-value, warranting quieter exfil)</li>
</ul>

<p>Either interpretation is operationally significant. In the first case: the operator monitors defensive research in near-real-time and iterates faster than incident response. In the second case: the payload has environment-aware OPSEC that adapts its behavior based on detected victim context. Both cases describe a capability that is qualitatively different from commodity malware.</p>

###### Detection across the cluster

<p>The shared toolchain means a single detection rule covers the entire cluster. Any one of the following signals catches all four campaigns:</p>

<ul role="list">
<li><strong>Bun download during <code>npm install</code>.</strong> Outbound egress to <code>github.com/oven-sh/bun/releases</code> from a CI runner that does not legitimately use Bun is a high-confidence indicator. This signal is present in every payload across the cluster. Flag version <code>1.3.13</code> specifically for highest confidence; flag any Bun download from an install step for broad coverage.</li>
<li><strong>Package size anomaly on patch release.</strong> A patch version bump that increases package size by more than 50% is anomalous. <code>intercom-client@7.0.4</code> grew from 6 MB to 17.8 MB. This signal does not require payload analysis — it is detectable from npm registry metadata before installation.</li>
<li><strong>SLSA provenance gap on a package with prior attestations.</strong> Every Shai-Hulud-phase payload was published manually, breaking the OIDC trusted-publisher chain that prior versions established. A package that shipped SLSA provenance in version N but not in version N+1 — especially on a patch bump — is a high-confidence indicator of account compromise or build pipeline injection.</li>
<li><strong>Private GitHub repository creation by CI service accounts.</strong> In the Shai-Hulud phase, exfil happens via private repo creation by whatever GitHub token the CI environment holds. A GitHub audit log entry showing repository creation by a service account immediately following an install step is a behavioral indicator.</li>
<li><strong>Detached child process with <code>__DAEMONIZED=1</code> surviving beyond <code>npm install</code> exit.</strong> The payload breaks process-tree correlation by daemonizing. Process telemetry that shows a child process surviving after its parent (<code>npm install</code>) exits, carrying <code>__DAEMONIZED=1</code> in its environment, is a conclusive payload indicator.</li>
<li><strong><code>__decodeScrambled</code> in any JavaScript payload.</strong> YARA/grep rule on this function name catches every variant in the cluster. Zero false positive risk — the name has no legitimate use.</li>
</ul>

###### Indicators of compromise

<p><strong>Compromised packages across the cluster (by phase):</strong></p>
<ul role="list">
<li>TeamPCP: <code>aquasecurity/trivy-action</code>, <code>aquasecurity/setup-trivy</code>, <code>checkmarx/kics-github-action</code>, <code>@bitwarden/cli@2026.4.0</code>, <code>xinference</code> PyPI 2.6.0–2.6.2, <code>litellm</code> PyPI 1.82.7–1.82.8</li>
<li>CanisterWorm: <code>@automagik/genie</code> 4.260421.33–4.260421.39, <code>pgserve</code> 1.1.11–1.1.13, <code>@fairwords/websocket</code>, <code>@fairwords/loopback-connector-es</code>, <code>@openwebconcept/design-tokens</code>, <code>@openwebconcept/theme-owc</code></li>
<li>Mini Shai-Hulud: <code>@cap-js/sqlite@2.2.2</code>, <code>@cap-js/postgres@2.2.2</code>, <code>@cap-js/db-service@2.10.1</code>, <code>mbt@1.2.48</code>, <code>lightning</code> PyPI 2.6.2–2.6.3</li>
<li>Shai-Hulud: <code>intercom-client@7.0.4</code></li>
</ul>

<p><strong>Toolchain fingerprints (present in all phases):</strong></p>
<ul role="list">
<li><code>__decodeScrambled</code> function name (232 occurrences per payload)</li>
<li>Bun v1.3.13 download from <code>github.com/oven-sh/bun/releases</code> during install</li>
<li><code>__DAEMONIZED=1</code> environment variable on forked child process</li>
<li><code>setup.mjs</code> + <code>router_runtime.js</code> or <code>execution.js</code> payload file names</li>
</ul>

<p><strong>Network indicators (defanged, by phase):</strong></p>
<ul role="list">
<li>TeamPCP: <code>scan.aquasecurtiy[.]org</code>, <code>checkmarx[.]zone</code>, <code>audit.checkmarx[.]cx</code>, <code>championships-peoples-point-cassette.trycloudflare[.]com</code></li>
<li>CanisterWorm: <code>telemetry.api-monitor[.]com</code>, <code>cjn37-uyaaa-aaaac-qgnva-cai.raw.icp0[.]io</code></li>
<li>Mini Shai-Hulud: public GitHub repos with description <code>A Mini Shai-Hulud has Appeared</code></li>
<li>Shai-Hulud: <code>api.github[.]com</code> (private repo creation — indistinguishable from legitimate traffic by URL alone; detect by behavior)</li>
</ul>

<p><strong>Behavioral artifacts:</strong></p>
<ul role="list">
<li>Outbound connection to <code>169.254.169[.]254</code> (AWS IMDS) from a process spawned by npm install</li>
<li>Outbound connection to <code>metadata.google[.]internal</code> from same context</li>
<li>GitHub commit messages containing <code>OhNoWhatsGoingOnWithGitHub</code> (Mini Shai-Hulud dead-drop)</li>
<li>GitHub commits authored by <code>claude &lt;claude@users.noreply.github.com&gt;</code> touching <code>.vscode/</code> or <code>.claude/</code> paths (Mini Shai-Hulud persistence)</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Enable <code>npm install --ignore-scripts</code> by default in CI.</strong> The entire preinstall/postinstall execution chain across this cluster is blocked by this flag. This is the structural fix that applies to every campaign variant, present and future. Explicitly allowlist packages that require lifecycle hooks; treat every other hook as suspicious.</li>
<li><strong>Pin GitHub Actions by SHA, not by version tag.</strong> The TeamPCP initial compromise relied on tag force-update. Pinning to a commit SHA means the tag can be moved without affecting your workflow. This is the structural fix for the GitHub Actions vector.</li>
<li><strong>Gate on SLSA provenance attestations.</strong> Any npm package that drops provenance attestations in a patch release should trigger review before installation. Automate this check in CI. The Shai-Hulud compromise was detectable at this layer before payload analysis.</li>
<li><strong>Monitor for Bun egress from CI runners.</strong> Add a network egress alert for outbound connections to <code>github.com/oven-sh/bun/releases</code> from CI runners that do not legitimately use Bun. This single rule covers the entire cluster.</li>
<li><strong>Enforce IMDSv2 with hop limit 1 on all EC2 instances and containers.</strong> The Shai-Hulud payload queries the AWS Instance Metadata Service. IMDSv2 with a hop limit of 1 prevents container-originating queries from reaching the IMDS, blocking cloud credential theft at Stage 3.</li>
<li><strong>Audit GitHub audit logs for private repository creation by CI service accounts.</strong> In the Shai-Hulud phase, the exfil signal moved to private repos. The GitHub audit log still records repository creation events. Any CI service account creating a private repository immediately following an install step is a high-confidence indicator.</li>
<li><strong>Treat any host that ran an affected version as fully compromised.</strong> Rotate all credentials reachable from the environment: GitHub tokens, npm publish tokens, AWS/GCP/Azure credentials, Kubernetes service account tokens, SSH keys, and secrets in environment variables.</li>
</ul>

###### Attribution discipline

<p>"TeamPCP," "Shai-Hulud," "Mini Shai-Hulud," and "CanisterWorm" are tracking names assigned by Wiz, StepSecurity, Aikido, and Socket respectively. The <code>__decodeScrambled</code> fingerprint, the Bun v1.3.13 pin, and the daemonization pattern link them at the toolchain level. This is sufficient to conclude that the campaigns share a codebase. It is not sufficient to conclude they share an operator.</p>

<p>Three scenarios are consistent with the available evidence:</p>
<ul role="list">
<li>A single operator running the entire cluster, iterating the toolchain across six weeks</li>
<li>A primary operator (TeamPCP) whose toolchain was reused by one or more additional actors (CanisterWorm, Shai-Hulud) who independently obtained or forked the code</li>
<li>A toolchain sold or shared privately within a small group, with each actor adapting the C2 infrastructure independently</li>
</ul>

<p>The OPSEC iteration observed between Mini Shai-Hulud and Shai-Hulud — public to private exfil repos within 24 hours — is the strongest evidence for the single-operator scenario. A forked codebase used by an independent actor would not have iterated in real time in response to a detection signal from a different variant. But this inference is not conclusive: a coordinated group operating the same toolchain could also produce this pattern.</p>

<p>We use the individual campaign names throughout hub articles for traceability against existing reporting. We treat the toolchain overlap as a linking indicator, not a hard attribution. Defenders can act on the shared IOCs and detection rules without needing the attribution question settled.</p>

###### Criminal-market signal

<p>We ran comprehensive dark-web sweeps across 19+ live onion search engines for every campaign in this cluster. Results are consistent across all phases:</p>

<ul role="list">
<li><strong>TeamPCP</strong> — clean negative across 18 pages (2 exploit forums, 15 marketplaces, 1 other)</li>
<li><strong>Mini Shai-Hulud</strong> — clean negative across 19 pages (2 exploit forums, 16 marketplaces, 1 other)</li>
<li><strong>Shai-Hulud</strong> — clean negative across 76 pages (13 exploit forums, 59 marketplaces, 3 news mirrors)</li>
<li><strong><code>__decodeScrambled</code></strong> — pending sweep (Tier 1 artifact, zero homonym risk)</li>
<li><strong><code>OhNoWhatsGoingOnWithGitHub</code></strong> — pending sweep (Tier 2 self-attribution marker)</li>
</ul>

<p>Three clean negatives across the same cluster is a pattern, not a sensor failure. The interpretation is H2 (genuine absence): this operator uses the toolchain in-house. They are not selling the worm capability, not selling the stolen credentials through observable dark-web markets, and not commoditizing the attack. The absence is consistent with an operator whose business model is the credentials themselves — not the sale of the attack capability to others.</p>

<p>This shapes defensive monitoring posture. There is no early warning available from criminal-market surveillance for this campaign class. The detection surface is the registry layer: package metadata anomalies, SLSA provenance gaps, Bun egress from CI runners. Not .onion forums.</p>

###### What six weeks of continuous operation means

<p>This cluster has been running since March 20, 2026. As of this writing it has compromised at least ten distinct software vendors, infected packages across npm and PyPI with a combined download count exceeding 25 million weekly at peak, and stolen credentials from CI/CD pipelines across the software development supply chain. It has iterated its C2 architecture twice and its OPSEC at least once.</p>

<p>The operational longevity — six weeks of continuous activity with no apparent disruption — reflects the structural advantages of the campaign's design. The worm propagates without sustained operator effort. Each new victim extends the reach automatically. The C2 infrastructure, once moved to private GitHub repos, produces traffic that is functionally invisible to network perimeter controls. The only layer where this campaign is reliably detectable is the one most organizations have invested least in: npm and PyPI publish audit logs, package provenance attestation, and CI install script monitoring.</p>

<p>The structural defenses have not changed since the first TeamPCP compromise in March. <code>--ignore-scripts</code> by default. GitHub Actions pinned by SHA. SLSA provenance as a CI gate. IMDSv2 enforced. None of these are new recommendations. The campaign has persisted for six weeks because the recommendations are not widely implemented, not because defenders lack the tools to stop it.</p>
