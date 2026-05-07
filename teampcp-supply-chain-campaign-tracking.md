---
title: "Team PCP: tracking a six-week supply-chain campaign through Trivy, Checkmarx, Bitwarden, and beyond"
date: 2026-04-30
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451ebf03b6b874b09745aa_blog%202.svg"
description: "A self-spreading credential-theft campaign that has chained through six security-tooling vendors since March 2026. Patterns, IOCs, and detection guidance."
slug: teampcp-supply-chain-campaign-tracking
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-03-20
targetSectors:
  - Software development
  - Security tooling
  - Cloud infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - supply chain
  - GitHub Actions
  - npm
  - PyPI
  - Team PCP
  - CanisterWorm
sources:
  - title: "TeamPCP attack on KICS GitHub Action — Wiz"
    url: "https://www.wiz.io/blog/teampcp-attack-kics-github-action"
  - title: "Trivy under attack again — Socket"
    url: "https://socket.dev/blog/trivy-under-attack-again-github-actions-compromise"
  - title: "Trivy compromised a second time — StepSecurity"
    url: "https://www.stepsecurity.io/blog/trivy-compromised-a-second-time---malicious-v0-69-4-release"
  - title: "TeamPCP deploys worm via npm and Trivy compromise — Aikido"
    url: "https://www.aikido.dev/blog/teampcp-deploys-worm-npm-trivy-compromise"
  - title: "TeamPCP and aquasec.com GitHub org compromise — Open Source Malware"
    url: "https://opensourcemalware.com/blog/teampcp-aquasec-com-github-org-compromise"
  - title: "Checkmarx supply chain compromise — Socket"
    url: "https://socket.dev/blog/checkmarx-supply-chain-compromise"
  - title: "Bitwarden CLI compromised — Socket"
    url: "https://socket.dev/blog/bitwarden-cli-compromised"
  - title: "Bitwarden statement on Checkmarx supply chain incident"
    url: "https://community.bitwarden.com/t/bitwarden-statement-on-checkmarx-supply-chain-incident/96127"
  - title: "Xinference compromise — JFrog"
    url: "https://research.jfrog.com/post/xinference-compromise/"
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "GitHub Actions tag force-update; npm/PyPI/OpenVSX/Docker Hub publish accounts"
    c2_primary: "checkmarx[.]zone / audit.checkmarx[.]cx"
    c2_fallback: "championships-peoples-point-cassette.trycloudflare[.]com (Cloudflare Tunnel)"
    exfil_channel: "ICP canister tdtqy-oyaaa-aaaae-af2dq-cai.raw.icp0[.]io; public GitHub repos tpcp-docs/docs-tpcp"
    c2_resilience_tier: 3
  capability:
    initial_access: ci-cd-injection
    execution: postinstall-hook
    evasion: bun-runtime-loader
    persistence: systemd-user-unit
  victim:
    direct: "Aqua Security (Trivy), Checkmarx (KICS + extensions), Bitwarden (CLI), Xinference, litellm"
    blast_radius: "All CI/CD pipelines using affected GitHub Actions by version tag (not SHA)"
---

<p>Since March 20, 2026, a credential-theft operation tracked under the campaign name <strong>Team PCP</strong> has chained through six security-tooling vendors and at least nine published packages across npm, PyPI, Docker Hub, OpenVSX, and the GitHub Actions marketplace. Each compromise feeds the next: stolen CI/CD credentials are used to publish trojanized versions of downstream packages, whose installations in turn yield more credentials. The campaign's distinguishing feature is its <strong>meta-targeting</strong> — the victims are mostly the security industry's own supply-chain scanning tools (Trivy, Checkmarx KICS, Aqua Security's GitHub org, Bitwarden's release pipeline). The operator infrastructure has self-named a worm component "CanisterWorm" and uses Cloudflare Tunnel and Internet Computer Protocol canisters as fallback C2 channels. This piece consolidates the public reporting from Wiz, Socket, StepSecurity, Aikido, Sysdig, JFrog, and Open Source Malware into one campaign timeline and one detection playbook.</p>

###### What we know

<p>The campaign's confirmed timeline, in chronological order:</p>

<ul role="list">
<li><strong>March 20, 2026</strong> — <code>aquasecurity/trivy-action</code>, <code>aquasecurity/setup-trivy</code>, and a malicious Trivy <code>v0.69.4</code> release. Many existing tags force-updated to malicious commits. Per StepSecurity, credential-stealing logic injected into <code>action.yaml</code> at commit <code>8afa9b9</code>; clean tag is <code>v0.2.6</code> aligned with <code>3fb12ec</code>. Exfiltration to <code>scan.aquasecurtiy[.]org</code> (typosquat of aquasecurity.org).</li>
<li><strong>March 23, 2026</strong> — Aqua Security's internal GitHub org defaced. CanisterWorm propagates via stolen tokens. C2 domain rotation begins. Open Source Malware and Aikido publish IOC sets.</li>
<li><strong>March 23, 2026 (12:58–16:50 UTC)</strong> — Checkmarx <code>kics-github-action</code> compromised via imposter commits and tag hijacking. New C2: <code>checkmarx[.]zone</code>. Kubernetes-oriented persistence added. Wiz attributes to Team PCP based on overlapping TTPs and infrastructure.</li>
<li><strong>March 23, 2026</strong> — <code>ast-github-action</code> tag <code>2.3.28</code> observed malicious (Sysdig). OpenVSX <code>ast-results</code> 2.53.0 and <code>cx-dev-assist</code> 1.7.0 published via the <code>ast-phoenix</code> account on Open VSX. VS Code Marketplace versions described as unaffected.</li>
<li><strong>March 24, 2026</strong> — <code>litellm</code> on PyPI (versions 1.82.7 and 1.82.8). The vector cited is the compromised Trivy GitHub Action stealing PyPI publishing credentials from litellm's CI/CD pipeline. Malicious <code>litellm_init.pth</code> file means execution at Python interpreter startup, no explicit import required.</li>
<li><strong>April 22, 2026</strong> — <code>xinference</code> on PyPI (versions 2.6.0, 2.6.1, 2.6.2). JFrog identifies the marker string <code># hacked by teampcp</code> in decoded payload. Exfiltration to <code>whereisitat[.]lucyatemysuperbox[.]space</code>.</li>
<li><strong>April 22, 2026</strong> — Checkmarx KICS Docker images at <code>checkmarx/kics</code> trojanized; tags <code>v2.1.20</code>, <code>v2.1.20-debian</code>, <code>debian</code>, <code>alpine</code>, <code>latest</code> overwritten; fake <code>v2.1.21</code> tag published. Checkmarx VS Code and OpenVSX extensions <code>cx-dev-assist</code> 1.17.0/1.19.0 and <code>ast-results</code> 2.63.0/2.66.0 published with hidden MCP-addon feature that downloads <code>mcpAddon.js</code> from a backdated orphaned commit (<code>68ed490b</code>) inside the legitimate <code>Checkmarx/ast-vscode-extension</code> repository.</li>
<li><strong>April 22, 2026 (5:57 PM – 7:30 PM ET)</strong> — <code>@bitwarden/cli@2026.4.0</code> published to npm with bundled <code>bw1.js</code> second-stage payload. Bitwarden confirms abuse of a GitHub Action in its CI/CD pipeline; package pulled within roughly 90 minutes of detection. Bitwarden's public statement reports no end-user vault data accessed.</li>
</ul>

<blockquote>The campaign is reflexive: the tools that scan supply chains for compromise are themselves the supply-chain compromises. Trivy → Aqua's GitHub org → Checkmarx's KICS → Bitwarden's release pipeline. Each victim was a vendor whose product is supposed to detect this kind of attack.</blockquote>

###### Targeting

<ul role="list">
<li><strong>Security tooling vendors</strong> — disproportionate selection. Trivy (Aqua Security), Checkmarx KICS, Bitwarden's release infrastructure. The pattern compromises the vendor whose CI/CD pipeline has the credentials to publish artifacts that downstream defenders will install and trust.</li>
<li><strong>CI/CD pipelines using affected GitHub Actions</strong> — pinning by version tag rather than by SHA exposed many victims, since tag force-updates pointed existing references at malicious commits.</li>
<li><strong>Developer workstations</strong> — broad credential collection on non-CI hosts, with systemd-user persistence on Linux (per Wiz, polling <code>https://checkmarx[.]zone/raw</code>).</li>
<li><strong>Kubernetes environments</strong> — provisioner-style persistence using pod names <code>host-provisioner-std</code> and <code>host-provisioner-iran</code>, container names <code>provisioner</code> and <code>kamikaze</code>.</li>
</ul>

###### TTPs and infrastructure

<ul role="list">
<li><strong>Initial access</strong> — credential theft from compromised CI/CD runners running affected GitHub Actions, then re-use of those credentials to publish trojanized versions of downstream packages. Self-spreading through the chain.</li>
<li><strong>Execution</strong> — multiple delivery mechanisms across the campaign: GitHub Actions <code>action.yaml</code> injection, malicious <code>setup.sh</code> in imposter commits, <code>preinstall</code> hooks in npm packages, <code>__init__.py</code> and <code>.pth</code> files on PyPI, hidden "MCP addon" features in VS Code and OpenVSX extensions that fetch second stages via Bun runtime.</li>
<li><strong>Persistence</strong> — systemd user units polling C2 (<code>checkmarx[.]zone/raw</code>); Kubernetes provisioner pods.</li>
<li><strong>Defense evasion</strong> — backdated orphaned commits referenced by hardcoded URL but not visible in active branch history; <code>__decodeScrambled</code> obfuscation with seed <code>0x3039</code>; Bun runtime as second stage to bypass Node-focused static analysis (shared TTP with the Mini Shai-Hulud cluster — see <a href="https://keepsecure.io/hub/bun-runtime-supply-chain-stealer-april-2026">the Bun-runtime supply-chain analysis</a>).</li>
<li><strong>Command and control</strong> — tiered. Primary C2 at custom domains: <code>scan.aquasecurtiy[.]org</code> (Trivy phase), <code>checkmarx[.]zone</code> (KICS / extensions phase), <code>audit.checkmarx[.]cx/v1/telemetry</code> (Checkmarx Docker / Bitwarden phase). Fallback to Cloudflare Tunnel domains (<code>*.trycloudflare[.]com</code>) and an Internet Computer canister (<code>tdtqy-oyaaa-aaaae-af2dq-cai.raw.icp0[.]io</code>). When direct C2 is disrupted, the worm creates public repositories (<code>tpcp-docs</code>, <code>docs-tpcp</code>) on victims' GitHub accounts using <code>GITHUB_TOKEN</code> and uploads stolen material as release assets.</li>
<li><strong>Lateral movement</strong> — CanisterWorm: stolen credentials from one victim used to publish trojanized versions of downstream packages, whose installations on new victims yield more credentials.</li>
</ul>

###### Indicators of compromise

<p>Consolidated from Wiz, Socket, StepSecurity, Aikido, Sysdig, JFrog, and Datadog Security Research:</p>

<p><strong>Network — C2 infrastructure:</strong></p>

<ul role="list">
<li><code>scan.aquasecurtiy[.]org</code> — Trivy phase typosquat (note transposed <code>i</code> and <code>t</code>)</li>
<li><code>aquasecurtiy[.]org</code> — base typosquat</li>
<li><code>checkmarx[.]zone</code> — KICS / extensions phase</li>
<li><code>audit.checkmarx[.]cx/v1/telemetry</code> — Checkmarx Docker / Bitwarden phase</li>
<li><code>whereisitat[.]lucyatemysuperbox[.]space</code> — Xinference phase</li>
<li><code>tdtqy-oyaaa-aaaae-af2dq-cai.raw.icp0[.]io</code> — ICP canister fallback</li>
<li><code>championships-peoples-point-cassette.trycloudflare[.]com</code> — Cloudflare tunnel</li>
<li><code>investigation-launches-hearings-copying.trycloudflare[.]com</code> — Cloudflare tunnel</li>
<li><code>souls-entire-defined-routes.trycloudflare[.]com</code> — Cloudflare tunnel</li>
<li><code>83.142.209.11</code> — direct IP (KICS phase)</li>
</ul>

<p><strong>Filesystem and behavioral artifacts:</strong></p>

<ul role="list">
<li><code>/tmp/pglog</code> — CanisterWorm payload drop path</li>
<li>Pod names containing <code>host-provisioner-std</code> or <code>host-provisioner-iran</code></li>
<li>Container names <code>kamikaze</code> or <code>provisioner</code></li>
<li>Public GitHub repositories named <code>tpcp-docs</code> or <code>docs-tpcp</code> on victim accounts, with stolen material as release assets</li>
<li>Marker string <code># hacked by teampcp</code> in decoded payloads</li>
</ul>

<p><strong>VirusTotal hash (Trivy phase):</strong> <code>18a24f83e807479438dcab7a1804c51a00dafc1d526698a66e0640d1e5dd671a</code></p>

<p><strong>Xinference SHA-256s (JFrog):</strong></p>

<ul role="list">
<li><code>xinference/__init__.py</code> — <code>e1e007ce4eab7774785617179d1c01a9381ae83abfd431aae8dba6f82d3ac127</code></li>
<li>Decoded stage 1 — <code>077d49fa708f498969d7cdffe701eb64675baaa4968ded9bd97a4936dd56c21c</code></li>
<li>Decoded stage 2 — <code>fe17e2ea4012d07d90ecb7793c1b0593a6138d25a9393192263e751660ec3cd0</code></li>
</ul>

<p><strong>GitHub identities used to publish malicious tags:</strong></p>

<ul role="list">
<li><code>cx-plugins-releases</code> (account ID <code>225848595</code>) — KICS phase</li>
<li><code>ast-phoenix</code> — OpenVSX extensions</li>
</ul>

###### Detection and mitigation

<p>For environments that may have run any affected version, treat compromise as the working hypothesis until proven otherwise. The campaign harvests broadly and exfiltrates immediately.</p>

<ul role="list">
<li><strong>Pin GitHub Actions by SHA, not by version tag.</strong> The Trivy and KICS compromises both relied on tag force-update — workflows that pinned to <code>@v3</code> or similar received the malicious commit when the tag moved. SHA pinning is the structural fix that survives tag-hijack.</li>
<li><strong>Audit GitHub audit logs</strong> for unusual create/delete branch sequences from service accounts (TeamPCP rapidly created and deleted branches to test stolen tokens), unfamiliar repository creation, and any commits to <code>tpcp-docs</code> or <code>docs-tpcp</code> repositories.</li>
<li><strong>Block egress</strong> to the listed C2 domains at the network perimeter and on agent hosts. The Cloudflare Tunnel and ICP-canister fallbacks are harder to block without breaking legitimate traffic, but the primary domains and the typosquat are clean blocks.</li>
<li><strong>Hunt for the systemd user-unit persistence</strong> on Linux developer hosts and CI workers that touched any affected version: any user-unit polling <code>checkmarx[.]zone/raw</code> is the persistence mechanism.</li>
<li><strong>Rotate everything reachable</strong> from any host that ran an affected version: GitHub PATs, npm publish tokens, AWS / Azure / GCP credentials, Kubernetes service account tokens, SSH keys, signing material, secrets in non-sensitive environment variables. The malware enumerates broadly; assume everything is exfiltrated.</li>
<li><strong>Ban Bun fetches from CI runners</strong> that don't legitimately use Bun. Outbound to <code>github.com/oven-sh/bun/releases</code> from a runner mid-install is a strong adversary signal across the Bun-runtime variant of the payload.</li>
<li><strong>Treat security tooling like any other supply chain.</strong> The same review hygiene that protects against generic npm/PyPI compromise — lockfile pinning, <code>--ignore-scripts</code> by default, signed releases, provenance attestation — applies to the security industry's own tools.</li>
</ul>

###### Attribution discipline

<p>"Team PCP" is a tracking name applied independently by Wiz, Sysdig, Aikido, Open Source Malware, and Datadog Security Research based on overlapping TTPs and infrastructure across the campaign chain. The marker string <code># hacked by teampcp</code> appears in payloads, and the campaign's worm component self-labels as CanisterWorm. Third-party reporting has associated related activity with the aliases <strong>DeadCatx3</strong>, <strong>PCPcat</strong>, <strong>ShellForce</strong>, and <strong>CanisterWorm</strong>; these are self-applied labels in payload material and onion-site claims, not independent confirmation of actor identity. The Mini Shai-Hulud npm cluster of April 29-30 (Lightning PyPI + SAP CAP npm packages) shows tooling overlap; whether it's the same operator is consistent with available reporting but not confirmed.</p>

<p>We use the campaign codename Team PCP throughout this writeup. We do not claim hard attribution to a specific country, group, or named individual. Defenders can act on the IOCs and TTPs without needing attribution to be settled.</p>

###### What this signals for 2026

<p>Three durable observations from the chain so far:</p>

<ul role="list">
<li><strong>Self-spreading credential theft scales.</strong> The CanisterWorm pattern — stolen credentials used to publish trojanized downstream packages — converts each successful compromise into multiple new victims without operator effort. As long as security tooling pipelines hold publishing credentials with broad scope, this pattern continues.</li>
<li><strong>Security tooling is high-value target.</strong> Six weeks of repeated targeting against Trivy, Checkmarx, and Bitwarden is not coincidence. The vendors whose products defenders trust to detect supply-chain compromise are the highest-leverage victims. Treat your scanner-vendor's release pipeline with the same rigor you'd apply to any production supply chain.</li>
<li><strong>Tag pinning is over.</strong> GitHub Action consumers who pin by version tag get whatever the maintainer (or the maintainer's compromised account) currently points the tag at. Two separate vendors got force-updated within five weeks. SHA pinning is the structural answer; allowlisted action versions in policy-as-code is the second-best.</li>
</ul>

<p>The closest peer pattern in the recent CVE record is <a href="https://keepsecure.io/hub/cve-2026-12091-npm-postinstall-maintainer-takeover">CVE-2026-12091</a> — npm maintainer-account takeover with postinstall credential theft. Same shape, different vector. The <a href="https://keepsecure.io/hub/copyfail-time-to-criminalization-seven-days">time-to-criminalization framework</a> applied to that CVE shows what to expect from this class of bug: the implant work is already done, so commoditization happens on the order of days rather than months. Patch and rotate accordingly.</p>

<p>Final observation on the threat-intelligence shape. Despite Team PCP's loud operational footprint — six weeks of researcher coverage, public IOCs in widely-shared formats, named C2 domains — the campaign has no measurable presence in commodity criminal markets. There's no broker pricing for the worm, no exploit-kit packaging of the GitHub Actions injection technique, no forum trade volume. That isn't a sensor failure. It's the campaign's actual shape: Team PCP is the operator, not a vendor selling capability. The credentials they exfiltrate may eventually appear in stealer-log markets, but the attack itself doesn't commoditize. <a href="https://keepsecure.io/hub/ai-ide-marketplace-security-telemetry">Same telemetry-surface lesson as the AI-IDE marketplace surface</a>: monitor the legitimate channel where the attack actually lives — package registries, GitHub Actions tag history, OAuth grants — not .onion forums where it doesn't.</p>
