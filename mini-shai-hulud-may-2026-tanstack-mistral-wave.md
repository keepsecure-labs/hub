---
title: "Mini Shai-Hulud May 2026: SLSA-signed malicious packages across TanStack, Mistral AI, Guardrails AI, UiPath, and OpenSearch"
date: 2026-05-12
heroImage: "/art/mini-shai-hulud-may-2026-tanstack-mistral-wave.png"
description: "The May 11, 2026 Mini Shai-Hulud wave compromised 42 TanStack packages and spread to Mistral AI, Guardrails AI, UiPath, and OpenSearch — 373–416 malicious package-version entries across npm and PyPI. The attack produced valid SLSA Build Level 3 provenance attestations on malicious packages, bypassing the supply-chain defense that most organizations consider sufficient. The payload introduces two capabilities not seen in prior waves: Session P2P network exfiltration (no domain to block), and post-uninstall persistence via Claude Code hooks and VS Code auto-run tasks."
slug: mini-shai-hulud-may-2026-tanstack-mistral-wave
type: threat-intel
campaign: "Mini Shai-Hulud / Team PCP"
firstObserved: 2026-05-11
targetSectors:
  - JavaScript / React ecosystem
  - AI / ML tooling
  - Enterprise automation
  - Search infrastructure
targetRegions:
  - Global (Russia excluded; Israel/Iran targeted for destructive routine)
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - npm
  - PyPI
  - supply chain
  - Mini Shai-Hulud
  - TeamPCP
  - TanStack
  - Mistral AI
  - SLSA bypass
  - Session P2P
  - Claude Code hooks
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
    destructive_component: conditional (Israel/Iran hosts, 1-in-6 probability)
  infrastructure:
    registry_layer: "npm — 42 TanStack packages, UiPath, OpenSearch; PyPI — Mistral AI, Guardrails AI"
    c2_primary: "Session P2P network via *.getsession[.]org (encrypted P2P exfiltration)"
    c2_fallback: "api.masscan[.]cloud, git-tanstack[.]com"
    exfil_channel: "Session P2P encrypted traffic — indistinguishable from messenger usage at network layer"
    c2_resilience_tier: 4
  capability:
    initial_access: pull_request_target-oidc-theft + github-actions-cache-poisoning
    execution: bun-runtime-loader + router_runtime.js
    evasion: valid-SLSA-level-3-attestation + russian-locale-exit
    persistence: claude-code-hooks + vscode-autotask (survives-npm-uninstall)
    destructive: conditional-rm-rf (Israel/Iran locale, 1-in-6)
  victim:
    direct: "TanStack, Mistral AI, Guardrails AI, UiPath, OpenSearch CI/CD pipelines"
    blast_radius: "373–416 malicious package-version entries; Endor Labs: 160+ npm packages; any developer or CI environment that installed affected versions"
sources:
  - title: "BleepingComputer: Shai Hulud attack ships signed malicious TanStack, Mistral npm packages"
    url: "https://www.bleepingcomputer.com/news/security/shai-hulud-attack-ships-signed-malicious-tanstack-mistral-npm-packages/"
  - title: "Socket: TanStack NPM Packages Compromised in Ongoing Mini Shai-Hulud Supply-Chain Attack"
    url: "https://socket.dev/blog/tanstack-npm-packages-compromised-mini-shai-hulud-supply-chain-attack"
  - title: "StepSecurity: Mini Shai-Hulud Is Back — Self-Spreading Supply Chain Attack Compromises TanStack npm Packages"
    url: "https://www.stepsecurity.io/blog/mini-shai-hulud-is-back-a-self-spreading-supply-chain-attack-hits-the-npm-ecosystem"
  - title: "Datadog Security Research: Shai-Hulud worm card (updated May 11)"
    url: "https://app.datadoghq.com/security/feed"
  - title: "Mini Shai-Hulud SAP CAP precursor — this hub"
    url: "https://keepsecure.io/hub/mini-shai-hulud-sap-npm-bun-worm-april-2026"
  - title: "npm supply-chain worm cluster analysis — this hub"
    url: "https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026"
---

<p>On May 11, 2026, Mini Shai-Hulud — the self-propagating npm worm documented in the <a href="https://keepsecure.io/hub/mini-shai-hulud-sap-npm-bun-worm-april-2026">April 29 SAP CAP compromise</a> — returned at scale. The attack began with TanStack, the widely-used React data-fetching library ecosystem, and spread to Mistral AI, Guardrails AI, UiPath, and OpenSearch using stolen CI/CD credentials. Aikido recorded 373 malicious package-version entries; Socket tracked 416 compromised artifacts across npm and PyPI; Endor Labs identified over 160 compromised npm packages.</p>

<p>Three capabilities distinguish this wave from all prior Mini Shai-Hulud activity:</p>

<ol>
<li><strong>Valid SLSA Build Level 3 provenance attestations on malicious packages.</strong> The 84 malicious TanStack package versions carried genuine, unforgeable SLSA attestations — the packages were signed by npm's infrastructure and tied to the legitimate TanStack/router release workflow. From a developer's perspective, they appeared cryptographically authentic. This is the first confirmed supply-chain attack to bypass SLSA Level 3 provenance as a defense.</li>
<li><strong>Session P2P network exfiltration.</strong> Previous waves used GitHub dead-drop commits (api.github.com). This wave exfiltrates via the Session (formerly Loki) P2P messenger network. The traffic is encrypted messenger traffic with no central endpoint — standard domain blocklists and SIEM rules for suspicious GitHub API calls do not apply.</li>
<li><strong>Post-uninstall persistence.</strong> The payload writes itself into Claude Code hooks and VS Code auto-run tasks. Removing the malicious npm package does not remove the payload — it remains active on the developer's machine until explicitly hunted and deleted.</li>
</ol>

###### What we know

<ul role="list">
<li><strong>May 11, 2026</strong> — StepSecurity's OSS Package Security Feed detected malicious versions in official @tanstack packages. The attack spread to additional projects using stolen credentials throughout the day.</li>
<li><strong>Entry point:</strong> TanStack/router repository. The attacker exploited three chained vulnerabilities: (1) a <code>pull_request_target</code> workflow that ran in the context of the target repository, granting access to repository secrets on pull requests from forks; (2) GitHub Actions cache poisoning; (3) OIDC token theft from runner process memory.</li>
<li><strong>Scale:</strong> 84 malicious versions across 42 TanStack packages (StepSecurity/TanStack post-mortem). Socket: 416 compromised artifacts. Aikido: 373 malicious package-version entries. Endor Labs: 160+ npm packages.</li>
<li><strong>All victims share the same payload.</strong> SafeDep confirmed that compromised Mistral AI and TanStack packages drop identical credential-stealing payloads. This is a single worm, not parallel independent attacks.</li>
<li><strong>Valid SLSA attestations:</strong> The malicious packages carried valid Sigstore attestations and GitHub Actions signatures tied to the legitimate TanStack/router Release workflow. Snyk noted this means "the attack produces valid SLSA Build Level 3 attestations for malicious packages."</li>
</ul>

###### The SLSA bypass: four-step analysis

<p>This is the analytically significant finding. The community has treated SLSA Level 3 provenance as a strong supply-chain defense. This attack falsifies that assumption.</p>

<p><strong>Observation:</strong> 84 malicious TanStack package versions carry valid SLSA Level 3 attestations, issued by npm's signing infrastructure, tied to the legitimate TanStack/router release workflow. The attestations are genuine — they were not forged. Multiple security vendors confirmed this independently.</p>

<p><strong>Mechanism:</strong> SLSA Level 3 attestations prove WHERE a package was built — specifically, that it was built in the named CI/CD workflow. They do not prove that the build workflow was uncompromised at the time of the build. The attacker stole OIDC tokens from the <code>pull_request_target</code> workflow by running attacker-controlled code in the legitimate workflow context. The resulting packages were genuinely built by the TanStack/router Release workflow — with attacker-injected payloads included. The signing infrastructure signed what it was given.</p>

<p><strong>Inference:</strong> SLSA Level 3 is a necessary but not sufficient supply-chain defense. It binds packages to build pipelines. It does not bind build pipelines to integrity. An attestation answers "was this package built by workflow X?" — it cannot answer "was workflow X clean when it built this?" A compromised <code>pull_request_target</code> workflow that runs in the privileged repository context can inject any payload and still receive a valid attestation.</p>

<p><strong>Confidence bound:</strong> This inference is robust against alternative explanations. The only scenario where SLSA Level 3 would provide stronger protection is if the OIDC issuance were scoped to verified-clean workflow runs — a capability not present in current npm/GitHub OIDC integration. SLSA Level 4 (proposed, unreleased) includes builder integrity requirements that would partially address this by constraining which commits can trigger signing workflows.</p>

<blockquote>The practical implication: organizations that have implemented SLSA Level 3 verification as a complete supply-chain defense need to revisit their threat model. SLSA Level 3 tells you the package came from the workflow it claims. It does not tell you the workflow was clean. Behavioral analysis at install time is required in addition to, not instead of, provenance verification.</blockquote>

###### TanStack entry chain: three vulnerabilities

<p>The attack on TanStack was not a single vulnerability — it required chaining three weaknesses in the CI/CD configuration:</p>

<ol>
<li><p><strong><code>pull_request_target</code> workflow with access to repository secrets.</strong> GitHub's <code>pull_request_target</code> event was designed to allow PR workflows to have write access to the target repository, enabling actions like labeling PRs from forks. When a workflow using <code>pull_request_target</code> checks out PR code and runs it, that code executes with the target repository's secrets — including OIDC tokens. TanStack/router had such a workflow.</p></li>
<li><p><strong>GitHub Actions cache poisoning.</strong> The attacker poisoned a GitHub Actions cache entry to inject malicious code into subsequent workflow runs. Cache poisoning is a known attack class against GitHub Actions that enables persistent code injection without requiring ongoing access.</p></li>
<li><p><strong>OIDC token theft from runner memory.</strong> StepSecurity confirmed the payload reads GitHub Actions runner process memory to extract OIDC tokens — the same technique used in the April 29 SAP CAP compromise (where <code>Runner.Worker</code> memory was read to extract GitHub Actions secrets). With a valid OIDC token from a release workflow, the attacker could publish packages through npm's legitimate publishing path, receiving SLSA attestations from npm's signing infrastructure.</p></li>
</ol>

<p>Endor Labs identified an additional delivery technique: an orphaned commit pushed to a fork of TanStack/router. Because GitHub's shared fork object storage makes commits accessible from any fork via the parent repository, this orphaned commit could be referenced as an optional dependency and fetched during npm install without appearing in any branch.</p>

###### New capabilities in this wave

<p><strong>Session P2P exfiltration.</strong> Previous Mini Shai-Hulud waves exfiltrated credentials via GitHub dead-drop repositories (attacker-controlled public repos with randomized names). This wave uses the Session P2P network — an encrypted, decentralized messenger network with no central server. Exfil traffic appears as encrypted messenger communications. The C2 endpoints are <code>*.getsession[.]org</code>. There is no single domain to sinkhole, no GitHub API pattern to alert on, and no central repository to take down. This is a meaningful operational upgrade from the GitHub dead-drop pattern.</p>

<p><strong>Post-uninstall persistence.</strong> The payload writes itself into:</p>
<ul role="list">
<li><strong>Claude Code hooks</strong> — configuration files that Claude Code reads and executes on startup</li>
<li><strong>VS Code auto-run tasks</strong> — tasks defined in <code>.vscode/tasks.json</code> that execute automatically when a workspace is opened</li>
</ul>
<p>Both mechanisms survive <code>npm uninstall</code> of the malicious package. The payload is not in the package directory — it is in IDE configuration files that will be read and executed the next time the developer opens their IDE or runs Claude Code. Remediation requires explicitly hunting and deleting these files, not just updating the package.</p>

<p><strong>Destructive routine.</strong> On hosts where the detected locale matches Israel or Iran, the payload has a 1-in-6 probability of executing <code>rm -rf /</code>. Microsoft Threat Intelligence confirmed this in analysis of the Mistral AI PyPI payload. This behavior parallels CanisterWorm's March 2026 destructive routine that targeted Kubernetes platforms matching Iran's timezone. The consistency across campaigns strengthens the inference of a shared codebase.</p>

###### Mistral AI and PyPI branch

<p>The Mistral AI compromise extended the worm to PyPI. Microsoft Threat Intelligence analyzed the payload delivered via a malicious <code>@mistralai/mistralai</code> version. The payload was named <code>transformers.pyz</code> — a deliberate impersonation of the Hugging Face Transformers library. Three <code>@mistralai/mistralai</code> versions were published in rapid succession on May 11: 2.2.2, 2.2.3, and 2.2.4 within an 8-minute window (22:45–22:53 UTC), consistent with the worm's automated publication cadence observed in the April 29 wave.</p>

<p>The PyPI branch uses the same credential-stealing payload (confirmed by SafeDep) but packages it as a Python zip archive rather than a JavaScript bundle. The geofencing logic is identical: Russian locale detected → exit; Israel/Iran detected → 1-in-6 destructive wipe.</p>

###### Indicators of compromise

<p><strong>Compromised victims (partial — full lists at vendor sources):</strong></p>
<ul role="list">
<li>42 <code>@tanstack/*</code> npm packages — 84 malicious versions (TanStack post-mortem)</li>
<li><code>@mistralai/mistralai</code> versions 2.2.2, 2.2.3, 2.2.4 (npm); PyPI packages pending full vendor list</li>
<li>Guardrails AI, UiPath, OpenSearch npm/PyPI packages — version details in vendor reports</li>
</ul>

<p><strong>Network indicators (defanged):</strong></p>
<ul role="list">
<li><code>*.getsession[.]org</code> — Session P2P exfil endpoints</li>
<li><code>api.masscan[.]cloud</code> — C2 fallback endpoint</li>
<li><code>git-tanstack[.]com</code> — typosquat staging domain</li>
</ul>

<p><strong>Filesystem persistence artifacts (survive npm uninstall):</strong></p>
<ul role="list">
<li>Claude Code hooks directory — check <code>~/.claude/</code> and project-level <code>.claude/</code> for unexpected hooks or modified <code>settings.json</code></li>
<li>VS Code auto-run tasks — check <code>.vscode/tasks.json</code> for unexpected <code>runOn: folderOpen</code> tasks, especially those referencing <code>setup.mjs</code>, <code>router_runtime.js</code>, or Bun</li>
<li><code>setup.mjs</code> and <code>router_runtime.js</code> in any path outside the expected package directory</li>
</ul>

<p><strong>Propagation artifacts:</strong></p>
<ul role="list">
<li><code>OhNoWhatsGoingOnWithGitHub</code> in GitHub commit messages (token dead-drop marker, carried from April wave)</li>
<li>GitHub repositories with description <em>"A Mini Shai-Hulud has Appeared"</em></li>

</ul>

<p><strong>Payload naming (Mistral AI PyPI):</strong></p>
<ul role="list">
<li><code>transformers.pyz</code> — Python zip payload; impersonates Hugging Face Transformers library</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Scan for IDE persistence immediately — this is not optional.</strong> Check <code>~/.claude/</code>, project-level <code>.claude/</code>, and all <code>.vscode/tasks.json</code> files for unexpected entries. The payload survives package removal; every compromised developer machine will still be running it until explicitly cleaned. Run: <code>grep -r "setup.mjs\|router_runtime\|getsession" ~/.claude/ .vscode/ 2>/dev/null</code></li>
<li><strong>Block all three C2 domains at DNS and proxy.</strong> <code>*.getsession.org</code>, <code>api.masscan.cloud</code>, <code>git-tanstack.com</code> — any connection to these from developer machines or CI/CD is active exfiltration in progress.</li>
<li><strong>Treat SLSA Level 3 attestations as incomplete defense for this attack class.</strong> Package provenance confirms origin workflow; it does not confirm workflow integrity. Until SLSA Level 4 (or equivalent build environment integrity guarantees) are implemented, add behavioral analysis at install time: alert on <code>preinstall</code> hooks that download Bun, spawn detached processes, or make outbound connections during <code>npm install</code>.</li>
<li><strong>Rotate all credentials from affected environments.</strong> Collection scope: GitHub Actions OIDC tokens, PATs, Git credentials, npm publish tokens, AWS (Secrets Manager, IAM, ESC task credentials), Kubernetes service accounts, HashiCorp Vault tokens, SSH keys, Claude Code configs, VS Code tasks, <code>.env</code> files.</li>
<li><strong>Audit <code>pull_request_target</code> workflows in your repositories.</strong> Any workflow that checks out PR code and runs it in the <code>pull_request_target</code> context with access to repository secrets is vulnerable to this attack class. The fix is to isolate PR code checkout from secret-access workflows, or use ephemeral runners with no persistent credential access.</li>
<li><strong>Enforce lockfile-only installs.</strong> <code>npm ci</code> (not <code>npm install</code>) prevents auto-resolution of new patch versions. This does not block intentionally published malicious versions, but it prevents silent adoption of worm-published versions outside your lockfile window.</li>
<li><strong>Check GitHub audit logs for <code>OhNoWhatsGoingOnWithGitHub</code>.</strong> Token dead-drop marker consistent across all Mini Shai-Hulud waves. Any match means prior-wave credentials are circulating in the worm's token pool.</li>
<li><strong>For compromised Mistral AI PyPI environments:</strong> Hunt for <code>transformers.pyz</code> in any Python environment that installed affected <code>@mistralai/mistralai</code> versions. This is the renamed Python payload — its presence confirms execution.</li>
</ul>

###### Attribution and cluster position

<p>This is the largest Mini Shai-Hulud wave to date by artifact count (373–416 malicious entries vs. the April 29 SAP CAP wave's four packages). The payload is unchanged at its core — Bun v1.3.13, <code>router_runtime.js</code>, <code>setup.mjs</code>, Russian locale exit — confirming the same codebase as the prior waves. New capabilities (Session P2P exfil, IDE persistence, conditional destructive routine) represent operational upgrades to an existing toolkit, not a new actor.</p>

<p>The SLSA Level 3 bypass is the most significant tactical evolution. It means the attack is now invisible to one of the most widely-recommended supply-chain defenses. Defenders who verified SLSA attestations on the TanStack packages would have seen valid signatures from the legitimate workflow — and still installed the malicious payload.</p>

<p>See the <a href="https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026">npm supply-chain worm cluster analysis</a> for the full cluster timeline, and <a href="https://keepsecure.io/hub/mini-shai-hulud-sap-npm-bun-worm-april-2026">Mini Shai-Hulud April 29</a> for the direct precursor event that established the worm's toolchain.</p>

###### Criminal-market signal

<p>No dark-web presence for Mini Shai-Hulud tooling or exfiltrated credentials has been observed. The operator-run credential-collection model confirmed across the entire TeamPCP/Mini Shai-Hulud cluster applies. The Session P2P exfil upgrade is consistent with an operator actively improving operational security — a commodity actor selling access would not invest in replacing GitHub dead-drop with a P2P network. The conditional destructive routine (Israel/Iran wipe) is not a criminal-market monetization; it is a geopolitically motivated destructive capability that further distinguishes this operator from financially motivated commodity actors. H2 operator-run pattern confirmed, with the destructive component as an additional indicator of non-commodity intent.</p>
