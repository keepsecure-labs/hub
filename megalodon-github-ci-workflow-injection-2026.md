---
title: "Megalodon: automated campaign backdoors 5,500+ GitHub repositories through CI workflow injection"
date: 2026-05-22
heroImage: "/art/megalodon-github-ci-workflow-injection-2026.png"
description: "Megalodon is an automated supply chain campaign that pushed 5,718 malicious commits to over 5,500 GitHub repositories in a six-hour window on May 18, 2026, injecting GitHub Actions workflows that harvest CI secrets, cloud credentials, SSH keys, OIDC tokens, and source code secrets and exfiltrate them to a C2 server. The compromise propagated downstream into the npm package @tiledesk/tiledesk-server (versions 2.18.6–2.18.12)."
slug: megalodon-github-ci-workflow-injection-2026
type: threat-intel
campaign: Megalodon
firstObserved: 2026-05-18
cves: []
targetSectors:
  - Software development
  - Open source maintainers
  - CI/CD infrastructure
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - github-actions
  - supply-chain
  - ci-cd
  - credential-theft
  - npm
  - megalodon
  - tiledesk
  - oidc
hasArtifacts: true
diamond:
  adversary:
    operator_model: operator-run
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "GitHub (compromised PATs/deploy keys); npm (@tiledesk/tiledesk-server downstream)"
    c2_primary: "216.126.225[.]129:8443"
    c2_fallback: null
    exfil_channel: "HTTPS POST to 216.126.225[.]129:8443"
    c2_resilience_tier: 1
  capability:
    initial_access: compromised-github-pat-or-deploy-key
    execution: base64-decoded-bash-via-github-actions-workflow
    evasion: forged-author-identity-routine-commit-messages
    persistence: null
    post_exploitation: aws-gcp-azure-imds-oidc-token-theft
  victim:
    direct: "GitHub repository maintainers with write-accessible PATs or deploy keys"
    blast_radius: "5,500+ repositories across GitHub; downstream npm installs of @tiledesk/tiledesk-server 2.18.6–2.18.12"
sources:
  - title: "Megalodon: Mass GitHub Repo Backdooring via CI Workflows"
    url: "https://safedep.io/megalodon-mass-github-repo-backdooring-ci-workflows/"
  - title: "Datadog Security Research: Megalodon card"
    url: "https://app.datadoghq.com/security/feed"
---

<p>Megalodon is an automated supply chain campaign that pushed 5,718 malicious commits to over 5,500 GitHub repositories in a six-hour window (approximately 11:36 to 17:48 UTC on May 18, 2026), at a rate of roughly 15 repositories per minute. First reported by SafeDep, the campaign used compromised personal access tokens (PATs) or deploy keys to push GitHub Actions workflows containing base64-encoded bash payloads that harvest CI secrets, cloud credentials, OIDC tokens, SSH keys, and source code secrets, then exfiltrate everything to <code>216.126.225[.]129:8443</code>.</p>

<p>The Tiledesk project was a downstream victim: commit <code>acac5a9</code> replaced a Docker build workflow in the <code>tiledesk-server</code> repository via the targeted <code>Optimize-Build</code> variant, and the legitimate maintainer subsequently published <code>@tiledesk/tiledesk-server</code> versions 2.18.6 through 2.18.12 to npm from the poisoned source without realizing it.</p>

<p>Megalodon is a distinct campaign from TeamPCP/Mini Shai-Hulud. The C2 infrastructure, payload mechanism, and attribution markers are different. This is a separate actor — or at minimum a separate operation — using GitHub-native delivery rather than package-registry compromise.</p>

###### Two payload variants

<p>Both variants request <code>id-token: write</code>, <code>actions: read</code>, and <code>contents: read</code> permissions, and execute a <code>set +e; echo "..." | base64 -d | bash</code> one-liner.</p>

<ul role="list">
<li><strong>Mass variant (SysDiag):</strong> Adds a new workflow at <code>.github/workflows/ci.yml</code> named <code>SysDiag</code>. Triggers on <code>push</code> across all branches and on <code>pull_request_target</code> — the latter is particularly dangerous because it runs in the context of the base repository with full secrets access even for pull requests originating from forks. This is the variant responsible for the bulk of the 5,718 commits.</li>
<li><strong>Targeted variant (Optimize-Build):</strong> Replaces an existing workflow file, renaming it to <code>Optimize-Build</code> with a <code>workflow_dispatch</code> trigger. Creates a dormant backdoor the attacker can invoke on demand through the GitHub API. Used against Tiledesk.</li>
</ul>

###### Credential harvesting scope

<p>The decoded payload performs comprehensive collection from every CI runner that executes it:</p>

<ul role="list">
<li>All CI environment variables, <code>/proc/*/environ</code>, and PID 1 environment</li>
<li>AWS per-profile access keys and session tokens via the <code>aws</code> CLI, plus IMDSv2 instance role credentials</li>
<li>GCP access tokens via <code>gcloud auth print-access-token</code> and GCP metadata service queries</li>
<li>Azure IMDS endpoint queries for instance role credentials</li>
<li>GitHub Actions OIDC token request URL and token, enabling cloud identity impersonation without long-lived credentials</li>
<li><code>GITHUB_TOKEN</code>, GitLab CI/CD tokens, and Bitbucket tokens</li>
<li>SSH private keys, Docker auth configs, <code>.npmrc</code>, <code>.netrc</code>, Kubernetes configs, Vault tokens, Terraform credentials, shell history</li>
<li>Regex-based grep scan of source code for 30+ secret patterns (API keys, database connection strings, JWTs, PEM private keys)</li>
<li><code>.env</code> files, <code>credentials.json</code>, <code>service-account.json</code>, and other configuration files across the workspace and common server paths</li>
</ul>

###### Forged identity tradecraft

<p>The attacker rotated through four forged author names (<code>build-bot</code>, <code>auto-ci</code>, <code>ci-bot</code>, <code>pipeline-bot</code>) with two generic email addresses, using <code>git config</code> to forge author identity before pushing. Commit messages were chosen to mimic routine CI maintenance: <code>ci: add build optimization step</code>, <code>chore: optimize pipeline runtime</code>, <code>fix: correct build workflow</code>. The six-hour window and 15-repos-per-minute rate indicate full automation — no human in the loop per-repository.</p>

###### Downstream npm impact: @tiledesk/tiledesk-server

<p>The targeted <code>Optimize-Build</code> variant replaced a Docker build workflow in the <code>tiledesk-server</code> repository (commit <code>acac5a9</code>). The legitimate Tiledesk maintainer then published seven npm versions (2.18.6 through 2.18.12) from the poisoned source. Any application depending on <code>@tiledesk/tiledesk-server</code> that updated within that window installed a version built from a backdoored repository. The CI workflow would have executed the credential-harvesting payload during the build process, meaning Tiledesk's own CI secrets were likely exfiltrated before the poisoned package reached npm consumers.</p>

<p>Pin to <code>@tiledesk/tiledesk-server@2.18.5</code> or earlier until the Tiledesk team confirms a clean release.</p>

###### Detection

<ul role="list">
<li><strong>Search repositories for commits authored by <code>build-system@noreply.dev</code> or <code>ci-bot@automated.dev</code>.</strong> Both email addresses are exclusively associated with Megalodon activity.</li>
<li><strong>Inspect <code>.github/workflows/</code> for workflows named <code>SysDiag</code> or <code>Optimize-Build</code>.</strong> Neither name has legitimate standing in most codebases.</li>
<li><strong>Audit <code>.github/workflows/</code> for any workflow containing <code>base64 -d | bash</code>.</strong> This is not a pattern used by legitimate CI workflows.</li>
<li><strong>Review git logs for commits from <code>build-bot</code>, <code>auto-ci</code>, <code>ci-bot</code>, or <code>pipeline-bot</code>.</strong></li>
<li><strong>Search GitHub audit logs for workflow runs making outbound connections to <code>216.126.225[.]129</code> on port 8443.</strong></li>
</ul>

###### Indicators of compromise

<p><strong>C2 endpoint (defanged):</strong> <code>216.126.225[.]129:8443</code></p>

<p><strong>Forged author emails:</strong> <code>build-system@noreply.dev</code>, <code>ci-bot@automated.dev</code></p>

<p><strong>Forged author names:</strong> <code>build-bot</code>, <code>auto-ci</code>, <code>ci-bot</code>, <code>pipeline-bot</code></p>

<p><strong>Malicious workflow names:</strong> <code>SysDiag</code> (mass variant), <code>Optimize-Build</code> (targeted variant)</p>

<p><strong>Mass variant workflow path:</strong> <code>.github/workflows/ci.yml</code>, triggers on <code>push</code> (all branches) and <code>pull_request_target</code></p>

<p><strong>Tiledesk orphan commit:</strong> <code>acac5a9</code></p>

<p><strong>Compromised npm package:</strong> <code>@tiledesk/tiledesk-server</code> versions 2.18.6–2.18.12 (pin to 2.18.5 or earlier)</p>

<p><strong>Campaign identifier string:</strong> <code>megalodon</code></p>

###### Criminal-market signal

<p>No dark-web presence for Megalodon tooling, infrastructure, or the <code>216.126.225[.]129</code> C2 IP has been observed. The automated, high-volume pattern (5,500+ repos in six hours) is more consistent with operator-run credential collection than commodity marketplace activity. The absence of a public claim — unlike TeamPCP's self-attribution via <code>@pcpcats</code> and Tor-linked disclosures — leaves attribution confidence at the infrastructure level only. H2 (operator-run, no commodity market) is the most probable hypothesis pending further analysis.</p>
