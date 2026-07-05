---
title: "ChocoPoC: trojanized GitHub PoC repos hide a Python RAT in the requirements.txt, targeting vulnerability researchers"
date: 2026-07-03
heroImage: "/art/chocopoc-rat-trojanized-poc-pypi-2026.png"
description: "Sekoia and others report at least seven GitHub proof-of-concept exploit repositories that deliver ChocoPoC, a Python RAT, to the researchers who clone them. The malware is not in the exploit file — it hides in a malicious PyPI dependency pulled in by requirements.txt, executing when the researcher runs pip install to set up the exploit environment. C2 is staged as a Mapbox dead-drop."
slug: chocopoc-rat-trojanized-poc-pypi-2026
type: threat-intel
campaign: ChocoPoC
firstObserved: 2026-07-01
cves: []
targetSectors:
  - Security research
  - Vulnerability management teams
  - Red teams / pentest
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - supply-chain
  - pypi
  - github
  - rat
  - researcher-targeting
  - chocopoc
  - mapbox-c2
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: inferred
  infrastructure:
    registry_layer: "GitHub (fake PoC repos) + PyPI (malicious dependency, e.g. skytext)"
    c2_primary: "Mapbox API used as dead-drop C2 (feature IDs reused across waves)"
    c2_fallback: null
    exfil_channel: "Python RAT: file exfiltration, browser-credential theft, command execution"
    c2_resilience_tier: 3
  capability:
    initial_access: trojanized-poc-dependency
    execution: pip-install-native-extension-load
    evasion: timestomping-peb-walking-export-hashing-anti-debug
    persistence: rat-binary
  victim:
    direct: "Vulnerability researchers cloning and running fake PoC exploits"
    blast_radius: "≥7 PoC repos; malicious PyPI package skytext downloaded ~2,400 times (mostly Linux)"
sources:
  - title: "New ChocoPoC RAT Targets Vulnerability Researchers via Fake PoC Exploit Repos — The Hacker News"
    url: "https://thehackernews.com/2026/07/new-chocopoc-rat-targets-vulnerability.html"
  - title: "New ChocoPoC malware targets researchers via trojanized PoC exploits — BleepingComputer"
    url: "https://www.bleepingcomputer.com/news/security/new-chocopoc-malware-targets-researchers-via-trojanized-poc-exploits/"
  - title: "Don't eat the ChocoPoCs! How vulnerability researchers were repeatedly targeted by trojanised exploits — Sekoia"
    url: "https://www.sekoia.com/blog/dont-eat-the-chocopocs-how-vulnerability-researchers-were-repeatedly-targeted-by-trojanised-exploits"
---

<p><strong>ChocoPoC</strong> is a Python remote access trojan delivered through <strong>fake proof-of-concept (PoC) exploit repositories on GitHub</strong>, aimed squarely at the people who download exploits for a living: <strong>vulnerability researchers</strong>. The clever part is where the malware lives. It is <em>not</em> embedded in the exploit script — a researcher reading the code would not spot it. Instead, the operator adds a malicious package to the PoC's dependency list, hosted on <strong>PyPI</strong>. When the researcher runs <code>pip install</code> to stand up the exploit environment, a compiled native Python extension loads and the RAT is on the box. First reported by <strong>Sekoia</strong>, with tier-1 coverage from The Hacker News and BleepingComputer.</p>



###### What we know

<ul role="list">
<li><strong>Late 2025 – 2026</strong> — Multiple waves of trojanized PoC repositories are observed; investigators link them to a single actor reusing an "opsec" kit across campaigns.</li>
<li><strong>2026-07-01/02</strong> — Sekoia publishes analysis; The Hacker News and BleepingComputer report the campaign. At least <strong>seven</strong> GitHub PoC repos are identified.</li>
<li>The malicious PyPI package <strong><code>skytext</code></strong> is reported downloaded roughly <strong>2,400 times</strong>, mostly on Linux, with download spikes following disclosure of popular vulnerabilities.</li>
</ul>

<p><em>Source-layer flag: the RAT capabilities, the seven-repo count, the <code>skytext</code> download figure, and the Mapbox dead-drop C2 are reported from Sekoia and tier-1 press; we have not independently analyzed the malware sample or confirmed the PyPI artifact.</em></p>



###### Payload mechanics

<p>The delivery chain inverts the usual "read the code before you run it" defense. Researchers routinely audit the exploit file itself — but far fewer scrutinize the <code>requirements.txt</code> and the transitive dependencies that <code>pip install</code> pulls. ChocoPoC exploits exactly that gap: the fake exploit's dependency list includes a malicious PyPI entry; installing it loads a compiled native extension that is the RAT.</p>

<p>To make each wave credible, the operator hosts PoCs for genuinely current, high-interest CVEs, timing releases to disclosure spikes. Reported lures span: FortiWeb (CVE-2025-64446), React2Shell (CVE-2025-55182), MongoBleed (CVE-2025-14847), PAN-OS (CVE-2026-0257), Ivanti Sentry (CVE-2026-10520), Check Point VPN (CVE-2026-50751), and Joomla SP Page Builder (CVE-2026-48908).</p>

<p>Once resident, ChocoPoC is a full-featured RAT — file exfiltration, browser-credential harvesting, and arbitrary command execution — hardened with evasion: timestomping, file-lock mutexes, PEB walking, export hashing, and anti-debugging. C2 is staged as a <strong>Mapbox dead-drop</strong>, with reused Mapbox feature IDs among the clustering signals.</p>

<p><em>Source-layer flag: CVE lures, evasion techniques, and C2 tradecraft above are reported from Sekoia's writeup, not independently reverse-engineered here.</em></p>

<blockquote>The target selection is deliberate and compounding: compromise the researchers, and you get early access to exploit development, credentials into security orgs, and a trusted position from which the next batch of poisoned PoCs looks even more legitimate.</blockquote>



###### Indicators of compromise

<p><strong>Malicious PyPI package:</strong> <code>skytext</code> (~2,400 downloads; remove and audit any environment that installed it).</p>

<p><strong>Delivery:</strong> GitHub "PoC exploit" repositories whose <code>requirements.txt</code> pulls an unfamiliar package that ships a compiled native extension.</p>

<p><strong>CVE lures used as bait:</strong> <code>CVE-2025-64446</code> (FortiWeb), <code>CVE-2025-55182</code> (React2Shell), <code>CVE-2025-14847</code> (MongoBleed), <code>CVE-2026-0257</code> (PAN-OS), <code>CVE-2026-10520</code> (Ivanti Sentry), <code>CVE-2026-50751</code> (Check Point VPN), <code>CVE-2026-48908</code> (Joomla SP Page Builder).</p>

<p><strong>C2:</strong> <code>api.mapbox[.]com</code> abused as a dead-drop channel (reused Mapbox feature IDs across waves).</p>

<p><strong>Host tradecraft:</strong> timestomping, file-lock mutexes, PEB walking, export hashing, anti-debugging — presence of these in a Python-loaded native extension is a strong signal.</p>



###### Detection and mitigation

<ul role="list">
<li><strong>Never run untrusted PoCs on a real endpoint.</strong> Detonate exploit code — and its <code>pip install</code> — only in a disposable, network-isolated VM or container.</li>
<li><strong>Audit dependencies before install:</strong> read <code>requirements.txt</code> and pin/verify every package; treat an unfamiliar dependency in an "exploit" repo as hostile by default.</li>
<li><strong>Search build/research hosts for the <code>skytext</code> package</strong> and for outbound connections to <code>api.mapbox[.]com</code> from Python processes that have no legitimate mapping use.</li>
<li><strong>Rotate credentials</strong> (browser-stored, cloud, SSH) on any researcher host that ran one of these PoCs; assume browser-credential theft.</li>
<li><strong>Egress-monitor research subnets</strong> for dead-drop-style C2 riding legitimate SaaS APIs.</li>
</ul>



###### Attribution

<p>Investigators attribute the waves to a <strong>single actor</strong> based on operational reuse — shared Mapbox feature IDs, identical hashing gates, and consistent persistence techniques across late-2025 and 2026 campaigns. No named group or nation-state is asserted in the reporting reviewed. What is claimed: one operator, reusing an opsec kit, deliberately targeting security researchers. What is not claimed: a specific APT identity or state sponsor. Confidence in the single-actor clustering is <strong>inferred</strong> (behavioral, per Sekoia); confidence in any further identity is unknown. Researcher-targeting has historical precedent with DPRK-nexus activity, but no such link is claimed here.</p>



###### Criminal-market signal

<p><strong>Bounded negative (evidenced):</strong> on 2026-07-03 we swept ~25 dark-web search engines over Tor for <code>ChocoPoC</code> and the malicious PyPI package <code>skytext</code>; ~44 pages were crawled across both terms and <strong>zero</strong> were flagged relevant. No criminal-market listing or forum discussion was found on the venues our pipeline can see (closed forums and Telegram are out of scope). The campaign profile is <em>operator-run, not commodity</em>: bespoke researcher-targeting with a reused private opsec kit and a dead-drop C2 is the signature of an operator using the tooling for access, not selling it on forums (the H2 pattern, consistent with our Shai-Hulud/TeamPCP framing). Absence of a marketplace listing therefore does not reduce risk — detection belongs at the developer-workstation and PyPI-dependency layer, not criminal-market monitoring.</p>



###### CISO translation

<p>Attackers are publishing fake "exploit code" on GitHub that hides malware inside the Python packages it installs, specifically to infect security researchers who download and test it. This matters to us because our own security and engineering staff routinely pull down PoC exploits, and one careless <code>pip install</code> on a work machine can hand over browser passwords, cloud keys, and remote control of that host. Mandate that all untrusted exploit code — including its dependency install — runs only in throwaway isolated VMs, and audit any research host that recently installed the package <code>skytext</code>. Confidence is high that the campaign is real and active (independent vendor analysis, seven identified repos, thousands of downloads), though the specific operator is not named. The cost of inaction is the compromise of the very people who hold our security keys — a foothold that is unusually damaging because it lands inside the security team itself.</p>
