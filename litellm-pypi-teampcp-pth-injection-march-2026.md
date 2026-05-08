---
title: "litellm on PyPI: the first .pth injection in the TeamPCP chain and why the operator named it"
date: 2026-05-07
heroImage: "/art/litellm-pypi-teampcp-pth-injection-march-2026.png"
description: "The litellm PyPI compromise on March 24, 2026 was the first documented use of .pth file injection in the TeamPCP campaign. The operator later named the technique after this compromise in CanisterWorm's own payload string — Layer 1 evidence that litellm is where this class of persistence began in the cluster."
slug: litellm-pypi-teampcp-pth-injection-march-2026
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-03-24
targetSectors:
  - AI / ML
  - Software development
  - Cloud infrastructure
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
  - .pth injection
  - AI tooling
  - litellm
hasArtifacts: false
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "PyPI litellm package (compromised via Trivy GitHub Action credential theft)"
    c2_primary: "models.litellm[.]cloud (typosquatting litellm.ai)"
    c2_fallback: null
    exfil_channel: "AES-256 + RSA-4096 encrypted payload exfiltrated to typosquat domain"
    c2_resilience_tier: 1
  capability:
    initial_access: ci-cd-injection
    execution: pth-injection
    evasion: double-base64-encoded-payload
    persistence: pth-site-packages
  victim:
    direct: "BerriAI / litellm maintainers — CI/CD pipeline running compromised Trivy GitHub Action"
    blast_radius: "litellm 1.82.7 and 1.82.8 — widely-used LLM proxy, 100+ provider integrations; all Python environments with affected versions installed"
sources:
  - title: "litellm compromise report — BerriAI GitHub issue #24512"
    url: "https://github.com/BerriAI/litellm/issues/24512"
  - title: "Datadog Security Research: litellm PyPI compromise card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "TeamPCP campaign tracking — this hub"
    url: "https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking"
  - title: "CanisterWorm analysis — this hub"
    url: "https://keepsecure.io/hub/canisterworm-npm-icp-c2-supply-chain-2026"
---

<p>On March 24, 2026, a community member reported that <code>litellm</code> — a Python package used as a unified LLM proxy across over 100 provider APIs — had been compromised on PyPI. Malicious versions 1.82.7 and 1.82.8 were removed from the registry after the report. The compromise was the first documented use of <code>.pth</code> file injection in the <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">TeamPCP campaign cluster</a>.</p>

<p>The operator subsequently named the technique after this compromise. Six weeks later, CanisterWorm's payload contained the self-attribution string <code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code>. That string — embedded in the operator's own malware — is Layer 1 evidence: the operator considered the litellm compromise to be the technique's origin point in their toolchain. This finding documents that origin.</p>

###### What we know

<ul role="list">
<li><strong>March 24, 2026</strong> — Community report filed at BerriAI/litellm GitHub issue #24512. Malicious code confirmed in <code>litellm==1.82.7</code> and <code>litellm==1.82.8</code>. Both versions subsequently removed from PyPI.</li>
<li><strong>Vector</strong> — The compromise entered litellm's CI/CD pipeline via a Trivy GitHub Action that had been compromised by TeamPCP on March 20. The Action stole PyPI publishing credentials from litellm's pipeline. Those credentials were used to publish the malicious versions.</li>
<li><strong>Two delivery variants across the two versions:</strong>
  <ul>
    <li>Version 1.82.8: malicious code delivered as <code>litellm_init.pth</code> in the package distribution. Python's <code>site.py</code> evaluates <code>.pth</code> files at interpreter startup before any import. The payload executed on every <code>python</code> invocation in environments where the package was installed — no explicit import required.</li>
    <li>Version 1.82.7: malicious payload embedded directly in <code>proxy/proxy_server.py</code>. Requires the compromised module to be imported, a narrower trigger condition than the <code>.pth</code> variant.</li>
  </ul>
</li>
</ul>

<blockquote>The <code>.pth</code> variant in 1.82.8 is the operationally significant one. A <code>.pth</code> file in <code>site-packages</code> is not an import — it is a startup hook. It runs before the Python application has loaded. It survives package reinstallation if the <code>.pth</code> file itself is not removed. It runs in every subprocess that inherits the Python environment. The operator recognized this persistence advantage: by CanisterWorm's release, <code>.pth</code> injection had become the preferred PyPI persistence mechanism in the toolchain, named after this compromise.</blockquote>

###### Why litellm was targeted

<p>litellm's attack surface profile explains why it appeared in the TeamPCP propagation chain. It is used primarily in AI engineering workflows — the same environments that hold the credentials TeamPCP systematically harvests:</p>

<ul role="list">
<li><strong>Cloud provider credentials at scale.</strong> litellm is a proxy that routes requests to AWS Bedrock, GCP Vertex AI, Azure OpenAI, and 100+ other providers. Environments running litellm typically have cloud credentials with broad scope — the exact credential class TeamPCP's sweeper targets.</li>
<li><strong>CI/CD pipeline deployment.</strong> Production litellm deployments frequently run in CI/CD environments for evaluation pipelines and automated testing. Those environments hold the combination of cloud credentials and package-publish tokens that enable worm propagation.</li>
<li><strong>High download volume.</strong> A widely-installed package maximises the blast radius per compromised publish token. litellm's reach into the AI/ML development community made it a high-value propagation target.</li>
</ul>

###### Payload mechanics

<p>The following analysis is from the community disclosure at BerriAI/litellm#24512, verified against the primary source.</p>

<p>The <code>litellm_init.pth</code> file (34,628 bytes, SHA-256: <code>ceNa7wMJnNHy1kRnNCcwJaFjWX3pORLfMh7xGL8TUjg</code>) contained a single Python statement that spawned a subprocess running a double-base64-decoded payload. The trigger reads:</p>

<pre><code>import os, subprocess, sys; subprocess.Popen([sys.executable, "-c",
"import base64; exec(base64.b64decode('...'))"])</code></pre>

<p>The payload collected a specific credential surface from the victim environment, including: all environment variables; SSH private keys; AWS credentials and IMDS token; Kubernetes config files and service account tokens; GCP application default credentials; Azure credentials; Docker authentication configs; npm tokens (<code>.npmrc</code>); Vault tokens; shell histories; cryptocurrency wallet directories; SSL/TLS private keys; CI/CD configuration files (<code>.gitlab-ci.yml</code>, <code>Jenkinsfile</code>, <code>.travis.yml</code>); and database credential files.</p>

<p>Collected data was written to a temporary file, encrypted with AES-256-CBC using PBKDF2 key derivation via <code>openssl enc</code>, and the AES session key was separately encrypted with a hardcoded 4096-bit RSA public key using OAEP padding. Both files were packed into an archive named <strong><code>tpcp.tar.gz</code></strong> — a direct self-attribution: the archive carries the campaign name. The archive was exfiltrated via HTTP POST to <code>https://models.litellm[.]cloud/</code> with <code>Content-Type: application/octet-stream</code>.</p>

<p>The typosquat domain <code>litellm[.]cloud</code> (versus the legitimate <code>litellm.ai</code>) evades casual egress log review by appearing plausible, and the campaign name embedded in the archive name is the second self-attribution marker in the TeamPCP cluster — the first being <code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code> in CanisterWorm's payload six weeks later.</p>

###### The .pth technique origin: four-step analysis

<p>The claim that litellm is the first use of <code>.pth</code> injection in the TeamPCP chain rests on four steps:</p>

<p><strong>Observation:</strong> The CanisterWorm payload contains the string <code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code>. The March 24 litellm compromise is the earliest documented use of <code>.pth</code> injection in the TeamPCP cluster timeline.</p>

<p><strong>Mechanism:</strong> Self-attribution strings in malware are deliberate operator choices, not build artifacts. An operator who names a technique after a specific compromise is recording its origin for internal reference. The string references "LiteLLM" specifically — not "PyPI injection" or a generic description — indicating the operator associates the technique with this particular victim, not the technique class broadly.</p>

<p><strong>Inference:</strong> litellm was the originating application of <code>.pth</code> injection in the TeamPCP toolchain. The technique was subsequently standardized and reused: in elementary-data (April 30, <code>elementary.pth</code>), in CanisterWorm's PyPI propagation logic, and in TeamPCP's broader toolkit.</p>

<p><strong>Confidence bound:</strong> This inference would be weakened by evidence of <code>.pth</code> injection in TeamPCP activity before March 24, 2026. No such evidence is known. The three-week gap between the litellm compromise (March 24) and the next confirmed <code>.pth</code> use in the cluster (Xinference, April 22) is consistent with a technique being refined between deployments.</p>

###### Indicators of compromise

<p><strong>Compromised versions:</strong></p>
<ul role="list">
<li><code>litellm==1.82.7</code> — payload in <code>proxy/proxy_server.py</code></li>
<li><code>litellm==1.82.8</code> — payload as <code>litellm_init.pth</code> in <code>site-packages</code></li>
</ul>

<p><strong>Filesystem artifacts:</strong></p>
<ul role="list">
<li><code>litellm_init.pth</code> in any Python <code>site-packages</code> directory — SHA-256: <code>ceNa7wMJnNHy1kRnNCcwJaFjWX3pORLfMh7xGL8TUjg</code></li>
<li><code>tpcp.tar.gz</code> in temp directories — the exfil archive; presence confirms active data collection</li>
</ul>

<p><strong>Network indicator (defanged):</strong></p>
<ul role="list">
<li><code>https://models.litellm[.]cloud/</code> — exfiltration endpoint (typosquatting <code>litellm.ai</code>); POST with <code>Content-Type: application/octet-stream</code></li>
</ul>

<p><strong>Campaign self-attribution markers (Layer 1):</strong></p>
<ul role="list">
<li><code>tpcp.tar.gz</code> — archive name embedding the campaign identifier</li>
<li><code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code> — in CanisterWorm payload six weeks later, naming this compromise as the technique origin</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Hunt for <code>litellm_init.pth</code> immediately.</strong> Run <code>find $(python3 -c "import site; print(' '.join(site.getsitepackages()))") -name "litellm_init.pth" 2>/dev/null</code> on all Python environments. If present, the 1.82.8 payload executed and is persisting across all Python invocations in that environment.</li>
<li><strong>Audit for any unexpected <code>.pth</code> files in <code>site-packages</code>.</strong> Any <code>.pth</code> file containing executable code rather than plain path entries is malicious. This detection covers the entire TeamPCP <code>.pth</code> injection class, not just litellm.</li>
<li><strong>Rotate all credentials reachable from affected environments.</strong> The collection scope was broad: cloud credentials, SSH keys, CI/CD secrets, database passwords, API keys. Treat any environment with 1.82.7 or 1.82.8 installed as fully compromised.</li>
<li><strong>Check for worm propagation from affected environments.</strong> If the environment held PyPI or npm publish tokens, check both registries for unexpected patch releases on packages those tokens could access. The TeamPCP worm uses stolen publish credentials to extend its reach downstream.</li>
<li><strong>Block <code>models.litellm[.]cloud</code> egress.</strong> This domain has no legitimate use. Any outbound connection to it is evidence of active exfiltration.</li>
<li><strong>Pin your Trivy GitHub Actions by SHA.</strong> The confirmed entry vector for this compromise was a Trivy Action pinned by version tag rather than commit SHA. SHA pinning means the tag can be moved without affecting your workflow. This is the structural fix for the entire GitHub Actions tag-hijack vector.</li>
</ul>

###### Attribution and cluster position

<p>The litellm compromise is the third confirmed event in the TeamPCP campaign timeline, following the Trivy Action compromise (March 20) and the Aqua Security GitHub org defacement (March 23). It is the first confirmed PyPI victim and the first confirmed use of <code>.pth</code> injection in the cluster.</p>

<p>The progression matters for defenders: TeamPCP began with a GitHub Actions compromise (npm-focused, affecting the Trivy ecosystem), then used stolen credentials to pivot into PyPI distribution (litellm), establishing a cross-ecosystem worm that expanded over the following six weeks to Checkmarx, Bitwarden, Xinference, and the Shai-Hulud npm cluster. The litellm compromise is where TeamPCP's PyPI branch began.</p>

<p>See the <a href="https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026">npm supply-chain worm cluster analysis</a> for the full six-week timeline and cross-campaign technical analysis.</p>

###### Criminal-market signal

<p>Dark-web sweeps for TeamPCP across multiple runs have returned clean negatives. The litellm compromise is part of the same operator-run toolchain. No criminal-market presence for this technique or this campaign is expected or has been observed. The operator uses the credentials; they do not sell the attack. Detection must occur at the registry and CI/CD layer — specifically, hunting for unexpected <code>.pth</code> files in Python environments and monitoring for Trivy Action SHA drift.</p>
