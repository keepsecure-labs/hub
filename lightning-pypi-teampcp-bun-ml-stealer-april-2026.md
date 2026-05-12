---
title: "lightning on PyPI: TeamPCP expands into Python ML at scale with Bun runtime and GitHub dead-drop exfiltration"
date: 2026-05-12
heroImage: "/art/lightning-pypi-teampcp-bun-ml-stealer-april-2026.png"
description: "The lightning Python package on PyPI was compromised on April 30, 2026, in a campaign where TeamPCP claimed responsibility via a Tor-linked GitHub issue. With hundreds of thousands of daily downloads from Python machine learning environments, this is the highest-volume PyPI victim in the TeamPCP cluster to date. The exfiltration mechanism — AES-encoded commits to attacker-controlled GitHub repositories — cannot be blocked at the network layer by standard domain blocklists."
slug: lightning-pypi-teampcp-bun-ml-stealer-april-2026
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-04-30
targetSectors:
  - AI / ML
  - Python machine learning
  - Software development
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - PyPI
  - supply chain
  - TeamPCP
  - Bun runtime
  - AI tooling
  - lightning
  - GitHub dead-drop
hasArtifacts: false
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "PyPI lightning package (compromised via CI/CD credential theft or account takeover)"
    c2_primary: "Attacker-controlled GitHub repositories (dead-drop exfil via encoded commits)"
    c2_fallback: null
    exfil_channel: "AES-encoded data committed to attacker-controlled GitHub repos; downstream npm tarball tampering via stolen publish tokens"
    c2_resilience_tier: 4
  capability:
    initial_access: import-time-execution
    execution: bun-runtime-loader
    evasion: obfuscated-11MB-bundle
    persistence: null
  victim:
    direct: "Python ML engineers, AI pipeline operators, CI/CD environments running lightning"
    blast_radius: "lightning 2.6.2 and 2.6.3 — hundreds of thousands of daily downloads; any Python ML environment with affected versions installed"
sources:
  - title: "Socket: Lightning PyPI package compromised by Bun-based credential stealer"
    url: "https://socket.dev/blog/lightning-pypi-package-compromised"
  - title: "Datadog Security Research: Lightning PyPI compromise card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "TeamPCP campaign tracking — this hub"
    url: "https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking"
  - title: "npm supply-chain worm cluster analysis — this hub"
    url: "https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026"
---

<p>The <code>lightning</code> Python package on PyPI — a widely-used distributed training framework in the Python machine learning ecosystem — was compromised on April 30, 2026. Malicious versions 2.6.2 and 2.6.3 were identified and removed from the registry. A group calling itself Team PCP claimed responsibility via a Tor onion site linked from a GitHub issue on the project.</p>

<p>Lightning receives several hundred thousand downloads per day from Python ML environments, making this the highest-volume PyPI target in the TeamPCP cluster documented to date. The attack follows the TeamPCP pattern established in litellm (March 24) and elementary-data (April 30): compromise a CI/CD pipeline or package account to publish malicious versions, execute on import, harvest credentials using a Bun-runtime loader, and exfiltrate. The distinguishing feature in this campaign is the exfiltration channel: rather than posting to a dedicated C2 domain, the malware commits AES-encoded stolen data to attacker-controlled GitHub repositories. This makes the exfiltration traffic indistinguishable from legitimate GitHub API calls — standard network egress monitoring cannot block it without blocking GitHub entirely.</p>

<p>The Bun runtime loader pattern and tooling overlap with Shai-Hulud and Mini Shai-Hulud were noted by Socket in their primary analysis. Primary-source verification of the specific Bun version pin is pending (Task #15); if confirmed as v1.3.13, this becomes a high-confidence linking artifact with the existing cluster.</p>

###### What we know

<ul role="list">
<li><strong>April 30, 2026</strong> — Socket published primary analysis of the compromise. Malicious versions 2.6.2 and 2.6.3 identified and removed from PyPI.</li>
<li><strong>Clean version:</strong> 2.6.1, published January 30, 2026, is reported clean by Socket.</li>
<li><strong>Self-attribution:</strong> TeamPCP claimed responsibility via a Tor onion site linked from a GitHub issue on the lightning project repository. This is Layer 1 self-attribution — the operator named themselves at the point of disclosure.</li>
<li><strong>Download volume:</strong> Several hundred thousand downloads per day from Python ML environments. Lightning is used across AI/ML training pipelines, academic computing clusters, and production ML infrastructure.</li>
</ul>

###### Why lightning was targeted

<p>The lightning attack surface profile is the broadest in the TeamPCP PyPI branch. Lightning is not a narrow LLM proxy (litellm) or a distributed analytics tool (elementary-data) — it is a foundational distributed ML training framework used across the full spectrum of Python ML development:</p>

<ul role="list">
<li><strong>Cloud training credentials at maximum scope.</strong> Lightning deployments run on AWS, GCP, and Azure compute with credentials scoped for GPU cluster management, storage access, and model registry write. These credentials have broader scope than typical production service credentials.</li>
<li><strong>Research and production in the same codebase.</strong> Lightning spans academic research environments (where security controls are weaker) and production ML infrastructure (where the credential blast radius is highest). A single compromised PyPI installation reaches both.</li>
<li><strong>CI/CD integration for automated training runs.</strong> Automated retraining pipelines running on CI/CD hold the combination of cloud credentials and PyPI publish tokens that the TeamPCP worm exploits for propagation. Lightning's prevalence in automated ML pipelines puts it squarely in the propagation surface TeamPCP has been systematically targeting.</li>
<li><strong>Scale as the selection criterion.</strong> Socket reports hundreds of thousands of downloads per day. At that scale, even a brief window of malicious versions in the registry produces thousands of compromised environments before the package is yanked.</li>
</ul>

###### Payload mechanics

<p>The following analysis is from Socket's primary report, as characterized by Datadog Security Research. Primary-source payload artifact verification is pending (Task #15).</p>

<p>The malicious code is bootstrapped by a <code>start.py</code> module that runs when the package is imported — not at install time. This is a different trigger point than the litellm <code>.pth</code> file (which executed at Python startup) and the elementary-data <code>postinstall</code> hook (which executed during <code>npm install</code>). An import-triggered payload means it does not execute in environments that install the package but never import it — a narrower trigger than <code>.pth</code> injection but still broad given lightning's usage patterns.</p>

<p>The bootstrapper detects host OS and architecture, downloads the Bun JavaScript runtime from GitHub, and launches a hidden <code>_runtime/router_runtime.js</code> payload as a daemon thread with suppressed output. The Bun runtime is downloaded from GitHub at runtime rather than bundled in the package — this keeps the malicious package size normal and means static package analysis tools scanning the tarball for suspicious content will not find the payload directly.</p>

<p>The second stage is an 11 MB obfuscated JavaScript bundle. Socket describes hundreds of references to environment variables and authentication tokens within the payload. Collection targets include:</p>

<ul role="list">
<li>GitHub personal access tokens and GitHub Actions tokens</li>
<li>npm publish tokens from <code>.npmrc</code></li>
<li>Cloud provider credentials from environment variables and on-disk configuration files</li>
<li>Other secrets accessible from the developer or CI environment</li>
</ul>

<p>Socket reports two uses of stolen tokens after collection. First, the malware commits AES-encoded exfiltrated data to attacker-controlled repositories. Second, it tampers with downstream npm package tarballs in repositories to which the victim holds push access — the propagation mechanism.</p>

<blockquote>The GitHub dead-drop exfil channel is the operational innovation in this campaign. Exfiltrating data via commits to attacker-controlled GitHub repositories means the outbound traffic is GitHub API calls — HTTPS to <code>api.github.com</code>. Any organization that has not implemented content-level egress inspection cannot distinguish this from a developer pushing code. The standard detection action for supply-chain C2 (block the C2 domain) does not apply. Detection must occur either at the package install/import stage or via behavioral analysis of GitHub API call patterns from unexpected processes.</blockquote>

###### Cluster position and Bun linking artifact

<p>Lightning is the fourth confirmed PyPI victim in the TeamPCP cluster, following litellm (March 24), Xinference (April 22), and elementary-data (April 30). The Bun-runtime loader pattern is consistent across all PyPI victims — TeamPCP standardized on Bun after establishing the technique in the npm cluster (Shai-Hulud, Mini Shai-Hulud).</p>

<p>Socket notes tooling overlap with Shai-Hulud and Mini Shai-Hulud. In the existing cluster analysis, the Bun v1.3.13 pin is a high-confidence linking artifact: pinning to a specific non-LTS Bun point release is a deliberate build choice that independent codebases would not make identically. If Socket's primary analysis confirms v1.3.13 in the lightning payload, this is the same proof of shared codebase that links Shai-Hulud and Mini Shai-Hulud. That verification is the open task (Task #15).</p>

###### Indicators of compromise

<p><strong>Compromised versions:</strong></p>
<ul role="list">
<li><code>lightning==2.6.2</code> — malicious; payload in bootstrapper loaded at import</li>
<li><code>lightning==2.6.3</code> — malicious; same payload chain</li>
<li><code>lightning==2.6.1</code> — clean (reported by Socket)</li>
</ul>

<p><strong>Filesystem artifacts:</strong></p>
<ul role="list">
<li><code>_runtime/router_runtime.js</code> within the lightning package directory — 11 MB obfuscated bundle; presence of this file in site-packages confirms malicious version was installed</li>
<li>Bun binary downloaded to a temporary or hidden path at import time — anomalous binary appearing in temp directories during Python process execution</li>
</ul>

<p><strong>Network indicators:</strong></p>
<ul role="list">
<li>Outbound connections to GitHub API for commit operations from within a Python process during model training or package import — not normal behavior</li>
<li>Bun runtime download from GitHub during Python package import — a Python ML package has no legitimate reason to download the Bun JavaScript runtime</li>
</ul>

<p><strong>Campaign self-attribution (Layer 1):</strong></p>
<ul role="list">
<li>Tor onion site linked from lightning GitHub issue — TeamPCP claiming responsibility at disclosure</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Upgrade immediately.</strong> Any environment with <code>lightning==2.6.2</code> or <code>2.6.3</code> installed should be treated as compromised. Pin to a version after 2.6.3 once Socket and the lightning maintainers confirm a clean release.</li>
<li><strong>Audit for <code>_runtime/router_runtime.js</code>.</strong> Run <code>find $(python3 -c "import site; print(' '.join(site.getsitepackages()))") -name "router_runtime.js" 2>/dev/null</code>. Presence confirms malicious version was installed and the payload loaded.</li>
<li><strong>Rotate all credentials from affected environments.</strong> The collection scope includes GitHub tokens, npm tokens, and cloud credentials. Any environment that imported the malicious version should be treated as fully compromised — all accessible secrets rotated.</li>
<li><strong>Check for worm propagation via npm publish tokens.</strong> If the environment held npm publish tokens, check for unexpected patch releases on any packages those tokens could access. TeamPCP uses stolen publish credentials to extend the worm downstream.</li>
<li><strong>Audit GitHub API call patterns from ML pipeline processes.</strong> Commit operations originating from a Python ML training process are not expected behavior. If your SIEM ingests GitHub audit logs, alert on commits from CI systems running lightning during the April 30 window.</li>
<li><strong>Detect Bun download in Python contexts.</strong> Any process with a Python parent downloading Bun binary artifacts from GitHub is anomalous. This behavioral rule covers the TeamPCP Bun-runtime loader pattern across all PyPI victims, not just lightning.</li>
</ul>

###### Attribution and cluster position

<p>TeamPCP's self-attribution via Tor-linked GitHub issue on April 30 is consistent with the campaign's pattern of claiming victims at disclosure. The claimed LAPSUS$ connection noted in Socket's analysis remains unverified and is not assessed to change the threat model for defenders.</p>

<p>See the <a href="https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026">npm supply-chain worm cluster analysis</a> for the full cluster timeline and the technical analysis connecting TeamPCP, Shai-Hulud, Mini Shai-Hulud, and CanisterWorm through the Bun version pin and <code>__decodeScrambled</code> build artifact.</p>

###### Criminal-market signal

<p>TeamPCP sweeps across multiple runs have returned clean negatives. The operator-run credential-collection model documented across the cluster applies to lightning. The GitHub dead-drop exfil is consistent with the operator using credentials directly rather than selling access — a broker or commodity actor would use a simpler, more operationally detached exfil channel. No dark-web presence for lightning compromise tooling is expected or has been observed (H2 operator-run pattern).</p>
