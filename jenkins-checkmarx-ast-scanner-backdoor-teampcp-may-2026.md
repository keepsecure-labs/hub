---
title: "Jenkins Checkmarx AST Scanner plugin backdoored (TeamPCP): trojanized JAR + runtime cli.js download from cx-plugins-releas staging repo"
date: 2026-05-10
heroImage: "/art/jenkins-checkmarx-ast-scanner-backdoor-teampcp-may-2026.png"
description: "Version 2026.5.09 of the official Jenkins Checkmarx AST Scanner plugin was replaced with a malicious build on May 9, 2026. The compromised version fetches a trojanized checkmarx-ast-scanner.jar and a runtime-downloaded obfuscated cli.js from an attacker-controlled GitHub repository labeled 'mini Shai-Hulud'. Exfiltration endpoint is audit.checkmarx[.]cx/v1/telemetry — shared C2 with the April 2026 Checkmarx KICS Docker/VS Code compromises. Persistence drops: .claude/router_runtime.js and .vscode/tasks.json."
slug: jenkins-checkmarx-ast-scanner-backdoor-teampcp-may-2026
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-05-09
cves: []
targetSectors:
  - Jenkins CI/CD infrastructure
  - DevSecOps tooling
  - Enterprise development pipelines
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - Jenkins
  - supply-chain
  - TeamPCP
  - Mini Shai-Hulud
  - Checkmarx
  - CI/CD
  - credential-theft
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "plugins.jenkins.io/checkmarx — version 2026.5.09 replaced with malicious build"
    c2_primary: "audit.checkmarx[.]cx/v1/telemetry (AES-256-GCM encrypted payloads)"
    c2_fallback: null
    exfil_channel: "HTTPS POST to audit.checkmarx[.]cx/v1/telemetry"
    c2_resilience_tier: 2
  capability:
    initial_access: compromised-plugin-registry-account
    execution: trojanized-jar-plus-runtime-downloaded-cli-js
    evasion: runtime-download-from-staging-repo-avoids-static-analysis
    persistence: ".claude/router_runtime.js; .vscode/tasks.json"
  victim:
    direct: "Jenkins instances running checkmarx-ast-scanner 2026.5.09 for SAST scan steps"
    blast_radius: "All CI/CD secrets, cloud credentials, and source code accessible to Jenkins build agents"
sources:
  - title: "Berk Albayrak (@brkalbyrk7): original disclosure"
    url: "https://xcancel.com/brkalbyrk7/status/2053175077194117590"
  - title: "Checkmarx: Ongoing security updates"
    url: "https://checkmarx.com/blog/ongoing-security-updates/"
  - title: "Datadog Security Research: Jenkins Checkmarx AST Scanner card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "Checkmarx KICS Docker/VS Code compromise — this hub"
    url: "https://keepsecure.io/hub/checkmarx-kics-docker-vscode-teampcp-april-2026"
---

<p>Version <code>2026.5.09</code> of the official Jenkins Checkmarx AST Scanner plugin was replaced with a malicious build on May 9, 2026. First reported by Adnan Khan and Berk Albayrak, the backdoored version fetches two additional artifacts into any Jenkins build that runs a Checkmarx scan step: a trojanized <code>checkmarx-ast-scanner.jar</code> bundled with the plugin, and an obfuscated <code>cli.js</code> downloaded at runtime from an attacker-controlled GitHub repository (<code>github.com/cx-plugins-releas</code>).</p>

<p>The attacker-controlled staging repository is labeled <em>"mini Shai-Hulud"</em> — explicit Dune campaign branding linking this to the TeamPCP cluster. The exfiltration endpoint is <code>audit.checkmarx[.]cx/v1/telemetry</code> with AES-256-GCM encrypted payloads — shared C2 infrastructure with the April 2026 Checkmarx KICS Docker and VS Code extension compromises. This is the same actor, expanding their foothold in the Checkmarx toolchain from developer IDEs to CI/CD pipeline scan steps.</p>

###### Attack chain

<p>The backdoor operates at scan execution time, not plugin install time:</p>

<ol>
<li>Jenkins CI job runs a Checkmarx AST scan step using the compromised plugin version</li>
<li>Plugin fetches <code>checkmarx-ast-scanner.jar</code> — the bundled trojanized version replaces the expected artifact</li>
<li>Plugin downloads <code>cli.js</code> at runtime from <code>raw.githubusercontent[.]com/cx-plugins-releas/...</code> — the runtime download means the malicious payload is not present in the plugin tarball and will not be detected by static package analysis of the plugin itself</li>
<li>Payload harvests secrets from the CI runner environment: GitHub tokens, AWS credentials, Azure and Google Cloud credentials, npm publish tokens, SSH keys, environment variables, MCP configuration files</li>
<li>Encrypted payload exfiltrated to <code>audit.checkmarx[.]cx/v1/telemetry</code></li>
</ol>

<p>The runtime-download pattern is operationally deliberate: it separates the delivery mechanism (the plugin) from the payload (cli.js), allowing the payload to be updated without publishing a new plugin version. It also defeats static analysis of the plugin tarball.</p>

###### Shared C2 infrastructure with KICS compromise

<p>The <code>audit.checkmarx[.]cx/v1/telemetry</code> exfiltration endpoint is identical to the C2 used in the April 22, 2026 Checkmarx KICS Docker image and VS Code extension compromise. The shared endpoint confirms common operator infrastructure across both campaigns. The attacker has maintained and reused this infrastructure from April through May 2026 — suggesting either high operator confidence in its continued operational security, or that the infrastructure is rotated less frequently than in other TeamPCP campaigns.</p>

###### Persistence drops

<p>Two persistence mechanisms are noted:</p>

<ul role="list">
<li><code>.claude/router_runtime.js</code> — Claude Code configuration directory injection, consistent with the Mini Shai-Hulud pattern of targeting AI coding assistant configurations for hook persistence</li>
<li><code>.vscode/tasks.json</code> — VS Code auto-run task injection, enabling payload re-execution on next IDE session open</li>
</ul>

<p>Both mechanisms survive <code>npm uninstall</code> or removal of the Jenkins plugin — the persistence is on the build agent's filesystem, not in the plugin itself.</p>

###### Indicators of compromise

<p><strong>Affected plugin version:</strong> <code>checkmarx-ast-scanner</code> <code>2026.5.09</code></p>

<p><strong>File hashes (SHA-256, reported by Albayrak):</strong></p>
<ul role="list">
<li><code>checkmarx-ast-scanner.jar</code>: <code>f50a96d26a5b0beb29de4127e82b2bf350c21511e5a43d286e43f798dc6cd53f</code></li>
<li><code>cli.js</code>: <code>08352b4c37808a25895cda1cae27ec8a83cf7ee9de15e2d4dd9560a2906730f4</code></li>
</ul>

<p><strong>Attacker staging repository:</strong> <code>github.com/cx-plugins-releas</code> (labeled "mini Shai-Hulud")</p>

<p><strong>Exfiltration endpoint (defanged):</strong> <code>audit.checkmarx[.]cx/v1/telemetry</code> — AES-256-GCM encrypted, <code>X-Vault-Token</code> or <code>Authorization: Bearer</code> header</p>

<p><strong>Persistence artifacts:</strong></p>
<ul role="list">
<li><code>.claude/router_runtime.js</code></li>
<li><code>.vscode/tasks.json</code></li>
</ul>

###### Detection

<ul role="list">
<li><strong>Check Jenkins plugin version.</strong> Navigate to Manage Jenkins → Manage Plugins → Installed. Version <code>2026.5.09</code> is the compromised build.</li>
<li><strong>Review build logs</strong> for jobs that invoked the Checkmarx AST Scanner step for unexpected downloads of <code>checkmarx-ast-scanner.jar</code> or <code>cli.js</code>.</li>
<li><strong>Search workspace and temp directories</strong> on Jenkins agents for files matching the JAR/JS SHA-256 hashes above.</li>
<li><strong>Audit DNS and proxy logs from Jenkins agents</strong> for connections to <code>audit.checkmarx[.]cx</code> or downloads from <code>raw.githubusercontent.com/cx-plugins-releas</code>.</li>
</ul>

<p>SIEM query for the shared C2:</p>
<pre><code>@http.url_details.host:audit.checkmarx.cx OR @dns.question.name:audit.checkmarx.cx</code></pre>

###### Remediation

<ul role="list">
<li><strong>Disable or remove the plugin immediately.</strong> Do not execute build jobs with version <code>2026.5.09</code> installed. Wait for Checkmarx to publish a verified clean release.</li>
<li><strong>Rotate all secrets accessible to affected Jenkins agents:</strong> GitHub tokens, AWS credentials, Azure and Google Cloud credentials, npm publish tokens, SSH keys, Jenkins credential store values, MCP configuration material.</li>
<li><strong>Remove persistence artifacts</strong> (<code>.claude/router_runtime.js</code>, <code>.vscode/tasks.json</code>) from any agent filesystems where the plugin ran.</li>
<li><strong>Audit GitHub audit logs and Actions runs</strong> for the Shai-Hulud exfiltration pattern (unexpected repository creation, <code>${{ toJSON(secrets) }}</code> workflow injection).</li>
<li><strong>Restrict Jenkins agent egress</strong> to approved endpoints. Block downloads from arbitrary GitHub raw content URLs as defense-in-depth.</li>
</ul>

###### Criminal-market signal

<p>No dark-web presence for Jenkins Checkmarx AST Scanner payload tooling or <code>audit.checkmarx.cx</code> infrastructure has been observed. The TeamPCP operator-run pattern confirmed across the cluster applies. H2 (operator-run, no commodity market) is confirmed by the shared C2 infrastructure across multiple campaign phases.</p>
