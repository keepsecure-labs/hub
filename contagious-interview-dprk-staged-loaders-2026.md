---
title: "Contagious Interview goes cross-ecosystem: DPRK staged loaders hit npm, PyPI, Go, Rust, and PHP"
date: 2026-04-28
heroImage: "/art/contagious-interview-dprk-staged-loaders-2026.png"
description: "North Korea's Contagious Interview operation spread a staged loader cluster across five package ecosystems simultaneously. The Windows build of one package includes a full post-compromise toolkit and AnyDesk-based remote access."
slug: contagious-interview-dprk-staged-loaders-2026
type: threat-intel
campaign: "Contagious Interview"
firstObserved: 2026-04-01
targetSectors:
  - Software development
  - AI / ML
  - Blockchain / cryptocurrency
  - Financial technology
targetRegions:
  - Global
  - North America
  - Europe
  - East Asia
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - DPRK
  - npm
  - PyPI
  - Go
  - crates.io
  - Packagist
  - supply chain
  - staged loader
  - BeaverTail
  - social engineering
hasArtifacts: false
sources:
  - title: "Contagious Interview Campaign Spreads Staged Malware Loaders Across Five Package Ecosystems — Socket"
    url: "https://socket.dev/blog/contagious-interview-campaign-spreads-across-5-ecosystems"
  - title: "Datadog Security Research: Contagious Interview campaign spreads staged malware loaders"
    url: "https://app.datadoghq.com/security/feed"
diamond:
  adversary:
    operator_model: operator-run
    monetization: cryptocurrency-theft
    confidence: attributed
  infrastructure:
    registry_layer: "npm, PyPI, Go Modules, crates.io, Packagist — golangorg/aokisasakidev/aokisasakidev1 aliases"
    c2_primary: "apachelicense[.]vercel[.]app / logkit[.]onrender[.]com / logkit-tau[.]vercel[.]app"
    c2_fallback: "66[.]45[.]225[.]94 (direct IP, license-utils-kit Windows variant)"
    exfil_channel: "Google Drive staging → ecw_update.zip → platform-specific binary; AnyDesk remote access"
    c2_resilience_tier: 1
  capability:
    initial_access: social-engineering
    execution: staged-loader
    evasion: benign-function-names
    persistence: anydesk-remote-access
  victim:
    direct: "Software developers targeted via fake recruiter job interviews"
    blast_radius: "dev-log-core, pino-debugger, debug-fmt, debug-glitz, logutilkit, apachelicense, fluxhttp, license-utils-kit, logtrace, golangorg/logkit — all ecosystems"
---

<p>The Contagious Interview operation — a long-running North Korean campaign that sends fake recruiter messages to software developers and hands them a "technical test" that is actually malware — has crossed a threshold. A newly reported package cluster, first identified by Socket, spreads a coordinated set of staged loaders across npm, PyPI, Go Modules, crates.io, and Packagist simultaneously. Publishing to five ecosystems under coordinated GitHub aliases is not an accident of tooling. It is a deliberate coverage strategy: a developer who reaches one malicious package on one registry and quarantines it may install a sibling package on a second registry without knowing they're pulling from the same campaign infrastructure. The Windows build of one package in this cluster — <code>license-utils-kit</code> on PyPI — drops a full post-compromise toolkit including shell access, keylogging, browser credential extraction, and AnyDesk-based persistent remote access. The thin-loader stage has hands-on intrusion capability attached.</p>

###### Background: what Contagious Interview is

<p>Contagious Interview (also tracked as FriendlyFerret, CL-STA-0240, and loosely associated with the Lazarus Group's IT-worker sub-cluster) has been operating since at least 2023. The campaign's social-engineering vector is consistent: a North Korean operator poses as a recruiter on LinkedIn, GitHub, or freelance platforms; initiates a job interview; and at some point asks the developer to clone a repository and run code locally as part of a "technical assessment" or "coding challenge." The code is the malware.</p>

<p>The payload family has evolved across three generations:</p>

<ul role="list">
<li><strong>BeaverTail</strong> — a JavaScript-based infostealer typically delivered as a trojanized npm package or as a fake Electron app. Targets browser credential stores, crypto wallet files, SSH keys, and cloud credentials. Has been found masquerading as video conferencing utilities and coding environment helpers.</li>
<li><strong>InvisibleFerret</strong> — a Python-based backdoor deployed as BeaverTail's second stage. Provides command execution, file upload/download, keylogging, and clipboard capture. Persists via cron, LaunchAgent (macOS), or startup registry entries.</li>
<li><strong>OtterCookie</strong> — a newer JavaScript-based payload observed from late 2024 onward, delivered via npm packages. Establishes a Socket.IO-based C2 channel and uses a JSON Web Token shared secret for authentication. Designed for persistent command-and-control rather than one-time credential theft.</li>
</ul>

<p>The 2026 cluster reported by Socket extends this pattern into four additional ecosystems (Go, Rust, PHP, PyPI) while maintaining npm as the JavaScript delivery surface. The multi-ecosystem reach is the operational evolution.</p>

###### The 2026 cluster: packages and personas

<p>Socket identified packages published under three coordinated GitHub aliases — <code>golangorg</code>, <code>aokisasakidev</code>, and <code>aokisasakidev1</code> — with a fourth alias (<code>maxcointech1010</code>) used for a GitHub account populated with cloned repositories across AI, blockchain, and developer tooling themes. The cloned repositories appear designed to lend the alias credibility rather than to publish packages directly.</p>

<p><strong>npm:</strong></p>
<ul role="list">
<li><code>dev-log-core</code></li>
<li><code>pino-debugger</code></li>
<li><code>debug-fmt</code></li>
<li><code>debug-glitz</code></li>
<li><code>logger-base</code> — dormant/sleeper at time of analysis</li>
<li><code>logkitx</code> — dormant/sleeper at time of analysis</li>
</ul>

<p><strong>PyPI:</strong></p>
<ul role="list">
<li><code>logutilkit</code></li>
<li><code>apachelicense</code></li>
<li><code>fluxhttp</code></li>
<li><code>license-utils-kit</code></li>
</ul>

<p><strong>Go Modules:</strong></p>
<ul role="list">
<li><code>github.com/golangorg/formstash</code></li>
<li><code>github.com/aokisasakidev/mit-license-pkg</code></li>
</ul>

<p><strong>Rust (crates.io):</strong></p>
<ul role="list">
<li><code>logtrace</code></li>
</ul>

<p><strong>PHP (Packagist):</strong></p>
<ul role="list">
<li><code>golangorg/logkit</code></li>
</ul>

###### Payload mechanics

<p>The Python, Go, Rust, and PHP branches share a common staged-loader workflow:</p>

<ol>
<li>Call out to attacker-controlled staging infrastructure to retrieve a payload URL.</li>
<li>Convert any Google Drive share link to a direct-download URL.</li>
<li>Pull a ZIP archive — typically named <code>ecw_update.zip</code> — and extract it into a hardcoded temporary directory named <code>410BB449A-72C6-4500-9765-ACD04JBV827V32V</code>.</li>
<li>Execute a platform-specific second-stage binary: <code>systemd-resolved</code> on Linux (name impersonates the systemd resolver), <code>com.apple.systemevents</code> on macOS (name impersonates an Apple process), and <code>py.exe</code> on Windows.</li>
</ol>

<p>The trigger functions — <code>log()</code>, <code>find_by_key()</code>, <code>trace()</code>, <code>CheckForUpdates()</code> — are named to look like ordinary library behavior. Reviewers who skim the package's public API surface see plausible utility code. The malicious staging call is embedded in these methods.</p>

<p>The npm branch works differently. Rather than staging a ZIP to disk, <code>dev-log-core</code> sends a <code>POST</code> request to a remote endpoint, base64-decodes the response body, and executes it via <code>new Function(require, decodedCode)(require)</code>. This technique — server-supplied arbitrary JavaScript evaluated inside the host Node.js process — has no local artifact to scan. No file is written before the payload executes. The entire second stage exists only in the HTTP response and in memory.</p>

<blockquote>The <code>new Function()</code> eval pattern is designed to defeat local artifact scanning. The payload isn't in the package tarball; it's fetched fresh from the operator's server at execution time. This means a clean hash on the installed package is not evidence of safety — if the C2 server is live and returning a payload, the package is weaponized regardless of what the tarball contains.</blockquote>

###### license-utils-kit: the full-capability variant

<p><code>license-utils-kit</code> on PyPI stands apart from the rest of the cluster. On Linux and macOS it behaves as a standard staged loader (Google Drive staging, <code>ecw_update.zip</code>, platform binary). On Windows it bundles a substantially more capable implant:</p>

<ul role="list">
<li><strong>Shell access</strong> — remote command execution</li>
<li><strong>Keylogging</strong> — keystroke capture</li>
<li><strong>Browser credential and session extraction</strong> — targeting Chromium-family browsers</li>
<li><strong>Cryptocurrency wallet extraction</strong> — targeting common desktop wallet files</li>
<li><strong>File harvesting and encrypted exfiltration archives</strong></li>
<li><strong>AnyDesk-based persistent remote access</strong> — leveraging the legitimate AnyDesk remote desktop client as a persistence and access mechanism to avoid triggering "unknown binary" defenses</li>
<li><strong>Separate C2 channel</strong> — direct connection to <code>66[.]45[.]225[.]94</code>, bypassing the staging-infrastructure indirection used by other packages in the cluster</li>
</ul>

<p>The Windows build effectively combines the BeaverTail credential-theft profile with an InvisibleFerret-style backdoor and AnyDesk-based remote-operator access into a single package. The social-engineering implication is significant: a Windows developer who installs this as part of a "technical assessment" hands the operator a persistent remote access session, not just a one-time credential dump.</p>

###### Social-engineering vector

<p>Contagious Interview's delivery mechanism is distinct from the supply-chain attacks documented in the TeamPCP and Shai-Hulud clusters. Those campaigns compromise existing trusted packages and wait for downstream consumers to install them passively. Contagious Interview <em>recruits</em> the victim: the developer is approached via LinkedIn or a freelance platform, cultivated over multiple messages, and eventually directed to clone a repository or install a package as part of an interview process.</p>

<p>This means the campaign specifically targets developers who are actively job-seeking or open to freelance work — a population that will accept unsolicited package-installation requests that a non-interview context would make suspicious. The technical-test framing normalizes running unfamiliar code locally. Developers who would never run <code>npm install &lt;unknown-package&gt;</code> in production may do exactly that when told it's a screening exercise from a recruiter.</p>

<p>Common recruitment signals associated with this campaign:</p>

<ul role="list">
<li>Recruiter messages referencing AI, blockchain, fintech, or crypto projects — sectors that match the fake-repository themes used by <code>maxcointech1010</code></li>
<li>Requests to "run a quick test project" or "review this demo repo" as part of the interview process, especially if the request arrives on a timeline that discourages careful review</li>
<li>GitHub profiles with many cloned or forked repositories across AI/crypto/ML themes but no meaningful commit history to any of them</li>
<li>Contact originating from platforms where identity verification is weak (LinkedIn, GitHub, freelance marketplaces)</li>
</ul>

###### Indicators of compromise

<p><strong>Malicious packages (all ecosystems):</strong></p>
<ul role="list">
<li>npm: <code>dev-log-core</code>, <code>pino-debugger</code>, <code>debug-fmt</code>, <code>debug-glitz</code>, <code>logger-base</code>, <code>logkitx</code></li>
<li>PyPI: <code>logutilkit</code>, <code>apachelicense</code>, <code>fluxhttp</code>, <code>license-utils-kit</code></li>
<li>Go: <code>github.com/golangorg/formstash</code>, <code>github.com/aokisasakidev/mit-license-pkg</code></li>
<li>crates.io: <code>logtrace</code></li>
<li>Packagist: <code>golangorg/logkit</code></li>
</ul>

<p><strong>Filesystem artifacts:</strong></p>
<ul role="list">
<li><code>ecw_update.zip</code> in any temporary directory</li>
<li>Directory named <code>410BB449A-72C6-4500-9765-ACD04JBV827V32V</code></li>
<li>Files named <code>start.py</code>, <code>systemd-resolved</code> (in non-systemd paths), <code>com.apple.systemevents</code> (in non-Apple paths), or <code>py.exe</code></li>
<li>AnyDesk installed on a developer workstation that the user did not install</li>
</ul>

<p><strong>Network indicators (defanged):</strong></p>
<ul role="list">
<li><code>apachelicense[.]vercel[.]app</code></li>
<li><code>ngrok-free[.]vercel[.]app</code></li>
<li><code>logkit[.]onrender[.]com</code></li>
<li><code>logkit-tau[.]vercel[.]app</code></li>
<li><code>66[.]45[.]225[.]94</code> — direct C2 IP (<code>license-utils-kit</code> Windows variant)</li>
<li>Suspicious Google Drive direct-download patterns from non-browser processes</li>
</ul>

<p><strong>Behavioral signals:</strong></p>
<ul role="list">
<li>Outbound HTTP to Vercel or Render endpoints from a process spawned during <code>npm install</code> or <code>pip install</code></li>
<li>ZIP archive download and extraction into a randomized directory name during package install</li>
<li><code>new Function()</code> execution pattern in Node.js processes spawned by npm lifecycle hooks</li>
<li>AnyDesk process running without corresponding user-initiated installation</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Audit all five ecosystems in your dependency trees.</strong> Check <code>package.json</code> / lockfiles, <code>requirements.txt</code> / <code>poetry.lock</code>, <code>go.mod</code> / <code>go.sum</code>, <code>Cargo.toml</code> / <code>Cargo.lock</code>, and <code>composer.json</code> for any of the listed package names. Recent build history matters — a package may have been removed from the current tree but was present in a prior build that ran on a CI runner.</li>
<li><strong>Treat any host that installed these packages as compromised.</strong> Rotate cloud credentials (AWS/Azure/GCP), GitHub tokens, npm and PyPI publish tokens, SSH keys, Kubernetes service account tokens, and any secrets in environment variables. The malware enumerates broadly; assume everything reachable from the host was exfiltrated.</li>
<li><strong>Search for filesystem artifacts</strong> listed above, particularly <code>ecw_update.zip</code> and the hardcoded extraction directory. The presence of either confirms payload execution.</li>
<li><strong>Block the listed C2 endpoints</strong> at the network perimeter. The direct IP <code>66[.]45[.]225[.]94</code> and the Vercel/Render staging domains are clean blocks that don't risk breaking legitimate traffic.</li>
<li><strong>Enable npm install script blocking by default in CI</strong> (<code>npm ci --ignore-scripts</code>). Allowlist packages that genuinely require lifecycle hooks. The <code>dev-log-core</code> postinstall path is blocked entirely by this flag.</li>
<li><strong>Educate developers on the recruitment vector.</strong> The social-engineering approach means developer awareness is a genuine control here — unusual for supply-chain attacks. A developer who knows to treat "run this test repo" as a red flag reduces the campaign's effective surface.</li>
<li><strong>Investigate unauthorized AnyDesk installations.</strong> On Windows, the presence of AnyDesk installed without user action on a developer workstation — especially one that recently ran a job-interview coding test — should be treated as evidence of active intrusion, not just initial compromise.</li>
</ul>

###### Attribution

<p>Socket attributes this cluster to North Korea's Contagious Interview operation based on TTP continuity: the staged Google Drive loader pattern, the BeaverTail/InvisibleFerret toolchain, the AI/crypto/developer-tooling cover personas, and the social-engineering recruitment vector. These are consistent with activity CrowdStrike tracks as Famous Chollima, Mandiant tracks under UNC4736/UNC2970, and the broader Lazarus Group IT-worker umbrella. The multi-ecosystem reach represents a tactical evolution but not a change in underlying infrastructure or targeting logic.</p>

<p>Attribution to North Korea in this context is well-supported: FBI and CISA have issued multiple advisories attributing Contagious Interview to DPRK-affiliated IT workers. The operational goal is currency generation — stolen credentials and persistent access convert into cryptocurrency theft and market manipulation. Developer workstations and CI/CD runners are the preferred targets because they hold both the credentials to access financial platforms and the ability to push code to production.</p>

###### Criminal-market signal

<p>Dark-web sweeps run on May 6, 2026 found no criminal-market presence for this campaign on publicly-observable venues.</p>

<p>The absence is structural, not circumstantial. Contagious Interview is a North Korean state operation with a specific monetization model: stolen credentials and persistent access feed DPRK's cryptocurrency theft program directly. The attack tooling — BeaverTail, InvisibleFerret, OtterCookie — is not a product. There is no market because there is no seller. The credentials go to the state; the capability stays internal. This pattern holds across every documented DPRK cyber operation and is unlikely to change regardless of how many sweeps are run or how much time passes.</p>

<p>Criminal-market monitoring provides no early warning for this campaign class. The social-engineering recruitment vector — fake recruiter, coding interview, install this repo — is the detection surface. Awareness training that specifically covers the fake-recruiter pattern is a required complement to any technical control, because the developer may intentionally install the malicious package believing it is a legitimate interview task.</p>

###### What distinguishes this from the TeamPCP cluster

<p>Both Contagious Interview and TeamPCP are active supply-chain campaigns operating in 2026 with overlapping technical tactics (staged loaders, credential theft, multi-ecosystem publishing). The meaningful distinctions for defenders:</p>

<ul role="list">
<li><strong>Initial access vector:</strong> TeamPCP compromises existing trusted packages via stolen CI/CD credentials, relying on passive installation by downstream consumers. Contagious Interview deploys fresh packages and actively recruits the first victim via social engineering. TeamPCP's blast radius is larger (2M-download packages); Contagious Interview's targeting is more intentional (developers in AI/crypto/fintech).</li>
<li><strong>Operator motivation:</strong> TeamPCP exfiltrates credentials and may sell or reuse them. Contagious Interview exfiltrates credentials and converts them directly to cryptocurrency theft — a state-level monetization operation, not a criminal market vendor.</li>
<li><strong>Persistence model:</strong> TeamPCP payloads are designed for quick credential sweep and worm propagation. The <code>license-utils-kit</code> Windows build is designed for sustained hands-on access via AnyDesk. The DPRK campaign expects to sit inside developer environments for extended periods.</li>
</ul>

<p>Defenders who monitor supply-chain registries for postinstall hooks and unexpected dependencies will catch both campaigns at the package level. But Contagious Interview's social-engineering entry point means the malicious package installation may be <em>intended</em> by the developer — standard SCA alerting on an unknown package may be dismissed as a false positive if the developer believes they're running a legitimate interview test. Awareness training that specifically covers the fake-recruiter pattern is a required complement to technical controls for this campaign.</p>
