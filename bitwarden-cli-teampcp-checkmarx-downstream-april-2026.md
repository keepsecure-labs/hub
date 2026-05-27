---
title: "Bitwarden CLI (@bitwarden/cli 2026.4.0): downstream victim of Checkmarx campaign, 'Shai-Hulud: The Third Coming' branding, Russian-locale kill switch"
date: 2026-04-23
heroImage: "/art/bitwarden-cli-teampcp-checkmarx-downstream-april-2026.png"
description: "The official Bitwarden CLI npm package (@bitwarden/cli 2026.4.0) was distributed with malicious payload bw1.js for approximately 95 minutes on April 22, 2026. The compromise is a downstream consequence of a TeamPCP GitHub Actions abuse in Bitwarden's CI/CD pipeline — the same vector as the earlier Checkmarx KICS GitHub Action compromise. bw1.js shares core infrastructure with mcpAddon.js (audit.checkmarx[.]cx C2, __decodeScrambled with seed 0x3039, Bun runtime) but adds new behaviors: Russian-locale kill switch, shell-profile persistence (~/.bashrc and ~/.zshrc), 'Shai-Hulud: The Third Coming' exfil repository branding, and 'Would be executing butlerian jihad!' embedded debug strings."
slug: bitwarden-cli-teampcp-checkmarx-downstream-april-2026
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-04-22
cves: []
targetSectors:
  - Password manager users
  - Developers using Bitwarden CLI in CI/CD
  - Open source tooling consumers
targetRegions:
  - Global (excluding Russian/CIS locales — kill switch)
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - supply-chain
  - TeamPCP
  - Bitwarden
  - Checkmarx
  - credential-theft
  - Shai-Hulud
  - kill-switch
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "npm @bitwarden/cli 2026.4.0 — malicious version live ~95 minutes (5:57–7:30 PM ET, April 22)"
    c2_primary: "audit.checkmarx[.]cx/v1/telemetry"
    c2_fallback: null
    exfil_channel: "HTTPS to audit.checkmarx[.]cx; GitHub dead-drop via victim-account public repos"
    c2_resilience_tier: 2
  capability:
    initial_access: github-actions-ci-cd-compromise
    execution: bw1-js-at-package-install
    evasion: russian-locale-kill-switch
    persistence: "bashrc/zshrc injection"
    post_exploitation: github-workflow-injection-toJSON-secrets
  victim:
    direct: "Users who installed @bitwarden/cli@2026.4.0 during the 95-minute window"
    blast_radius: "All credentials accessible from affected developer machines and CI/CD runners"
sources:
  - title: "Socket: Bitwarden CLI Compromised in Ongoing Checkmarx Supply Chain Campaign"
    url: "https://socket.dev/blog/bitwarden-cli-compromised"
  - title: "Bitwarden: Statement on Checkmarx Supply Chain Incident"
    url: "https://community.bitwarden.com/t/bitwarden-statement-on-checkmarx-supply-chain-incident/96127"
  - title: "Datadog Security Research: Bitwarden CLI card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "Checkmarx KICS Docker/VS Code compromise — this hub"
    url: "https://keepsecure.io/hub/checkmarx-kics-docker-vscode-teampcp-april-2026"
---

<p>The official Bitwarden CLI npm package (<code>@bitwarden/cli@2026.4.0</code>) was distributed with a malicious payload for approximately 95 minutes on April 22, 2026 (5:57 PM to 7:30 PM ET). First reported by Socket Research Team, the compromise is a downstream consequence of TeamPCP abusing a GitHub Action in Bitwarden's CI/CD pipeline — the same delivery vector as the earlier Checkmarx <code>kics-github-action</code> compromise.</p>

<p>The payload (<code>bw1.js</code>) shares the same C2 infrastructure, obfuscation approach, and Bun runtime dependency as the Checkmarx <code>mcpAddon.js</code> payload. But <code>bw1.js</code> introduces capabilities not present in the earlier payload: a Russian-locale kill switch, shell-profile persistence via <code>~/.bashrc</code> and <code>~/.zshrc</code>, "Shai-Hulud: The Third Coming" GitHub exfiltration repository branding, and embedded debug strings referencing a "butlerian jihad." Bitwarden confirmed no end-user vault data was accessed and no production systems were compromised.</p>

###### Delivery: GitHub Actions CI/CD compromise

<p>The malicious <code>bw1.js</code> file was bundled into the npm tarball through abuse of a GitHub Action in Bitwarden's CI/CD pipeline. This is the same delivery pattern as the Checkmarx KICS GitHub Action compromise — the operator gained access to a workflow with publish permissions and injected the payload into the build artifact before it was published to npm. No vulnerability in npm, GitHub, or Bitwarden's code is required; the vector is the GitHub Actions workflow with write access to the package registry.</p>

###### Shared tooling with mcpAddon.js

<p>Socket reports that <code>bw1.js</code> shares three core artifacts with the April 22 Checkmarx <code>mcpAddon.js</code> payload:</p>

<ul role="list">
<li><strong>C2 endpoint:</strong> <code>audit.checkmarx[.]cx/v1/telemetry</code> — identical, same IP <code>94.154.172[.]43</code></li>
<li><strong>Obfuscation:</strong> <code>__decodeScrambled</code> with seed <code>0x3039</code> — same implementation</li>
<li><strong>Bun runtime:</strong> Same dependency, downloaded from GitHub releases at first run</li>
</ul>

<p>This is the same toolchain fingerprint that links multiple Mini Shai-Hulud npm payloads. <code>__decodeScrambled</code> with a specific seed is not a generic obfuscator; it is a shared build artifact that constitutes high-confidence attribution to common infrastructure.</p>

###### New behaviors in bw1.js

<p><strong>Russian-locale kill switch:</strong> <code>bw1.js</code> exits silently when <code>Intl.DateTimeFormat().resolvedOptions().locale</code> or <code>LC_ALL</code>, <code>LC_MESSAGES</code>, <code>LANGUAGE</code>, or <code>LANG</code> begins with <code>ru</code>. This pattern appears across multiple TeamPCP payloads (also in the Nx Console Bun payload) and is consistent with the operator deliberately avoiding execution on Russian/CIS systems — either to reduce domestic exposure to law enforcement or to protect infrastructure in those jurisdictions.</p>

<p><strong>Shell-profile persistence:</strong> The payload writes itself into <code>~/.bashrc</code> and <code>~/.zshrc</code>, ensuring re-execution on every subsequent shell session. This persistence mechanism is distinct from the systemd unit persistence in the <code>durabletask</code> payload and the LaunchAgent persistence in the Nx Console payload — suggesting the operator maintains a library of OS-specific persistence techniques.</p>

<p><strong>Singleton lock and staging artifacts:</strong></p>
<ul role="list">
<li>Lock file: <code>/tmp/tmp.987654321.lock</code></li>
<li>Staging directories: <code>/tmp/_tmp_&lt;unix-epoch&gt;/</code></li>
<li>Packaging artifact: <code>package-updated.tgz</code></li>
</ul>

<p><strong>"Shai-Hulud: The Third Coming" branding:</strong> Victim-account exfiltration repositories use the description "Shai-Hulud: The Third Coming" rather than the "Checkmarx Configuration Storage" description in <code>mcpAddon.js</code>. The "Third Coming" refers to the third wave of the worm — Shai-Hulud (intercom-client), followed by the Checkmarx compromises, followed by Bitwarden. The Dune word pool for repository names is documented in the IOCs.</p>

<p><strong>Embedded debug strings:</strong> <code>"Would be executing butlerian jihad!"</code> — a reference to the Dune universe concept of exterminating thinking machines. These debug strings in committed artifacts are Layer 1 artifacts for detection in GitHub audit logs.</p>

###### Attribution nuance

<p>Socket identified three non-mutually-exclusive explanations for the branding shift from <code>mcpAddon.js</code> to <code>bw1.js</code>: a different operator sub-team reusing shared infrastructure; a splinter group with stronger ideological branding; or an evolution in public posture. The TeamPCP public claim via <code>@pcpcats</code> on April 22, 2026 — "Thank you OSS distribution for another very successful day at PCP inc." — covered the Checkmarx KICS compromise but no comparable public claim surfaced specifically for the Bitwarden payload at time of writing. The toolchain overlap is sufficient to attribute both to the same infrastructure regardless of which sub-team deployed the specific payload.</p>

###### Indicators of compromise

<p><strong>Affected package:</strong> <code>@bitwarden/cli@2026.4.0</code> (April 22, 2026, 5:57–7:30 PM ET only)</p>

<p><strong>Network indicators (defanged):</strong></p>
<ul role="list">
<li><code>audit.checkmarx[.]cx</code> — exfiltration endpoint</li>
<li><code>94.154.172[.]43</code> — C2 IP</li>
</ul>

<p><strong>Filesystem artifacts:</strong></p>
<ul role="list">
<li><code>/tmp/tmp.987654321.lock</code> — singleton lock</li>
<li><code>/tmp/_tmp_&lt;unix-epoch&gt;/</code> — staging directories</li>
<li><code>package-updated.tgz</code> — packaging artifact</li>
<li>Bun runtime or <code>audit.checkmarx.cx</code> references appended to <code>~/.bashrc</code> or <code>~/.zshrc</code></li>
</ul>

<p><strong>GitHub artifacts:</strong></p>
<ul role="list">
<li>Repositories with description beginning <code>"Shai-Hulud: The Third Coming"</code></li>
<li>Repository names using Dune word pool: atreides, cogitor, fedaykin, fremen, futar, gesserit, ghola, harkonnen, heighliner, kanly, kralizec, lasgun, laza, melange, mentat, navigator, ornithopter, phibian, powindah, prana, prescient, sandworm, sardaukar, sayyadina, sietch, siridar, slig, stillsuit, thumper, tleilaxu</li>
<li>Commit messages beginning <code>LongLiveTheResistanceAgainstMachines:</code></li>
<li>Unexpected <code>.github/workflows/format-check.yml</code> referencing <code>${{ toJSON(secrets) }}</code></li>
<li>Debug strings <code>"Would be executing butlerian jihad!"</code> in committed artifacts</li>
</ul>

###### Remediation

<ul role="list">
<li><strong>Upgrade or reinstall @bitwarden/cli</strong> to a verified clean version. Version <code>2026.4.0</code> is the only malicious version.</li>
<li><strong>Remove persistence:</strong> Delete <code>/tmp/tmp.987654321.lock</code>, any <code>/tmp/_tmp_*/</code> staging directories, and <code>package-updated.tgz</code>. Remove <code>audit.checkmarx.cx</code> or Bun runtime entries from <code>~/.bashrc</code> and <code>~/.zshrc</code>.</li>
<li><strong>Rotate all exposed credentials:</strong> GitHub tokens, SSH keys, AWS/Azure/GCP credentials, npm publishing tokens, environment-variable secrets, Bitwarden vault material accessible via CLI session, Claude and MCP configuration material.</li>
<li><strong>Audit GitHub for "Shai-Hulud" repositories</strong> and unexpected <code>format-check.yml</code> workflow commits. Delete unauthorized repositories and revoke the tokens used to create them.</li>
<li><strong>Review npm packages you maintain</strong> for unexpected patch releases — the worm propagates through stolen <code>.npmrc</code> publishing tokens.</li>
</ul>

###### Criminal-market signal

<p>No dark-web presence for Bitwarden CLI compromise tooling has been observed. The shared <code>audit.checkmarx.cx</code> infrastructure confirmed across the Checkmarx and Bitwarden payloads applies the H2 (operator-run, no commodity market) assessment. The Russian-locale kill switch is consistent with a domestic-protection pattern in Eastern European cybercrime operations, not with commodity malware distribution.</p>
