---
title: "durabletask PyPI (TeamPCP/Mini Shai-Hulud): multi-cloud credential stealer with lateral movement and conditional wiper"
date: 2026-05-19
heroImage: "/art/durabletask-pypi-teampcp-multi-cloud-stealer-may-2026.png"
description: "Three trojanized versions of Microsoft's official durabletask Python SDK (1.4.1, 1.4.2, 1.4.3) were published to PyPI on May 19, 2026 using compromised publishing credentials. The payload (rope.pyz) is the most capable TeamPCP/Mini Shai-Hulud intrusion framework to date: multi-cloud credential theft across AWS, Azure, and GCP, Kubernetes secret enumeration, 99 developer tooling paths, password manager vault access, lateral movement via AWS SSM SendCommand and kubectl exec, Russian folklore-themed GitHub exfiltration repositories, and a conditional destructive wiper targeting specific geographies."
slug: durabletask-pypi-teampcp-multi-cloud-stealer-may-2026
type: threat-intel
campaign: "Mini Shai-Hulud / Team PCP"
firstObserved: 2026-05-19
cves: []
targetSectors:
  - Azure / Microsoft cloud
  - Kubernetes operators
  - Python developers
  - CI/CD infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - PyPI
  - supply-chain
  - TeamPCP
  - Mini Shai-Hulud
  - durabletask
  - Microsoft
  - Azure
  - lateral-movement
  - credential-theft
  - destructive
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "PyPI durabletask package (compromised publishing credentials, no Trusted Publishing/OIDC)"
    c2_primary: "check.git-service[.]com"
    c2_fallback: "t.m-kosche[.]com (known TeamPCP infrastructure)"
    exfil_channel: "HTTPS to C2; GitHub dead-drop to Russian folklore-named public repos"
    c2_resilience_tier: 4
  capability:
    initial_access: import-time-execution
    execution: dropper-fetches-rope-pyz-second-stage
    evasion: no-oidc-trusted-publishing-direct-credential-upload
    persistence: "pgsql-monitor.service systemd unit"
    post_exploitation: "aws-ssm-sendcommand lateral-movement; kubectl-exec lateral-movement; secrets-manager-enumeration; conditional-wiper"
    destructive_component: conditional
  victim:
    direct: "Users of durabletask Python SDK — Microsoft Azure Durable Task Framework for Python"
    blast_radius: "~16,000 daily downloads; any Python environment that imported versions 1.4.1, 1.4.2, or 1.4.3"
sources:
  - title: "StepSecurity: Microsoft's durabletask PyPI Package Compromised in Supply Chain Attack"
    url: "https://www.stepsecurity.io/blog/microsofts-durabletask-pypi-package-compromised-in-supply-chain-attack"
  - title: "GitHub Issue #137: Compromise report"
    url: "https://github.com/Azure/durabletask-python/issues/137"
  - title: "Datadog Security Research: durabletask card"
    url: "https://app.datadoghq.com/security/feed"
  - title: "TeamPCP campaign tracking — this hub"
    url: "https://keepsecure.io/hub/teampcp-supply-chain-campaign-tracking"
---

<p>Three versions of Microsoft's official <code>durabletask</code> Python SDK — 1.4.1, 1.4.2, and 1.4.3 — were published to PyPI on May 19, 2026 using compromised publishing credentials, bypassing the repository's CI/CD pipeline entirely. The package (the Azure Durable Task Framework for Python) averages roughly 16,000 downloads per day. Microsoft confirmed the compromise and delisted the affected versions. The attack has been linked to the TeamPCP threat group and its Mini Shai-Hulud campaign — the same operation behind TanStack, Mistral AI, LiteLLM, and the <code>@antv</code> compromises.</p>

<p>The <code>rope.pyz</code> second-stage payload is the most capable TeamPCP intrusion framework documented to date. It goes beyond passive credential collection: it spreads laterally through AWS SSM <code>SendCommand</code> and Kubernetes <code>kubectl exec</code>, enumerates secrets from every cloud provider and namespace accessible from the compromised environment, and includes a conditional destructive wiper targeting specific geographies — the same conditional destructive component seen in the Mini Shai-Hulud TanStack/Mistral wave.</p>

###### Publish chain: no Trusted Publishing, direct token upload

<p>The attacker published directly to PyPI using compromised credentials. No corresponding version tags or CI/CD workflow runs exist in the GitHub repository — the commits that would normally precede a release are absent. The repository does not use PyPI Trusted Publishing (OIDC), so there was no mechanism to reject an upload from outside the CI/CD pipeline. This is the structural gap that made the attack possible: <code>durabletask</code>'s publishing model relied on long-lived PyPI tokens rather than short-lived OIDC credentials tied to a specific workflow run.</p>

<p>The three versions were published in a 35-minute window on May 19, 2026:</p>

<table>
<tr><th>Version</th><th>Upload time (UTC)</th><th>Files infected</th></tr>
<tr><td>1.4.1</td><td>16:19</td><td>1 (<code>__init__.py</code>)</td></tr>
<tr><td>1.4.2</td><td>16:49</td><td>2 (<code>__init__.py</code>, <code>task.py</code>)</td></tr>
<tr><td>1.4.3</td><td>16:54</td><td>5 (<code>__init__.py</code>, <code>task.py</code>, <code>entities/__init__.py</code>, <code>extensions/__init__.py</code>, <code>payload/__init__.py</code>)</td></tr>
</table>

<p>Each version expanded the infection surface, reaching more files in the package structure. The 5-minute gap between 1.4.2 and 1.4.3 suggests automated publishing rather than manual iteration.</p>

###### rope.pyz: credential collection scope

<p>Each trojanized version runs a dropper at import time on Linux hosts that fetches the second-stage payload (<code>rope.pyz</code>) from the C2 server. The collection scope is broader than any prior TeamPCP payload:</p>

<ul role="list">
<li><strong>Cloud provider credentials:</strong> AWS access keys, EC2 instance metadata, Secrets Manager and SSM Parameter Store values across 19 regions. Azure service principals, Key Vault secrets, and managed identity tokens. GCP service account keys and Secret Manager entries.</li>
<li><strong>Kubernetes:</strong> Kubeconfig files and service account tokens. Dumps secrets from every namespace and context — not just the default namespace.</li>
<li><strong>Developer tooling:</strong> 99 hardcoded credential paths covering SSH keys, Git credentials, Docker configs, npm/PyPI/Cargo tokens, Terraform state, shell histories, <code>.env</code> files, and IDE/AI tool configurations.</li>
<li><strong>Password managers:</strong> 1Password, Bitwarden, <code>pass</code>, and <code>gopass</code> vaults.</li>
<li><strong>HashiCorp Vault:</strong> Tokens and AppRole credentials. Enumerates all KV mounts.</li>
</ul>

###### Lateral movement: AWS SSM and kubectl exec

<p>The framework spreads beyond the initial compromised host through two mechanisms:</p>

<ul role="list">
<li><strong>AWS SSM <code>SendCommand</code>:</strong> Using stolen AWS credentials with SSM permissions, the payload issues <code>SendCommand</code> to run collection and payload-fetch commands on other EC2 instances in the same account. This turns a single compromised developer machine or CI runner into access to the entire EC2 fleet reachable by the stolen IAM role.</li>
<li><strong>Kubernetes <code>kubectl exec</code>:</strong> Using stolen kubeconfig credentials, the payload executes commands inside running containers across all accessible namespaces. Any Kubernetes workload reachable by the stolen service account token becomes a lateral movement target.</li>
</ul>

<p>This is qualitatively different from the credential-collection-only behavior of earlier TeamPCP PyPI victims (litellm, xinference, lightning). The operator is not just stealing credentials — they are moving through infrastructure in real time.</p>

###### Russian folklore exfiltration repositories

<p>Harvested data is committed to public GitHub repositories created under victim accounts. The repository names follow a Russian folklore theme: <code>BABA-YAGA</code>, <code>KOSCHEI</code>, <code>FIREBIRD</code>, <code>VASSILISA</code>, <code>RUSALKA</code>. This is a new exfiltration branding distinct from the prior Mini Shai-Hulud Dune-themed repository names (<code>sayyadina-stillsuit-852</code>, etc.). Both patterns serve the same operational function — using the victim's own GitHub credentials to host encrypted stolen data — but the branding shift may indicate different sub-teams or an intentional diversification.</p>

###### Persistence and artifacts

<ul role="list">
<li><strong>Dropper:</strong> <code>/tmp/managed.pyz</code></li>
<li><strong>Persistence:</strong> <code>pgsql-monitor.service</code> systemd unit — named to blend with legitimate PostgreSQL monitoring processes</li>
<li><strong>Marker file:</strong> <code>~/.cache/.sys-update-check</code> — singleton lock preventing re-execution</li>
</ul>

###### Indicators of compromise

<p><strong>Compromised versions:</strong></p>
<ul role="list">
<li><code>durabletask==1.4.1</code> — malicious</li>
<li><code>durabletask==1.4.2</code> — malicious</li>
<li><code>durabletask==1.4.3</code> — malicious</li>
<li><code>durabletask==1.4.0</code> — last known clean release</li>
</ul>

<p><strong>File hashes (SHA-256):</strong></p>
<ul role="list">
<li><code>durabletask-1.4.1</code> wheel: <code>7d80b3ef74ad7992b93c31966962612e4e2ceb93e7727cdbd1d2a9af47d44ba8</code></li>
<li><code>durabletask-1.4.1</code> sdist: <code>3de04fe2a76262743ed089efa7115f4508619838e77d60b9a1aab8b20d2cc8bf</code></li>
<li><code>durabletask-1.4.2</code> wheel: <code>aeaf583e20347bf850e2fabdcd6f4982996ba023f8c2cd56bbd299cfd56516f5</code></li>
<li><code>rope.pyz</code> payload: <code>069ac1dc7f7649b76bc72a11ac700f373804bfd81dab7e561157b703999f44ce</code></li>
</ul>

<p><strong>C2 infrastructure (defanged):</strong></p>
<ul role="list">
<li><code>check.git-service[.]com</code> — primary C2 domain (NameSilo, registered 2026-05-16, 3 days before attack)</li>
<li><code>t.m-kosche[.]com</code> — secondary domain, known TeamPCP infrastructure (also used in @antv wave)</li>
<li><code>160.119.64[.]3</code> — IP (AS7489, HostUS)</li>
</ul>

<p><strong>Host artifacts:</strong></p>
<ul role="list">
<li><code>/tmp/managed.pyz</code> — dropper</li>
<li><code>pgsql-monitor.service</code> — systemd persistence unit</li>
<li><code>~/.cache/.sys-update-check</code> — marker file</li>
<li>Unexpected public GitHub repositories named <code>BABA-YAGA</code>, <code>KOSCHEI</code>, <code>FIREBIRD</code>, <code>VASSILISA</code>, or <code>RUSALKA</code> under victim accounts</li>
</ul>

###### Detection and remediation

<ol>
<li><strong>Pin to 1.4.0 immediately.</strong> Run <code>pip show durabletask</code>; if 1.4.1, 1.4.2, or 1.4.3, treat the system as breached.</li>
<li><strong>Treat any import as a full breach.</strong> The payload runs at import time on Linux. Any Python process that imported the malicious package executed the dropper.</li>
<li><strong>Rotate all cloud credentials from affected systems.</strong> AWS access keys, Azure service principal secrets, GCP service account keys, Kubernetes service account tokens, SSH keys, GitHub tokens, npm/PyPI tokens, HashiCorp Vault tokens, and password manager material.</li>
<li><strong>Check AWS CloudTrail for <code>SendCommand</code> API calls</strong> from compromised instances to identify lateral movement targets.</li>
<li><strong>Check Kubernetes audit logs for anomalous <code>kubectl exec</code> activity</strong> across namespaces during the May 19 window.</li>
<li><strong>Audit GitHub for Russian folklore-named repositories</strong> under any account whose tokens may have been exposed.</li>
<li><strong>Remove persistence:</strong> Delete <code>pgsql-monitor.service</code> and <code>~/.cache/.sys-update-check</code>.</li>
<li><strong>Enable PyPI Trusted Publishing.</strong> This prevents direct token-based uploads from outside CI/CD.</li>
</ol>

###### Criminal-market signal

<p>No dark-web presence for durabletask compromise tooling or <code>rope.pyz</code> infrastructure has been observed. The confirmed TeamPCP operator-run pattern documented across the cluster applies. The lateral-movement capability (SSM SendCommand, kubectl exec) is consistent with direct operator use of stolen credentials — a commodity actor would not need to move laterally when the credential collection payload has already harvested everything accessible. H2 (operator-run, no commodity market) is confirmed.</p>
