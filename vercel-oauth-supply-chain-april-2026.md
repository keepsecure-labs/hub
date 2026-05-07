---
title: "Vercel's April 2026 incident: an OAuth-app supply chain in three hops"
date: 2026-04-30
heroImage: "https://assets.website-files.com/614519168cffbd131c32d792/61451eb41100bf61e2edaeb4_blog%201.svg"
description: "Vercel's April 2026 breach moved from a third-party AI tool to a Workspace account to internal Vercel access. The OAuth-app supply-chain pattern in detail."
slug: vercel-oauth-supply-chain-april-2026
type: threat-intel
campaign: "Vercel April 2026 incident"
firstObserved: 2026-04-19
targetSectors:
  - SaaS
  - Cloud platforms
  - Developer infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - OAuth
  - Google Workspace
  - third-party risk
  - supply chain
  - identity
  - Vercel
sources:
  - title: "Vercel April 2026 security incident bulletin"
    url: "https://vercel.com/kb/bulletin/vercel-april-2026-security-incident"
  - title: "Datadog Security Research: Vercel incident card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "Datadog Cloud SIEM Google Workspace OAuth detection rules"
    url: "https://docs.datadoghq.com/security/default_rules/def-000-043"
diamond:
  adversary:
    operator_model: operator-run
    monetization: access-sale
    confidence: unknown
  infrastructure:
    registry_layer: "Third-party AI tool OAuth app with Google Workspace read scope"
    c2_primary: "Google Workspace OAuth grant (legitimate channel abused)"
    c2_fallback: null
    exfil_channel: "Vercel internal API access via hijacked OAuth token chain"
    c2_resilience_tier: 1
  capability:
    initial_access: oauth-app-abuse
    execution: credential-access-via-oauth
    evasion: legitimate-oauth-flow
    persistence: persistent-oauth-grant
  victim:
    direct: "Vercel (internal access via Google Workspace OAuth chain)"
    blast_radius: "Developers with third-party AI tools connected to Google Workspace with broad OAuth scopes"
---

<p>On April 20, 2026, Vercel disclosed the root cause of an internal-systems compromise: a third-party AI tool (Context.ai) used by a Vercel employee was breached, the foothold was used to take over the employee's Vercel-linked Google Workspace account, and from there the attacker reached certain Vercel environments and environment variables. Three hops, each with its own trust boundary, each broken in turn. The pattern is "OAuth app as supply chain" — and it's the cleanest published example so far of how a SaaS company gets compromised through a tool its employees connected to their identity provider, not through any direct vulnerability in its own infrastructure.</p>

###### What we know

<ul role="list">
<li><strong>April 19, 2026</strong> — Vercel discloses the incident. Initial bulletin states attackers accessed certain internal systems; services remain operational; a limited subset of customers is impacted.</li>
<li><strong>April 20, 2026, 02:01 UTC</strong> — Vercel publishes the root cause. Three-hop chain: Context.ai compromise → Vercel employee's Google Workspace account → certain Vercel environments and environment variables stored as non-sensitive.</li>
<li><strong>Confirmed IOC</strong> — Google Workspace OAuth client ID <code>110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com</code>, corresponding to the third-party AI app in the attack path.</li>
<li><strong>Scope of access</strong> — Vercel reports values stored as <em>sensitive</em> environment variables remained unreadable. Values stored as <em>non-sensitive</em> environment variables should be treated as exposed.</li>
</ul>

<blockquote>The non-sensitive distinction is the lesson. Vercel's environment-variable model has two tiers; the lower tier is not encrypted-at-rest in a way that survives the kind of access an OAuth-app compromise grants. Anyone running secrets through non-sensitive Vercel env vars discovered they were trusting a different threat model than they thought.</blockquote>

###### Targeting

<p>Direct targeting was Vercel itself; second-order targeting includes any Vercel customer whose secrets sat in a non-sensitive environment variable on a project the compromised internal accounts could reach.</p>

<ul role="list">
<li><strong>Vercel internal infrastructure</strong> — the direct target. Internal accounts, certain environments, environment-variable inventories.</li>
<li><strong>The Vercel customers Vercel contacted</strong> — limited subset, specific to the projects/environments the compromised accounts could reach.</li>
<li><strong>Any organization using OAuth third-party AI tools widely</strong> — same vector applies. The Vercel chain is one realization of a general pattern.</li>
</ul>

###### TTPs and the OAuth-as-supply-chain pattern

<p>The chain in operational terms:</p>

<ul role="list">
<li><strong>Hop 1 — Context.ai compromise.</strong> Public reporting does not detail the initial-access mechanism on Context.ai's side. The salient observation is that a third-party AI tool installed by individual employees became the source of foothold.</li>
<li><strong>Hop 2 — Google Workspace OAuth grant abuse.</strong> Context.ai had been authorized as an OAuth third-party application against the employee's Google Workspace identity. With Context.ai compromised, the attacker leveraged the OAuth grant to access the employee's Google Workspace mailbox, drive, and adjacent services within the scope of the granted permissions.</li>
<li><strong>Hop 3 — internal Vercel access via Google identity federation.</strong> Vercel uses Google Workspace as the identity backbone. Once the attacker had the employee's Google Workspace foothold, services that federate identity from Workspace were reachable. From there the attacker reached Vercel environments and environment variables the employee's identity could see.</li>
</ul>

<p>The trust assumption that breaks each hop is different. Hop 1 assumes Context.ai's security is comparable to Vercel's (it isn't necessarily — third-party AI tools are often early-stage companies with smaller security programs). Hop 2 assumes the OAuth grant is a permission boundary; it is, but only against external attackers, not against attackers who have compromised the OAuth client itself. Hop 3 assumes Workspace federation is a strong identity boundary; it is, until you remember that the Workspace account itself was reached from outside.</p>

###### Indicators of compromise

<ul role="list">
<li><strong>Google Workspace OAuth client ID</strong> — <code>110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com</code>. Filter Workspace audit logs for activity from this client ID during the affected window.</li>
<li><strong>Context.ai authorization grants</strong> — review <code>Security &gt; Access and data control &gt; API controls &gt; Manage Third-Party App Access</code> in Workspace admin for the listed client ID; revoke wherever found.</li>
<li><strong>Vercel non-sensitive environment variables</strong> — inventory across every project. Secrets stored at this tier in the affected window should be treated as exposed.</li>
</ul>

<p><strong>Datadog log queries (provided by Datadog Security Research):</strong></p>

<ul role="list">
<li>Activity for the OAuth client ID: <code>source:gsuite @actor.applicationInfo.oauthClientId:110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com</code></li>
<li>Authorization events for the client ID: <code>source:gsuite @evt.name:authorize @event.parameters.client_id:110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com</code></li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Revoke the OAuth grant.</strong> Find the Context.ai client ID in your Workspace admin third-party app inventory and revoke. Have individual users review their personal Google account third-party access for the same client ID.</li>
<li><strong>Rotate everything in non-sensitive Vercel environment variables.</strong> API keys, access tokens, database credentials, signing material. Re-create as <em>sensitive</em> environment variables wherever the Vercel project supports the distinction.</li>
<li><strong>Audit Workspace OAuth grants holistically.</strong> The Vercel incident is one realization of a class. Inventory all third-party AI tools authorized as OAuth apps; review the scopes each holds; revoke ones that aren't load-bearing or whose vendor security posture you can't validate.</li>
<li><strong>Treat Workspace as a privileged identity boundary.</strong> If your engineering platform federates from Workspace, every Workspace account compromise is also a platform compromise. Phishing-resistant MFA on Workspace, conditional access policies, and short-lived federated sessions all reduce the blast radius of hop 3.</li>
<li><strong>Detect OAuth-key-driven privileged actions.</strong> Datadog Cloud SIEM ships rules covering OAuth keys performing account creation or security changes, and unfamiliar service accounts modifying group memberships. These are the post-OAuth-compromise patterns to watch for.</li>
<li><strong>Detect session-cookie hijacking.</strong> Google's own session-termination signals (suspicious cookie detection) are a useful tripwire — Cloud SIEM's <em>Google Workspace user account signed out due to suspicious session cookie</em> rule surfaces them.</li>
</ul>

###### What the OAuth-as-supply-chain pattern means

<p>Three observations stand out from the Vercel chain:</p>

<ul role="list">
<li><strong>Third-party AI tools are an under-modeled supply chain.</strong> Employees connect them to Workspace identities at individual discretion. The grant lifetime is long, the scope is often broader than the use case requires, and the vendor's security posture is often early-stage. Context.ai-class compromise is going to recur; the question is whether your organization's OAuth grant inventory tracks who's authorized what.</li>
<li><strong>"Sensitive" vs "non-sensitive" environment variables are a real boundary that defenders need to use.</strong> Vercel's distinction between the two tiers wasn't marketing — sensitive values were unreadable to the attacker; non-sensitive values were exposed. Other platforms have similar distinctions (Kubernetes Secret encryption-at-rest, AWS Parameter Store SecureString, GCP Secret Manager). Use the protected tier wherever it exists.</li>
<li><strong>Identity-federation-as-supply-chain is the next chapter.</strong> When Workspace is your platform identity, Workspace OAuth surface is your platform's perimeter. The same employees who would never grant a random vendor admin access to your AWS account routinely grant Workspace-OAuth-third-party-app access without much review. The grants are the perimeter now.</li>
</ul>

<p>The closest conceptual peer in the recent record is <a href="https://keepsecure.io/hub/ai-agent-confused-deputy-pattern">the AI-agent-as-confused-deputy pattern</a> — a privileged process operating on attacker-influenced input. Here the deputy is the Context.ai OAuth app, the input is whatever the attacker fed into Context.ai, and the privileges are Workspace-grant scope. Same pattern, different deputy. Capability narrowing — short-lived OAuth grants, scope-minimized rather than scope-maximized — is the structural defense, the same way it is for AI-agent runtimes.</p>

<p>For comparison with a different supply-chain shape running concurrently, see the analysis of <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">Team PCP's six-week chain through security tooling vendors</a>. Team PCP attacks the package-publish boundary. The Vercel chain attacks the OAuth-grant boundary. Both end with attacker-controlled code (or attacker-controlled identity) reaching credentials it shouldn't. Different vectors, same outcome class.</p>

<p>One final framing note. The Vercel chain has no observable presence in criminal markets — no Context.ai-credential brokering, no Workspace-OAuth-grant trade, no commodity packaging of the technique. That's the expected shape: identity-federation attacks like this one are operator-specific, executed by the same attacker who compromised hop 1, and don't commoditize into kits other operators can buy. The detection signal lives in your Workspace audit logs and your Vercel project inventory — not on .onion forums. Same lesson as the <a href="https://keepsecure.io/hub/ai-ide-marketplace-security-telemetry">AI-IDE marketplace surface</a>: monitor the legitimate channel where the threat actually lives.</p>
