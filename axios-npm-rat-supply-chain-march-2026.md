---
title: "axios compromised on npm: account takeover drops cross-platform RAT across 25 million weekly downloads"
date: 2026-03-31
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451ec9f48304bdea104c6a_blog%203.svg"
description: "The axios npm package, used by an estimated 25 million projects, was compromised via maintainer account takeover. Two malicious versions delivered a cross-platform remote access trojan before being pulled."
slug: axios-npm-rat-supply-chain-march-2026
type: threat-intel
campaign: "axios account takeover"
firstObserved: 2026-03-31
targetSectors:
  - Software development
  - Web applications
  - CI/CD infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - supply chain
  - account takeover
  - RAT
  - credential theft
hasArtifacts: false
sources:
  - title: "axios npm Package Compromised — Socket"
    url: "https://socket.dev/blog/axios-npm-package-compromised"
  - title: "axios Compromised on npm: Malicious Versions Drop Remote Access Trojan — StepSecurity"
    url: "https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan"
  - title: "axios incident response summary — Joe DeSimone"
    url: "https://gist.github.com/joe-desimone/36061dabd2bc2513705e0d083a9673e7"
  - title: "Datadog Security Research: axios npm package compromised"
    url: "https://app.datadoghq.com/security/feed"
diamond:
  adversary:
    operator_model: commodity
    monetization: access-sale
    confidence: unknown
  infrastructure:
    registry_layer: "npm jasonsaayman account (compromised); plain-crypto-js@4.2.1 staged dependency via nrwise account"
    c2_primary: "sfrclak[.]com:8000"
    c2_fallback: null
    exfil_channel: "Platform-specific RAT: /Library/Caches/com.apple.act.mond (macOS), %PROGRAMDATA%\\wt.exe (Windows), /tmp/ld.py (Linux)"
    c2_resilience_tier: 1
  capability:
    initial_access: account-takeover
    execution: postinstall-hook
    evasion: self-deleting-installer
    persistence: rat-binary
  victim:
    direct: "axios npm package maintainer (jasonsaayman account)"
    blast_radius: "axios@1.14.1 and axios@0.30.4 — ~25M weekly downloads, March 31 2026 window"
---

<p>The axios HTTP client for JavaScript — npm's single most downloaded package, present in an estimated 25 million projects — was compromised on March 31, 2026 via maintainer account takeover. Two malicious versions were published: <code>axios@1.14.1</code> and <code>axios@0.30.4</code>. Both delivered a cross-platform remote access trojan. At the time of initial discovery, npm's <code>latest</code> dist-tag pointed to <code>1.14.1</code>, meaning any unpinned <code>npm install axios</code> or <code>npm install</code> against an unlocked project installed the compromised version. The payload didn't steal credentials at install time and exit — it dropped a persistent RAT that gave the attacker an ongoing foothold on every machine that installed it.</p>

###### What we know

<ul role="list">
<li><strong>March 30, 2026</strong> — <code>plain-crypto-js@4.2.0</code> published to npm as a clean-looking decoy package. Later the same day, <code>plain-crypto-js@4.2.1</code> published from the throwaway account <code>nrwise</code> with a malicious <code>postinstall</code> hook. The package is designed to resemble the legitimate <code>crypto-js</code> library.</li>
<li><strong>March 31, 2026</strong> — The <code>jasonsaayman</code> maintainer account — the primary axios maintainer — is compromised. The attacker publishes <code>axios@1.14.1</code> and <code>axios@0.30.4</code>, both adding <code>plain-crypto-js@4.2.1</code> as a dependency. The maintainer email in npm metadata changed from <code>jasonsaayman@gmail.com</code> (on all legitimate releases) to <code>ifstap@proton.me</code> on both malicious versions — a forensic artifact that confirms the account compromise and distinguishes attacker-published versions from maintainer-published ones.</li>
<li><strong>Detection signal</strong> — StepSecurity noted that legitimate <code>axios</code> 1.x releases used npm's OIDC trusted publisher flow via GitHub Actions. <code>axios@1.14.1</code> was published manually, had no matching Git tag, and lacked the trusted-publisher metadata. The break in publishing provenance was the first hard signal before payload analysis confirmed the compromise. Safe version: <code>axios@1.14.0</code> or <code>0.30.3</code>.</li>
</ul>

<blockquote>The maintainer email change in npm metadata is a triage shortcut that should be automated. Legitimate releases across the axios 1.x history all carry <code>jasonsaayman@gmail.com</code>. The two malicious versions carry a Proton Mail address. Any CI system that compares the publisher identity on a version bump can flag this class of account-takeover publish before the payload is ever analyzed.</blockquote>

###### The staged dependency: plain-crypto-js

<p>The malicious payload doesn't live in the axios tarball — it lives in the staged dependency. <code>plain-crypto-js@4.2.1</code> adds a <code>postinstall</code> script that runs <code>node setup.js</code>. npm runs postinstall hooks for all dependencies during <code>npm install</code>, so installing a compromised axios transitively executes the payload without any explicit import or call by the victim's code.</p>

<p>The staging technique — publish a clean decoy version first, then publish the malicious version — serves two purposes. The decoy version establishes the package in npm's index before the attack, making it appear as a real dependency rather than a brand-new addition. And if the malicious version is yanked but the clean version remains, the dependency entry persists in the registry and can be re-weaponized.</p>

<p><code>plain-crypto-js</code> is never imported anywhere in the axios source tree. Its sole purpose is to be installed as a dependency and run its postinstall hook.</p>

###### Cross-platform RAT delivery

<p><code>setup.js</code> contacts <code>http://sfrclak[.]com:8000/6202033</code> and delivers a different payload depending on the detected platform:</p>

<ul role="list">
<li><strong>macOS</strong> — downloads a compiled RAT binary to <code>/Library/Caches/com.apple.act.mond</code> (a path designed to blend into Apple's cache directory structure) and launches it in the background.</li>
<li><strong>Windows</strong> — drops a VBScript bootstrapper and a PowerShell chain. The final payload is written to <code>%PROGRAMDATA%\wt.exe</code> (a path designed to be confused with Windows Terminal's legitimate <code>wt.exe</code> binary).</li>
<li><strong>Linux</strong> — downloads a Python RAT to <code>/tmp/ld.py</code> (using a name designed to evoke the legitimate <code>ld</code> linker) and starts it with <code>nohup</code> so it persists past the install session.</li>
</ul>

<p>All three platform paths result in a persistent remote access trojan — not a one-time credential sweep. The attacker retains access to every compromised machine for as long as the RAT remains running and the victim doesn't detect it. This is operationally different from the TeamPCP and Shai-Hulud payloads in this cluster, which are designed for one-time credential exfiltration followed by worm propagation. The axios RAT is designed for sustained access to developer workstations and CI runners.</p>

###### Self-deleting installer

<p>After launching the RAT, <code>setup.js</code> removes itself and replaces the malicious <code>package.json</code> with a clean stub. A developer or incident responder who inspects <code>node_modules/plain-crypto-js/</code> after the fact finds a normal-looking package with no visible malicious content. The payload is gone; only the running RAT process remains as evidence.</p>

<p>This self-deletion pattern makes post-infection filesystem forensics unreliable as a primary detection mechanism. Process telemetry and network telemetry — capturing the RAT's C2 beaconing and any lateral movement — are more durable artifacts than filesystem state.</p>

<p>One exception: the presence of <code>node_modules/plain-crypto-js</code> in any project is itself an indicator. Legitimate axios releases don't include this dependency. Finding the directory, even cleaned up, is evidence that an affected version was installed.</p>

###### Blast radius

<p>axios is the most downloaded package in the npm ecosystem. The compressed window between malicious publish and takedown — a matter of hours — nonetheless affected a large number of installs. Any project that ran <code>npm install</code> or <code>npm ci</code> against an unlocked dependency tree during the March 31 window may have installed the compromised version. CI/CD pipelines that install dependencies fresh on every run are the highest-risk surface: they install exact versions from npm's registry on a schedule independent of the developer's local environment.</p>

<p>Environments most likely to have been affected:</p>

<ul role="list">
<li>CI runners with <code>axios</code> as a direct dependency and no lockfile pin to a specific version</li>
<li>Developers who ran <code>npm install axios</code> or <code>npm update axios</code> on March 31 without a lockfile</li>
<li>Automated dependency update bots (Dependabot, Renovate) that may have opened PRs or merged updates to the malicious version</li>
</ul>

###### Indicators of compromise

<p><strong>Compromised packages:</strong></p>
<ul role="list">
<li><code>axios@1.14.1</code> — malicious; safe version is <code>1.14.0</code></li>
<li><code>axios@0.30.4</code> — malicious; safe version is <code>0.30.3</code></li>
<li><code>plain-crypto-js@4.2.1</code> — the staged malicious dependency; any version of this package is suspect</li>
</ul>

<p><strong>npm metadata forensics:</strong></p>
<ul role="list">
<li>Publisher email on malicious versions: <code>ifstap@proton.me</code> (all legitimate axios releases use <code>jasonsaayman@gmail.com</code>)</li>
<li>Malicious versions lack trusted publisher metadata (OIDC-sourced); published manually</li>
<li>No matching Git tag on the axios repository for <code>1.14.1</code> or <code>0.30.4</code></li>
</ul>

<p><strong>Filesystem artifacts (post-install; self-deletion may have removed setup.js):</strong></p>
<ul role="list">
<li><code>node_modules/plain-crypto-js/</code> — presence indicates affected axios version was installed</li>
<li>macOS: <code>/Library/Caches/com.apple.act.mond</code></li>
<li>Windows: <code>%PROGRAMDATA%\wt.exe</code></li>
<li>Linux: <code>/tmp/ld.py</code></li>
</ul>

<p><strong>Network indicator (defanged):</strong></p>
<ul role="list">
<li><code>sfrclak[.]com:8000</code> — dropper download and RAT C2</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Audit lockfiles and build history for March 31 installs.</strong> Any CI run or local install that resolved to <code>axios@1.14.1</code> or <code>axios@0.30.4</code> on March 31, 2026 executed the payload. Check build logs, lockfile Git history, and dependency audit outputs (<code>npm audit</code>, <code>npm ls axios</code>).</li>
<li><strong>Search for <code>node_modules/plain-crypto-js</code> across all repositories and build environments.</strong> This directory should not exist in any legitimate project. Its presence — even with a cleaned-up <code>package.json</code> — confirms the malicious axios version ran.</li>
<li><strong>Search for RAT artifacts.</strong> On macOS: <code>ls -la /Library/Caches/com.apple.act.mond</code>. On Windows: <code>%PROGRAMDATA%\wt.exe</code> (check for an executable in this path that isn't the legitimate Windows Terminal). On Linux: <code>/tmp/ld.py</code> or any Python process daemonized with <code>nohup</code> that lacks a clear owner.</li>
<li><strong>Review network telemetry for outbound connections to <code>sfrclak[.]com:8000</code>.</strong> This is a hard indicator of both the dropper firing and ongoing RAT C2.</li>
<li><strong>Treat any CI runner or developer workstation that ran the affected versions as fully compromised.</strong> Rotate npm publish tokens, cloud credentials (AWS/Azure/GCP), GitHub tokens, SSH keys, API keys, and any secrets accessible from the affected host. The RAT provides persistent access; assume the operator has had time to enumerate the environment fully.</li>
<li><strong>Rebuild affected CI runners from clean images.</strong> The RAT persists across session boundaries. A runner that isn't rebuilt may be re-compromised even after credential rotation.</li>
<li><strong>Check Dependabot/Renovate PR history.</strong> If your projects use automated dependency updates, review whether any bot opened or merged a PR bumping axios to 1.14.1 or 0.30.4 on or after March 31. Those PRs may have triggered CI jobs that installed the payload.</li>
</ul>

###### What to automate

<p>The axios compromise has three detection signals that are automatable and would have cut time-to-detection from "hours after discovery" to "minutes after publish":</p>

<ol>
<li><strong>Publisher identity drift.</strong> Compare the npm account email on each new release to the historical average for the package. A proton.me address on a package that has published from gmail.com for years is a high-confidence signal.</li>
<li><strong>OIDC/trusted-publisher gap.</strong> For packages that have adopted npm's OIDC trusted publisher flow, flag any release that wasn't published through it. The axios compromise broke this invariant immediately.</li>
<li><strong>New dependency introduction in a patch release.</strong> <code>plain-crypto-js</code> didn't exist in any prior axios release. A patch version bump that adds a dependency not present in the previous minor release is unusual and warrants review before the version is merged into lockfiles.</li>
</ol>

<p>All three signals are available from npm's package metadata before the tarball is ever installed. Integrating them into lockfile-update review processes — either via tooling like Socket's registry monitor or via custom checks in CI — would catch this class of attack at the metadata layer rather than at the payload layer.</p>

###### Attribution

<p>No public attribution to a specific group has been confirmed. The staging pattern (clean decoy package published first, malicious version second), the Proton Mail publisher identity, and the cross-platform RAT delivery are consistent with financially motivated actors operating in the npm ecosystem. The non-worm, non-credential-stealer payload profile — a persistent RAT — differentiates this from the TeamPCP and Shai-Hulud clusters, which prioritize token collection for propagation. The axios operator appears primarily interested in sustained access rather than supply-chain amplification.</p>

###### Criminal-market signal

<p>Dark-web sweeps run on May 6, 2026 found no criminal-market presence for this campaign on publicly-observable venues.</p>

<p>The absence here carries different weight than it does for the <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">TeamPCP cluster</a>. TeamPCP is operator-run — the actors use the tooling themselves and have no incentive to advertise it. The axios compromise is a commodity account-takeover delivering a persistent RAT, which is exactly the product access brokers sell. The clean negative on publicly-indexed dark-web venues does not mean the access is not being monetized. It means that if it is, the sale is happening through private Telegram channels or invite-only forums that this pipeline cannot reach.</p>

<p>The detection implication is the same regardless: registry-layer signals — the provenance gap, the manual publish breaking the OIDC chain, the staged dependency that appears in no legitimate axios release — are detectable before the RAT executes. Criminal-market monitoring adds nothing to early warning for this attack class.</p>

###### The axios compromise in context

<p>Two patterns from this incident are worth carrying forward as general detection principles.</p>

<p>First, the <strong>semantic mismatch between package name and purpose</strong>. <code>plain-crypto-js</code> sounds like a cryptography utility. It contains no cryptography. It is never imported by the dependent package. Its only code is a postinstall hook. A package that is added as a dependency but never imported anywhere in the depending package's source tree is a structural anomaly that static analysis can detect without executing the code. This pattern — phantom dependency added to a trusted package's manifest — is the canonical supply-chain payload delivery vector.</p>

<p>Second, the <strong>publishing provenance gap</strong>. The npm ecosystem's trusted publisher infrastructure (OIDC-based, GitHub Actions-sourced) creates a verifiable audit trail between a release and the source commit that produced it. A release that breaks this provenance chain — published manually when all prior releases were automated — is the most reliable early-warning signal for this class of attack. The SLSA provenance signal flagged in the <a href="https://keepsecure.io/hub/shai-hulud-npm-worm-intercom-client-2026">Shai-Hulud intercom-client compromise</a> is the same indicator: <code>7.0.3</code> had provenance attestations, <code>7.0.4</code> did not. Automating "did this release break the provenance chain its predecessors established?" is one check that cuts across all account-takeover supply-chain attacks, regardless of payload.</p>
