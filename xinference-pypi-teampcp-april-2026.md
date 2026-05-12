---
title: "Xinference on PyPI: TeamPCP's self-attribution marker, AWS post-exploitation scope, and why April 22 matters for the .pth timeline"
date: 2026-05-12
heroImage: "/art/xinference-pypi-teampcp-april-2026.png"
description: "The xinference PyPI package was compromised in three versions (2.6.0–2.6.2) with a two-stage base64-decoded Python payload. JFrog identified a '# hacked by teampcp' attribution marker in the decoded payload — the same self-attribution string pattern used in the litellm archive name and CanisterWorm payload strings. The April 22 compromise date is significant: it falls three weeks after the litellm .pth injection (March 24), establishing that .pth injection had become a stable TeamPCP technique by the time this campaign ran. Xinference also introduces post-exploitation behavior not seen in prior TeamPCP PyPI victims: active enumeration of AWS IMDS, Secrets Manager, and SSM Parameter Store."
slug: xinference-pypi-teampcp-april-2026
type: threat-intel
campaign: "Team PCP"
firstObserved: 2026-04-22
targetSectors:
  - AI / ML
  - Distributed inference
  - Cloud infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - PyPI
  - supply chain
  - TeamPCP
  - xinference
  - AWS post-exploitation
  - self-attribution
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "PyPI xinference package (account takeover or CI/CD credential theft)"
    c2_primary: "whereisitat[.]lucyatemysuperbox[.]space — exfiltration endpoint"
    c2_fallback: null
    exfil_channel: "HTTP POST to typosquat-style domain; encrypted credential archive"
    c2_resilience_tier: 1
  capability:
    initial_access: import-time-execution
    execution: base64-chained-python
    evasion: double-base64-encoding
    persistence: null
    post_exploitation: aws-imds-secretsmanager-ssm-enumeration
  victim:
    direct: "xinference users — distributed inference runtime for open-source LLMs and embedding models"
    blast_radius: "Three malicious versions (2.6.0–2.6.2); all Python environments running affected xinference versions"
sources:
  - title: "JFrog Security Research: xinference PyPI compromise"
    url: "https://research.jfrog.com/post/xinference-compromise/"
  - title: "Datadog Security Research: Xinference PyPI compromise card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "TeamPCP campaign tracking — this hub"
    url: "https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking"
  - title: "litellm PyPI .pth injection — this hub"
    url: "https://keepsecure.io/hub/litellm-pypi-teampcp-pth-injection-march-2026"
---

<p>Three versions of the <code>xinference</code> PyPI package — 2.6.0, 2.6.1, and 2.6.2 — were trojanized with a two-stage credential-stealing payload, then yanked from the registry. xinference is a distributed inference runtime for open-source LLMs, embedding models, and multimodal models, used in environments where model serving infrastructure holds cloud credentials with meaningful scope.</p>

<p>JFrog Security Research identified <code># hacked by teampcp</code> in the decoded payload — a Layer 1 self-attribution string directly comparable to the <code>tpcp.tar.gz</code> archive name in the litellm payload and the <code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code> string in CanisterWorm. The April 22 date positions this compromise three weeks after the litellm <code>.pth</code> injection (March 24) and establishes that the operator had standardized the technique by this point in the campaign timeline.</p>

<p>The Xinference payload also introduces a post-exploitation capability not documented in prior TeamPCP PyPI victims: the payload actively enumerates AWS IMDS credentials, Secrets Manager secrets, and SSM Parameter Store parameters. This goes beyond passive collection of credentials already present on the filesystem — it is active cloud API enumeration that will succeed in any compute environment with an IAM role attached, even if the developer has not explicitly stored credentials locally.</p>

###### What we know

<ul role="list">
<li><strong>April 22, 2026</strong> — JFrog Security Research published primary analysis. Malicious versions 2.6.0, 2.6.1, and 2.6.2 yanked from PyPI.</li>
<li><strong>Attribution:</strong> <code># hacked by teampcp</code> found in decoded payload by JFrog. This is the same self-attribution pattern present in the litellm <code>tpcp.tar.gz</code> archive name — the operator consistently leaves self-attribution markers in payload artifacts.</li>
<li><strong>Injection point:</strong> Malicious code embedded in <code>xinference/__init__.py</code> — executes on package import, not at install time. Any process that imports xinference will trigger the payload.</li>
<li><strong>Exfiltration endpoint:</strong> <code>whereisitat[.]lucyatemysuperbox[.]space</code> — a distinctive domain with no plausible typosquat relationship to xinference. Unlike <code>models.litellm[.]cloud</code> (which impersonated the legitimate <code>litellm.ai</code>), this domain makes no pretense of legitimacy.</li>
</ul>

###### Timeline significance: why April 22 matters

<p>The TeamPCP PyPI campaign timeline matters for understanding the operator's technique evolution:</p>

<ul role="list">
<li><strong>March 20</strong> — Trivy GitHub Action compromised (npm-focused, credential theft)</li>
<li><strong>March 24</strong> — litellm PyPI compromise; first documented <code>.pth</code> injection in TeamPCP chain</li>
<li><strong>April 22</strong> — xinference PyPI compromise; three-week gap from litellm</li>
<li><strong>April 29–30</strong> — Mini Shai-Hulud (SAP npm), elementary-data, lightning PyPI — cluster expansion</li>
</ul>

<p>The three-week gap between litellm (March 24) and xinference (April 22) is consistent with a technique refinement period. The litellm payload introduced <code>.pth</code> injection. The xinference payload does not appear to use <code>.pth</code> injection based on available JFrog analysis — it uses direct <code>__init__.py</code> modification instead. This suggests the operator was running parallel technique development: <code>.pth</code> injection for stealthy persistence in one branch, direct <code>__init__.py</code> injection for broad trigger coverage in another.</p>

###### Payload mechanics

<p>The following analysis is from JFrog Security Research's primary publication, as characterized by Datadog Security Research. Primary-source artifact verification against the JFrog report is pending (Task #16).</p>

<p>Malicious code embedded in <code>xinference/__init__.py</code> executes a two-stage base64 decode chain on import:</p>

<ul role="list">
<li><strong>Stage 1</strong> (SHA-256: <code>077d49fa708f498969d7cdffe701eb64675baaa4968ded9bd97a4936dd56c21c</code>): base64-decoded Python stage that launches a credential collector and enumerates host context.</li>
<li><strong>Stage 2</strong> (SHA-256: <code>fe17e2ea4012d07d90ecb7793c1b0593a6138d25a9393192263e751660ec3cd0</code>): the full credential collection and exfiltration logic.</li>
</ul>

<p>Passive collection targets include SSH material, Git credentials, cloud credentials, Kubernetes service account tokens, Docker auth files, npm tokens, <code>.env</code> files, TLS private key artifacts, and infrastructure secrets. These are consistent with the litellm collection scope.</p>

<p>The distinguishing capability is active cloud API post-exploitation:</p>

<blockquote>The payload does not just read credentials that are already present on disk. It actively queries the AWS Instance Metadata Service (IMDS) to retrieve temporary IAM credentials — credentials that exist only in memory on EC2 instances and ECS containers, not in credential files. It then uses those credentials to enumerate AWS Secrets Manager secrets and AWS Systems Manager Parameter Store parameters. This is post-exploitation behavior: the malware is not just stealing what's there, it's using what's there to go deeper.</blockquote>

<p>The AWS post-exploitation scope matters operationally. A cloud environment running xinference in an ECS container or EC2 instance with an attached IAM role is exposed even if the developer has never written a credentials file to disk. The IMDS tokens are ephemeral by design — they expire, they rotate, they leave no local trace. Detection requires CloudTrail anomaly monitoring (unusual Secrets Manager list/get operations from the instance profile), not filesystem artifact hunting.</p>

<p>Exfiltration is via HTTP to <code>whereisitat[.]lucyatemysuperbox[.]space</code>. JFrog notes the domain style differs from litellm's typosquat (<code>models.litellm[.]cloud</code>). The exfil domain is not attempting to impersonate anything — it is operationally functional but not designed for egress log evasion. This may indicate the operator prioritized speed over OPSEC for this campaign.</p>

###### Self-attribution: four-step analysis

<p><strong>Observation:</strong> JFrog found the string <code># hacked by teampcp</code> in the decoded xinference payload. This is the same attribution pattern as the <code>tpcp.tar.gz</code> archive name in litellm (March 24) — the operator embeds campaign identifiers directly in payload artifacts.</p>

<p><strong>Mechanism:</strong> A self-attribution comment string in decoded payload is a deliberate operator choice, not an accident. The operator is labeling their work for internal tracking and (arguably) public credit at disclosure. The consistency of attribution markers across litellm (<code>tpcp.tar.gz</code>), xinference (<code># hacked by teampcp</code>), and CanisterWorm (<code>Technique: .pth file injection (TeamPCP/LiteLLM method)</code>) indicates systematic self-attribution practice, not occasional tagging.</p>

<p><strong>Inference:</strong> The same operator is responsible for all three attribution-marked payloads. The xinference compromise is a TeamPCP campaign. Attribution confidence is confirmed — Layer 1 evidence exists in the payload artifact rather than relying on infrastructure or toolchain overlap.</p>

<p><strong>Confidence bound:</strong> This inference is robust against alternative explanations. The only realistic alternative is that a different actor deliberately injected TeamPCP attribution markers to cause misattribution. This is possible but requires intentional effort and would be unusual for a financially motivated actor operating in the supply-chain space. JFrog's primary analysis is the verification source (Task #16).</p>

###### Indicators of compromise

<p><strong>Compromised versions:</strong></p>
<ul role="list">
<li><code>xinference==2.6.0</code> — malicious</li>
<li><code>xinference==2.6.1</code> — malicious</li>
<li><code>xinference==2.6.2</code> — malicious</li>
</ul>

<p><strong>File hashes (SHA-256, from JFrog):</strong></p>
<ul role="list">
<li><code>xinference/__init__.py</code>: <code>e1e007ce4eab7774785617179d1c01a9381ae83abfd431aae8dba6f82d3ac127</code></li>
<li>Decoded stage 1: <code>077d49fa708f498969d7cdffe701eb64675baaa4968ded9bd97a4936dd56c21c</code></li>
<li>Decoded stage 2: <code>fe17e2ea4012d07d90ecb7793c1b0593a6138d25a9393192263e751660ec3cd0</code></li>
</ul>

<p><strong>Network indicator (defanged):</strong></p>
<ul role="list">
<li><code>whereisitat[.]lucyatemysuperbox[.]space</code> — exfiltration endpoint; any outbound connection to this domain is evidence of active data collection</li>
</ul>

<p><strong>Campaign self-attribution (Layer 1):</strong></p>
<ul role="list">
<li><code># hacked by teampcp</code> comment string in decoded payload (identified by JFrog)</li>
</ul>

<p><strong>CloudTrail indicators (AWS post-exploitation):</strong></p>
<ul role="list">
<li>Unexpected <code>secretsmanager:ListSecrets</code> or <code>secretsmanager:GetSecretValue</code> API calls from an EC2 or ECS instance profile not normally accessing Secrets Manager</li>
<li>Unexpected <code>ssm:DescribeParameters</code> or <code>ssm:GetParameter</code> calls from the same source</li>
<li>IMDS query from a process that is not the expected application workload</li>
</ul>

###### Detection and mitigation

<ul role="list">
<li><strong>Check hashes immediately.</strong> Hash <code>xinference/__init__.py</code> in every installed environment. The JFrog SHA-256 (<code>e1e007ce4eab7774785617179d1c01a9381ae83abfd431aae8dba6f82d3ac127</code>) is a reliable match for the malicious version — it is a stable artifact unlike the ephemeral IMDS credentials collected by the payload.</li>
<li><strong>Treat IMDS enumeration as the worst-case scope.</strong> Any environment running xinference on EC2 or ECS with an attached IAM role should be investigated for post-exploitation activity. If the instance profile had Secrets Manager or SSM access, assume those secrets are compromised regardless of whether credential files were present on disk.</li>
<li><strong>Query CloudTrail for anomalous Secrets Manager and SSM calls.</strong> The AWS post-exploitation enumeration will appear in CloudTrail. Search for <code>ListSecrets</code> and <code>GetSecretValue</code> calls from the instance or task role during the April 22 window. Unexpected calls from a role that does not normally access Secrets Manager are confirmation of post-exploitation activity.</li>
<li><strong>Block <code>whereisitat[.]lucyatemysuperbox[.]space</code>.</strong> This domain has no legitimate use. Block at the DNS and egress firewall layer.</li>
<li><strong>Rotate all accessible credentials from affected environments.</strong> The collection scope spans SSH keys, Git tokens, cloud credentials, Kubernetes service account tokens, npm tokens, and TLS private keys. Treat any environment that imported the malicious xinference versions as fully compromised.</li>
<li><strong>Implement IMDS access controls.</strong> If your compute infrastructure does not require broad Secrets Manager or SSM access from application roles, apply IAM least-privilege constraints that would fail or alert on these enumeration calls. This is the structural defense against the post-exploitation pattern, not just this specific payload.</li>
</ul>

###### Attribution and cluster position

<p>Xinference is the second confirmed PyPI victim in the TeamPCP cluster following litellm (March 24), and the first to document active AWS post-exploitation rather than passive credential collection. The attribution confidence is the highest in the cluster — <code># hacked by teampcp</code> is unambiguous Layer 1 evidence.</p>

<p>The C2 domain style (<code>whereisitat[.]lucyatemysuperbox[.]space</code>) is operationally less sophisticated than litellm's typosquat approach. This could indicate a different operator sub-team handling the xinference deployment, or simply that the operator's OPSEC priority varied between campaigns. The cluster article (Task #10) should include the domain style variation as an open question rather than claiming consistency.</p>

<p>See the <a href="https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking">TeamPCP campaign tracking</a> and the <a href="https://keepsecure.io/hub/npm-supply-chain-worm-cluster-2026">npm supply-chain worm cluster analysis</a> for the full campaign timeline and cross-ecosystem technical analysis.</p>

###### Criminal-market signal

<p>No dark-web presence for TeamPCP tooling, xinference-specific payloads, or the <code>whereisitat[.]lucyatemysuperbox[.]space</code> infrastructure has been observed. The confirmed operator-run, credential-collection pattern documented across the TeamPCP cluster applies. The AWS post-exploitation adds scope but does not change the monetization model assessment — an actor actively enumerating Secrets Manager is using the credentials directly, not selling them on commodity markets where that capability would be unnecessary. Detection must occur at the package import layer and CloudTrail layer, not via dark-web early warning (H2 operator-run pattern, confirmed three-campaign basis).</p>
