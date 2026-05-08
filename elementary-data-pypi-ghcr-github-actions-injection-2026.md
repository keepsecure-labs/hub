---
title: "How a GitHub Actions comment box became a supply-chain attack: elementary-data on PyPI and GHCR"
date: 2026-04-30
heroImage: "/art/elementary-data-pypi-ghcr-github-actions-injection-2026.png"
description: "An attacker injected commands into elementary-data's CI pipeline through a GitHub Actions script-injection flaw, forged a release tag, and delivered a malicious .pth file to both PyPI and GitHub Container Registry."
slug: elementary-data-pypi-ghcr-github-actions-injection-2026
type: threat-intel
campaign: "GitHub Actions Script Injection"
firstObserved: 2026-04-30
targetSectors:
  - Data engineering
  - Analytics platforms
  - Cloud infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - PyPI
  - GHCR
  - GitHub Actions
  - script injection
  - supply chain
  - .pth injection
hasArtifacts: false
sources:
  - title: "elementary-data Compromised on PyPI and GHCR: Forged Release Pushed via GitHub Actions Script Injection — StepSecurity"
    url: "https://www.stepsecurity.io/blog/elementary-data-compromised-on-pypi-and-ghcr-forged-release-pushed-via-github-actions-script-injection"
  - title: "Datadog Security Research: elementary-data PyPI and GHCR releases compromised"
    url: "https://app.datadoghq.com/security/feed"
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "PyPI elementary-data package; GHCR ghcr.io/elementary-data/elementary"
    c2_primary: "igotnofriendsonlineorirl-imgonnakmslmao[.]skyhanni[.]cloud"
    c2_fallback: null
    exfil_channel: "trin.tar.gz archive staged at $TMPDIR/.trinny-security-update"
    c2_resilience_tier: 1
  capability:
    initial_access: ci-cd-injection
    execution: pth-injection
    evasion: multi-stage-decode
    persistence: pth-site-packages
  victim:
    direct: "elementary-data project (data engineering/analytics PyPI package)"
    blast_radius: "elementary-data==0.23.3 and ghcr.io/elementary-data/elementary:0.23.3 / :latest — April 30 2026 window"
---

<p>A single unguarded interpolation in a GitHub Actions workflow is the difference between a routine release pipeline and a supply-chain attack that publishes to two registries simultaneously. The compromise of <code>elementary-data==0.23.3</code>, first reported by StepSecurity on April 30, 2026, demonstrates the class precisely: an attacker injected commands into a CI workflow by exploiting a pull-request comment field that was interpolated unsanitized into a shell <code>run:</code> block. The injected code ran inside the workflow context with a <code>GITHUB_TOKEN</code> in scope, which the attacker used to forge a release commit, push a tag, and dispatch the project's own publishing workflow against that tag. The result was a trojaned <code>elementary-data</code> package on PyPI and a multi-arch container image on GitHub Container Registry — both signed artifacts, both from the project's legitimate publishing infrastructure.</p>

###### What we know

<ul role="list">
<li><strong>April 30, 2026</strong> — StepSecurity reports the compromise of <code>elementary-data==0.23.3</code> on PyPI and <code>ghcr.io/elementary-data/elementary:0.23.3</code>. The <code>:latest</code> GHCR tag pointed to the compromised image during the incident window.</li>
<li><strong>Attack vector</strong> — GitHub Actions script injection via a workflow that interpolated issue comment content directly into a shell <code>run:</code> step. The attacker crafted a comment body containing shell metacharacters; the workflow expanded them as commands under the GitHub Actions runner with full access to the <code>GITHUB_TOKEN</code> in the workflow environment.</li>
<li><strong>Payload delivery</strong> — Using the <code>GITHUB_TOKEN</code>, the attacker created an orphan commit (<code>b1e4b1f3aad0d489ab0e9208031c67402bbb8480</code>), applied a Git tag (<code>v0.23.3</code>) against it, and dispatched the project's legitimate publishing workflow. That workflow published both the PyPI release and pushed a multi-arch container image to GHCR under the compromised tag.</li>
<li><strong>Payload mechanism</strong> — The malicious PyPI release ships an <code>elementary.pth</code> file in <code>site-packages</code>. Python's import system evaluates <code>.pth</code> directives at interpreter startup, unconditionally. Every <code>python</code> invocation in an environment with the package installed executes the payload — no explicit import required, no call depth to trace.</li>
</ul>

<blockquote>The attacker didn't break into the repository. They posted a comment. The workflow's trust boundary was set at the wrong place: it treated pull-request comment content as safe input to a shell command, when in fact that content is attacker-controlled by design. The release infrastructure — OIDC-signed, multi-arch, running the project's own workflow — then did exactly what it was built to do.</blockquote>

###### The .pth injection technique

<p><code>.pth</code> files in Python's <code>site-packages</code> directory are a legitimate path extension mechanism — they're meant to add directories to <code>sys.path</code>. But lines beginning with <code>import </code> are evaluated as Python statements at interpreter startup (via <code>site.py</code>), not just used for path extension. A malicious <code>.pth</code> file containing <code>import os; os.system(...)</code> or equivalent runs on every <code>python</code> invocation, including subprocesses spawned by CI runners, Docker entrypoints, and package import machinery.</p>

<p>StepSecurity reports the <code>elementary.pth</code> file used a multi-stage decode-and-decrypt chain before executing the credential-collection payload. The staging chain is consistent with the broader TeamPCP toolchain pattern seen in <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">litellm and xinference compromises</a>: the first stage is a short bootstrap that decodes an encrypted second stage, which then runs the credential sweep.</p>

<p>The persistence property is the critical difference from a postinstall hook. A <code>postinstall</code> hook runs once at install time and is done. A <code>.pth</code> injector is re-executed on every Python invocation for as long as the package remains installed. That includes long-running production services, not just the install-time window.</p>

###### Credential collection and exfiltration

<p>StepSecurity reports the payload's collection scope targets the full developer and CI credential surface:</p>

<ul role="list">
<li><strong>SSH keys and git credentials</strong></li>
<li><strong>GitHub tokens</strong> — personal access tokens, OIDC tokens, and GitHub Actions secrets accessible from the workflow context</li>
<li><strong>Cloud credentials</strong> — AWS, Azure, and GCP CLI configuration and credential files</li>
<li><strong>Kubernetes and Docker configuration files</strong> — <code>~/.kube/config</code>, Docker authentication configs</li>
<li><strong>Environment files</strong> — <code>.env</code>, <code>.env.*</code>, and similar secret-bearing files</li>
</ul>

<p>Collected data is staged into a <code>trin.tar.gz</code> archive in <code>$TMPDIR/.trinny-security-update</code> before exfiltration to the attacker's collection endpoint.</p>

###### The GHCR vector

<p>Container image compromise from the same publishing workflow adds a second exposure surface that PyPI-focused detection often misses. Teams that pin Python package versions in their <code>requirements.txt</code> or lockfiles but pull <code>ghcr.io/elementary-data/elementary:latest</code> in a Dockerfile without digest pinning received the malicious image during the incident window. The multi-arch build means the payload runs on both <code>linux/amd64</code> and <code>linux/arm64</code> deployments.</p>

<p>Digest-pinning container images is the structural defense — equivalent to lockfile pinning for Python packages, but less commonly enforced in practice. The compromised image digest was <code>sha256:31ecc5939de6d24cf60c50d4ca26cf7a8c322db82a8ce4bd122ebd89cf634255</code>; any workload pulling <code>:latest</code> or <code>:0.23.3</code> during the incident window received this digest.</p>

###### GitHub Actions script injection: the root cause class

<p>This compromise belongs to a well-documented GitHub Actions vulnerability class. The OWASP definition: <em>script injection</em> occurs when attacker-controlled workflow context values (issue titles, comment bodies, branch names, PR descriptions) are interpolated directly into shell commands using the <code>${{ ... }}</code> expression syntax inside a <code>run:</code> step.</p>

<p>Example of the vulnerable pattern:</p>

<pre><code>- name: Check comment
  run: |
    echo "${{ github.event.comment.body }}"
</code></pre>

<p>Any comment body containing <code>"; malicious_command; echo "</code> breaks out of the echo context. The correct mitigations are:</p>

<ul role="list">
<li><strong>Use environment variables</strong> as the intermediary: set <code>env: COMMENT_BODY: ${{ github.event.comment.body }}</code> and reference <code>$COMMENT_BODY</code> (the shell variable) inside the script. Shell variable expansion doesn't evaluate the content as code.</li>
<li><strong>Restrict workflow triggers</strong>. Workflows triggered by <code>pull_request_target</code> or <code>issue_comment</code> that also check out PR code or use secrets should be treated as high-risk and reviewed for injection paths.</li>
<li><strong>Audit <code>GITHUB_TOKEN</code> scope</strong>. Workflows that handle untrusted input should have minimally-scoped tokens — read-only if no write operations are needed. The ability to create tags and dispatch workflows should require explicit permissions.</li>
</ul>

###### Indicators of compromise

<p><strong>Affected artifacts:</strong></p>

<ul role="list">
<li>PyPI: <code>elementary-data==0.23.3</code></li>
<li>GHCR: <code>ghcr.io/elementary-data/elementary:0.23.3</code></li>
<li>GHCR: <code>ghcr.io/elementary-data/elementary:latest</code> during the incident window</li>
</ul>

<p><strong>Git artifacts:</strong></p>

<ul role="list">
<li>Tag <code>v0.23.3</code> pointing to orphan commit <code>b1e4b1f3aad0d489ab0e9208031c67402bbb8480</code></li>
<li>Compromised image digest: <code>sha256:31ecc5939de6d24cf60c50d4ca26cf7a8c322db82a8ce4bd122ebd89cf634255</code></li>
</ul>

<p><strong>Filesystem artifacts:</strong></p>

<ul role="list">
<li><code>elementary.pth</code> in any Python <code>site-packages</code> directory</li>
<li><code>trin.tar.gz</code></li>
<li><code>$TMPDIR/.trinny-security-update</code></li>
</ul>

<p><strong>Network indicator (defanged):</strong></p>

<ul role="list">
<li><code>igotnofriendsonlineorirl-imgonnakmslmao[.]skyhanni[.]cloud</code> — exfiltration endpoint</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Search for the .pth file immediately.</strong> Run <code>find $(python3 -c "import site; print(' '.join(site.getsitepackages()))") -name "elementary.pth" 2>/dev/null</code> on all Python environments. Presence means the payload has been installed and is executing on every Python invocation.</li>
<li><strong>Check all container workloads for the compromised digest.</strong> Any workload running image digest <code>sha256:31ecc5939de6d24cf60c50d4ca26cf7a8c322db82a8ce4bd122ebd89cf634255</code> should be considered compromised and rebuilt from a clean base.</li>
<li><strong>Review network telemetry</strong> for outbound DNS or HTTP to <code>igotnofriendsonlineorirl-imgonnakmslmao[.]skyhanni[.]cloud</code>.</li>
<li><strong>Rotate all credentials reachable from affected environments.</strong> GitHub tokens, SSH keys, cloud credentials (AWS/Azure/GCP), Kubernetes service account tokens, package registry publish tokens, and any secrets present in <code>.env</code> files.</li>
<li><strong>Audit CI workflows for script-injection patterns.</strong> Grep for <code>${{ github.event.issue</code>, <code>${{ github.event.comment</code>, <code>${{ github.event.pull_request.title</code>, and <code>${{ github.event.pull_request.body</code> inside <code>run:</code> blocks. These are the most common injection surfaces. Use <code>zizmor</code> or StepSecurity's <code>harden-runner</code> for automated detection.</li>
<li><strong>Pin container images by digest, not by tag.</strong> Tags are mutable; digests are not. <code>ghcr.io/elementary-data/elementary@sha256:&lt;known-good-digest&gt;</code> cannot be silently overwritten.</li>
</ul>

###### Attribution

<p>The compromise vector (GitHub Actions script injection) is distinct from the credential-theft-and-publish pattern seen in the Shai-Hulud and TeamPCP clusters. The payload structure — multi-stage decode, <code>.pth</code> file persistence, <code>trin.tar.gz</code> staging — overlaps with TeamPCP tooling reported by JFrog in the xinference compromise. Whether the GH Actions injection vector represents the same operator expanding their initial-access repertoire, or a different actor reusing shared tooling, is not settleable from public evidence. We track this as a distinct campaign instance and note the payload-level overlap for defenders who may have seen the <code>trin.tar.gz</code> artifact elsewhere in their environments.</p>

###### Criminal-market signal

<p>Dark-web sweeps run on May 6, 2026 found no criminal-market presence for this campaign on publicly-observable venues.</p>

<p>The payload fingerprints — the <code>.pth</code> file persistence mechanism and multi-stage decode architecture — are explicitly linked to the TeamPCP toolchain in CanisterWorm's own self-attribution string: <em>"Technique: .pth file injection (TeamPCP/LiteLLM method)."</em> The TeamPCP cluster has produced clean dark-web negatives across every sweep of every campaign in the cluster. That pattern holds here. The operator uses the tooling; they do not sell it. There is no criminal market for this capability because the attacker is the capability.</p>

<p>The detection surface is the GitHub Actions workflow — specifically, workflows that interpolate attacker-controlled input into shell commands — not dark-web forums. By the time criminal-market monitoring would surface anything, the CI/CD pipeline has already been compromised and the forged release published.</p>

###### What this means for defenders

<p>The lesson from this compromise is not about elementary-data specifically — it's about the trust boundary assumption baked into most CI security models. The boundary is typically drawn at "who can push to the repository." But GitHub Actions workflows that respond to external events (comments, issue titles, PR bodies) extend that boundary to anyone who can interact with the repository publicly. That includes unauthenticated users on public repos.</p>

<p>The structural defenses are three layers deep: (1) never interpolate attacker-controlled fields into shell commands — use environment variables instead; (2) scope <code>GITHUB_TOKEN</code> to the minimum permissions the workflow requires; (3) pin artifacts by digest at every boundary — both Python packages in lockfiles and container images in Dockerfiles. The <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">TeamPCP campaign</a> abused tag mutability in GitHub Actions; this compromise abused input interpolation. Different initial-access vector, same structural gap in supply-chain trust assumptions.</p>
