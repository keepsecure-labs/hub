---
title: "Glassworm compromises React Native packages with Solana blockchain C2: a new resilience tier for npm supply-chain attacks"
date: 2026-05-12
heroImage: "/art/glassworm-react-native-solana-c2-2026.png"
description: "Two React Native npm packages with ~30K weekly downloads were compromised by the Glassworm threat actor using a five-stage attack chain. The distinguishing feature is Stage 3: rather than hardcoding C2 domains, the malware resolves payload URLs via the Solana blockchain — a censorship-resistant C2 mechanism that cannot be blocked by domain takedown. Infrastructure overlap with the ForceMemo campaign (March 8–14, 2026) establishes this as a multi-campaign cluster."
slug: glassworm-react-native-solana-c2-2026
type: threat-intel
campaign: "Glassworm"
firstObserved: 2026-04-01
targetSectors:
  - Mobile development
  - React Native ecosystem
  - Software development
targetRegions:
  - Global (Russia excluded by geo-filter)
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - supply chain
  - Glassworm
  - Solana C2
  - React Native
  - blockchain C2
hasArtifacts: false
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
    geo_filter: Russia-excluding (locale/timezone check)
  infrastructure:
    registry_layer: "npm — react-native-country-select, react-native-international-phone-number"
    c2_primary: "Solana blockchain resolver — wallet 6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ; payload server 45.32.150[.]251"
    c2_fallback: null
    exfil_channel: "AES-256 encrypted payload delivered from C2 server, executed in-memory"
    c2_resilience_tier: 3
  capability:
    initial_access: npm-preinstall-hook
    execution: in-memory-eval
    evasion: RC4-string-encryption + javascript-obfuscator + geo-filter + 10-second-delay
    persistence: init.json-48h-lock
  victim:
    direct: "React Native developers and CI/CD pipelines installing affected npm packages"
    blast_radius: "~30K weekly downloads combined across two packages; all developer machines, CI runners, and build pipelines with affected versions installed"
sources:
  - title: "StepSecurity: Malicious npm releases found in popular React Native packages — 130k monthly downloads compromised"
    url: "https://www.stepsecurity.io/blog/malicious-npm-releases-found-in-popular-react-native-packages---130k-monthly-downloads-compromised"
  - title: "Datadog Security Research: Glassworm React Native npm packages card"
    url: "https://app.datadoghq.com/security/feed"
---

<p>Two popular React Native npm packages were compromised by a threat actor tracked as Glassworm, introducing a five-stage credential-harvesting attack chain that executes during <code>npm install</code>. The compromised packages — <code>react-native-country-select</code> (version 0.3.91) and <code>react-native-international-phone-number</code> (version 0.11.8) — together account for approximately 30,000 weekly downloads. Both have since been patched (0.4.0 and 0.11.9 respectively).</p>

<p>The analytically significant feature of this campaign is its command-and-control architecture. Rather than hardcoding a C2 domain or IP address, Glassworm resolves payload URLs from the Solana blockchain at runtime. An attacker-controlled wallet on the Solana Memo Program serves as a decentralized URL pointer: the malware reads base64-encoded payload addresses posted to that wallet and connects to the current payload server. This architecture means the C2 cannot be neutralized by domain takedown — the only way to disrupt payload delivery is to seize control of the Solana wallet, which is not a standard incident response action. This places Glassworm's infrastructure at C2 resilience tier 3, equivalent to the ICP canister architecture used by CanisterWorm.</p>

<p>Infrastructure overlap with the ForceMemo campaign (March 8–14, 2026) — identical Solana wallet, matching geo-filter logic, and the same AES encryption pattern — establishes Glassworm as a multi-campaign cluster, not a one-off incident. This finding documents the React Native arm of that cluster.</p>

###### What we know

<ul role="list">
<li><strong>Packages affected:</strong> <code>react-native-country-select</code> version 0.3.91 (malicious); 0.4.0 is clean. <code>react-native-international-phone-number</code> version 0.11.8 (malicious); 0.11.9 is clean.</li>
<li><strong>Combined reach:</strong> Approximately 30,000 weekly downloads — meaning a significant fraction of the React Native mobile development ecosystem was exposed.</li>
<li><strong>Discovery:</strong> First reported by StepSecurity. Datadog Security Research confirmed the activity and corroborated the ForceMemo infrastructure overlap.</li>
<li><strong>Geo-exclusion:</strong> The malware explicitly stops execution on systems with Russian locale settings (<code>LANG</code>, <code>LANGUAGE</code>, <code>LC_ALL</code>) or Russian timezones (<code>Europe/Moscow</code>, <code>Asia/Krasnoyarsk</code>, <code>Asia/Vladivostok</code>, <code>MSK</code>). This pattern is consistent with Russian-origin malware avoiding domestic prosecution — a standard operational constraint for financially motivated actors operating from Russia.</li>
</ul>

###### Five-stage attack chain

<p>The following stage analysis is sourced from StepSecurity's primary report, as characterized by Datadog Security Research. Primary-source verification of payload artifacts and exact IOCs is pending (Task #14).</p>

<p><strong>Stage 1 — Preinstall execution.</strong> The compromised versions add a <code>preinstall</code> script to <code>package.json</code> that runs <code>install.js</code> automatically during <code>npm install</code>, before any user code executes. The script uses RC4-based string encryption and array rotation obfuscation characteristic of the <code>javascript-obfuscator</code> toolchain — the same obfuscation framework used in other npm supply-chain campaigns. At this stage, execution is guaranteed for any developer or CI/CD system that runs <code>npm install</code> with either affected package in the dependency tree.</p>

<p><strong>Stage 2 — Geo-filtering.</strong> After a 10-second delay (a timing technique that can evade some sandboxes with short execution windows), the malware checks the environment for Russian locale and timezone markers. Russian locale detection halts execution entirely. The 10-second delay before the check is operationally meaningful: it allows the process to appear legitimate during automated analysis before taking any detectable action.</p>

<p><strong>Stage 3 — Solana blockchain C2 resolution.</strong> The malware queries nine Solana RPC endpoints for the wallet address <code>6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ</code> using the <code>getSignaturesForAddress</code> method. It reads base64-encoded payload URLs posted via the Solana Memo Program to that wallet. This resolves to the current payload server address.</p>

<blockquote>This is the critical capability distinction. A conventional C2 domain can be taken down, sinkholes, or blocked at DNS. A Solana wallet cannot. The attacker can post a new payload URL to the wallet at any time — rotating the actual server behind the same permanent blockchain anchor. From a defender's perspective, there is no domain to block at Stage 3. Detection must occur at Stage 1 (preinstall hook) or Stage 4 (outbound connection to the resolved server).</blockquote>

<p><strong>Stage 4 — Payload delivery.</strong> The malware connects to the resolved server (<code>45.32.150[.]251</code>) with platform identification sent via an <code>os</code> HTTP header. The server delivers a base64-encoded, AES-256-encrypted payload. The AES key and IV are provided in the HTTP response headers — the decryption material travels with the payload, meaning the encrypted payload on disk is not a stable IOC.</p>

<p><strong>Stage 5 — In-memory execution with persistence gate.</strong> The decrypted payload executes entirely in memory without writing to disk. On macOS and Linux, it uses <code>eval()</code>; on other platforms, <code>Node.js vm.Script</code>. A persistence lock file at <code>~/init.json</code> prevents re-execution within 48-hour windows — indicating the operator does not want duplicate collection runs from the same machine. The lock file is also a detection artifact: its presence on a developer machine confirms prior execution even if the npm package has since been updated.</p>

###### Why React Native was targeted

<p>React Native mobile development environments are an undermonitored credential surface. The ecosystem profile that makes them valuable targets:</p>

<ul role="list">
<li><strong>Cloud service credentials for mobile backends.</strong> React Native applications routinely connect to AWS, GCP, Firebase, and similar services. Developer machines and CI/CD pipelines hold the credentials for those backends. An <code>npm install</code> in a React Native project executes with access to the full credential surface of the developer's machine.</li>
<li><strong>Lower security baseline than backend ecosystems.</strong> The mobile development community has historically received less supply-chain security tooling than backend-focused npm users. Preinstall hook monitoring and SCA scanning penetration is lower in mobile-focused teams than in DevOps-focused teams.</li>
<li><strong>High-volume packages as propagation multipliers.</strong> 30,000 weekly downloads is approximately one order of magnitude below the largest npm supply-chain targets (axios: ~50M/week), but still represents meaningful scale for initial credential collection. The goal is not maximum blast radius — it is reaching the specific credential classes the operator wants.</li>
</ul>

###### ForceMemo cluster connection: four-step analysis

<p><strong>Observation:</strong> The Glassworm campaign uses Solana wallet <code>6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ</code> as its C2 resolver. The ForceMemo campaign (March 8–14, 2026), which targeted GitHub Python repositories, used an identical Solana blockchain C2 architecture with matching Russia geo-filter logic and AES-256 encryption. This infrastructure overlap was noted by Datadog Security Research in characterizing StepSecurity's findings.</p>

<p><strong>Mechanism:</strong> A specific Solana wallet address is an operator-controlled artifact: it requires possession of the corresponding private key to post new payload URLs. Two independent campaigns reusing the same wallet address is not a coincidence — the wallet is the persistent C2 anchor for the same operator's infrastructure. The geo-filter logic is a separate corroborating artifact: the same set of Russian locale and timezone strings appearing in two campaigns indicates shared codebase, not independently-written malware that happens to target the same region.</p>

<p><strong>Inference:</strong> Glassworm and ForceMemo share the same operator infrastructure and almost certainly the same codebase. This is a multi-campaign cluster. The March 8–14 ForceMemo campaign targeted GitHub Python repositories; the Glassworm campaign targeted React Native npm packages. The same actor is running operations across multiple ecosystems using the same Solana-backed C2 anchor.</p>

<p><strong>Confidence bound:</strong> This inference depends on the wallet address match being confirmed against StepSecurity's primary analysis for ForceMemo. If the ForceMemo wallet address differs and the overlap is at the AES/geo-filter pattern level only, the confidence drops to moderate — pattern overlap alone is insufficient for cluster attribution. Primary-source verification of the ForceMemo wallet address is the key falsification check (Task #14).</p>

###### Indicators of compromise

<p><strong>Compromised package versions:</strong></p>
<ul role="list">
<li><code>react-native-country-select</code> version <strong>0.3.91</strong> — malicious; patch to 0.4.0</li>
<li><code>react-native-international-phone-number</code> version <strong>0.11.8</strong> — malicious; patch to 0.11.9</li>
</ul>

<p><strong>Filesystem artifacts:</strong></p>
<ul role="list">
<li><code>~/init.json</code> — persistence lock file; presence confirms prior payload execution even if the malicious package version has since been replaced</li>
<li><code>install.js</code> in preinstall script position — RC4-obfuscated; any preinstall hook running a file with this name warrants inspection</li>
</ul>

<p><strong>Network indicators (defanged):</strong></p>
<ul role="list">
<li><code>45.32.150[.]251</code> — payload delivery server; outbound connection during <code>npm install</code> from a React Native package is not expected behavior</li>
<li>Solana RPC endpoint queries for <code>getSignaturesForAddress</code> from wallet <code>6YGcuyFRJKZtcaYCCFba9fScNUvPkGXodXE1mJiSzqDJ</code> — any npm install-time query to a Solana RPC endpoint is highly anomalous</li>
</ul>

<p><strong>Behavioral indicators:</strong></p>
<ul role="list">
<li>10-second sleep during <code>npm install</code> execution — timing evasion technique; legitimately no npm package requires a 10-second install delay</li>
<li>Outbound connection carrying <code>os</code> HTTP header with platform identification from a package installation context</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Check for <code>~/init.json</code> on all developer machines.</strong> This file's presence is a confirmed-execution artifact that persists beyond package update. If found, treat the machine as compromised and rotate all credentials.</li>
<li><strong>Update affected packages immediately.</strong> <code>react-native-country-select</code> → 0.4.0; <code>react-native-international-phone-number</code> → 0.11.9. The malicious code is absent from these versions.</li>
<li><strong>Audit CI/CD logs for Solana RPC connections.</strong> No npm package should make outbound connections to Solana RPC endpoints during installation. This is a class-level detection rule that covers the entire Glassworm/ForceMemo C2 architecture: block or alert on any outbound connection to <code>api.mainnet-beta.solana.com</code>, <code>solana-api.projectserum.com</code>, or other Solana RPC endpoints from <code>npm install</code> contexts.</li>
<li><strong>Monitor for <code>45.32.150[.]251</code> in egress logs.</strong> This IP has no legitimate use in development contexts.</li>
<li><strong>Enable preinstall hook auditing.</strong> <code>npm install --ignore-scripts</code> prevents preinstall hooks from running. For CI/CD pipelines that can tolerate this constraint, it is the structural defense against this class of attack. For development environments where scripts are necessary, configure egress monitoring on the install process.</li>
<li><strong>Do not rely on domain-block lists for this campaign.</strong> The blockchain C2 resolver means the actual payload server IP can change without modifying the malicious package. Domain-block strategies that work for tier-1 C2 do not apply here.</li>
</ul>

###### Attribution

<p>The Russia geo-filter is the strongest available attribution signal: Russian-origin financially motivated actors routinely exclude domestic targets to reduce their legal exposure. This does not confirm Russian origin — the filter could be adopted by non-Russian actors for misdirection — but the pattern is operationally consistent with Russian-speaking financially motivated threat actors.</p>

<p>The multi-ecosystem pattern (GitHub Python repositories in ForceMemo, npm React Native in Glassworm) is consistent with a technically capable operator testing the same C2 architecture across different surfaces. The credential scope targeted (cloud credentials, developer tokens) aligns with a credential-collection monetization model rather than ransomware or destructive operations.</p>

<p>LAPSUS$ attribution or state-actor attribution for this campaign is unsupported by available evidence.</p>

###### Criminal-market signal

<p>No dark-web presence for Glassworm or ForceMemo campaign tooling has been observed. The Russia geo-exclusion, the sophisticated Solana C2 architecture, and the multi-campaign operational pattern are all consistent with an operator-run credential harvesting operation — the operator uses the credentials directly, not via criminal-market sale. Detection must occur at the registry and installation layer. Dark-web monitoring for this campaign class is expected to return clean negatives (H2 operator-run pattern).</p>
