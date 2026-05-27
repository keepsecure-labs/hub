---
title: "actions-cool/issues-helper GitHub Action compromised through tag poisoning: all tags point to credential-stealing imposter commit"
date: 2026-05-18
heroImage: "/art/actions-cool-issues-helper-tag-poisoning-may-2026.png"
description: "The actions-cool/issues-helper GitHub Action — used by thousands of repositories for automated issue management — was compromised on May 18, 2026 through tag poisoning: an attacker with write access to the repository force-moved every existing Git tag to reference a single malicious imposter commit unreachable from any branch. The payload downloads Bun, scrapes Runner.Worker process memory for workflow secrets, and exfiltrates via HTTPS. Any workflow referencing the action by tag (@v1, @v2, @v3, etc.) pulled the malicious payload on its next run. Only SHA-pinned references are safe."
slug: actions-cool-issues-helper-tag-poisoning-may-2026
type: threat-intel
campaign: null
firstObserved: 2026-05-18
cves: []
targetSectors:
  - Open source maintainers
  - CI/CD infrastructure
  - GitHub Actions workflows
targetRegions:
  - Global
author:
  name: falco365
  github: falco365
tags:
  - threat-intel
  - github-actions
  - supply-chain
  - tag-poisoning
  - CI/CD
  - credential-theft
  - Bun runtime
  - runner-memory-scraping
hasArtifacts: false
diamond:
  adversary:
    operator_model: unknown
    monetization: credential-collection
    confidence: confirmed
  infrastructure:
    registry_layer: "GitHub Actions marketplace (actions-cool/issues-helper)"
    c2_primary: "Attacker-controlled HTTPS domain (not yet publicly identified)"
    c2_fallback: null
    exfil_channel: "HTTPS exfiltration"
    c2_resilience_tier: 1
  capability:
    initial_access: write-access-to-github-actions-repository
    execution: tag-poisoning-imposter-commit
    evasion: imposter-commit-unreachable-from-branch-history
    persistence: null
    post_exploitation: runner-worker-process-memory-scraping
  victim:
    direct: "Repositories referencing actions-cool/issues-helper by tag"
    blast_radius: "Thousands of repositories using tag-based references; any CI workflow secrets available during the compromise window"
sources:
  - title: "StepSecurity: actions-cool/issues-helper GitHub Action Compromised"
    url: "https://www.stepsecurity.io/blog/actions-cool-issues-helper-github-action-compromised-all-tags-point-to-imposter-commit-that-exfiltrates-ci-cd-credentials"
  - title: "Datadog Security Research: actions-cool/issues-helper card"
    url: "https://app.datadoghq.com/security/feed"
---

<p>The <code>actions-cool/issues-helper</code> GitHub Action — used by thousands of repositories for automated GitHub issue management — was compromised on May 18, 2026 through tag poisoning. First reported by StepSecurity, an attacker with write access to the repository force-moved every existing Git tag to reference a single malicious "imposter commit" that does not appear in the repository's normal branch history. Any workflow referencing the action by tag — <code>@v1</code>, <code>@v2</code>, <code>@v3</code>, and all others — pulled the malicious payload on its next run.</p>

<p>This attack follows the identical pattern as the tj-actions/changed-files compromise (CVE-2025-30066), which had significant industry impact. The mechanism is reproducible against any GitHub Action repository where an attacker can obtain write access: no vulnerability in GitHub Actions itself is required.</p>

###### Tag poisoning mechanism

<p>The imposter commit exists in the repository's Git object store — it is a valid Git object — but is unreachable from any branch head. This makes it invisible through normal repository browsing on GitHub: the Commits tab, the file browser, and pull request history will not surface it. The commit can only be fetched by its raw SHA.</p>

<p>By force-moving all existing tags to point to this commit, the attacker ensured that every tag-based reference in every consumer workflow would resolve to the malicious payload on the next run. The attack requires no changes to consumer workflows — the poisoning happens entirely at the source repository.</p>

###### Payload: Bun download + Runner.Worker memory scraping

<p>When a workflow executes the compromised action, the payload performs three steps:</p>

<ol>
<li><strong>Runtime download:</strong> Downloads the <code>bun</code> JavaScript runtime onto the GitHub Actions runner, providing an execution environment outside the standard Node.js Action runtime.</li>
<li><strong>Process memory harvesting:</strong> Reads memory from the <code>Runner.Worker</code> process, which holds the workflow's decrypted secrets at runtime. This technique bypasses standard secret masking — secrets that appear as <code>***</code> in workflow logs are present in plaintext in <code>Runner.Worker</code> memory and can be captured by any process with sufficient privileges on the same runner. Cloud credentials, registry tokens, OIDC tokens, and any other secrets configured for the workflow are exposed.</li>
<li><strong>Credential exfiltration:</strong> Outbound HTTPS call to an attacker-controlled domain with the harvested material.</li>
</ol>

<p>The <code>Runner.Worker</code> memory scraping technique is particularly significant: it indicates the attacker understands the GitHub Actions runner architecture at a deep level and is targeting the one place where all workflow secrets coexist in plaintext. Standard detection focused on <code>env</code> variable access or file-based secret paths will miss this collection vector.</p>

###### Scope: all tag references affected

<p>Because every tag was re-pointed, all tag-based references resolve to the malicious commit. The scope is not limited to a specific version — <code>@v1</code>, <code>@v2</code>, <code>@v3</code>, and any other tag-based reference all point to the same imposter commit. Only workflows pinned to a full 40-character commit SHA of a known-good commit are unaffected.</p>

###### Detection

<ul role="list">
<li><strong>Search your organization's workflows for <code>actions-cool/issues-helper</code>.</strong> Any reference by tag after May 18, 2026 should be treated as having executed the malicious payload.</li>
<li><strong>Review workflow run logs</strong> for unexpected outbound network connections from runs that executed the action by tag.</li>
<li><strong>Check for OIDC token usage</strong> from unexpected sources during the compromise window — the payload can harvest OIDC tokens that enable cloud identity impersonation.</li>
</ul>

###### Remediation

<ol>
<li><strong>Remove all references to <code>actions-cool/issues-helper</code> from workflows immediately.</strong> Comment out or delete the action until the maintainer confirms a verified clean commit.</li>
<li><strong>Rotate all secrets available to any workflow that executed this action by tag reference after May 18, 2026.</strong> This includes cloud provider credentials, container registry tokens, npm/PyPI publish tokens, and any other secrets in the affected repositories.</li>
<li><strong>Audit workflow run logs</strong> for outbound network connections to unexpected domains.</li>
<li><strong>Pin all remaining third-party GitHub Actions to full commit SHAs.</strong> This is the only reliable defense against tag-poisoning attacks. Tags are mutable references. SHA pins are not.</li>
<li><strong>Review OIDC-based keyless authentication workflows</strong> (e.g., <code>aws-actions/configure-aws-credentials</code>) for unauthorized cloud activity during the compromise window.</li>
</ol>

###### Class context: tag poisoning is a repeatable GitHub Actions attack pattern

<p>The tj-actions/changed-files compromise (CVE-2025-30066) established this attack pattern at scale and it is now confirmed repeatable. Tag poisoning requires only write access to the target repository — which can be obtained through stolen tokens, compromised maintainer accounts, or malicious pull request merges. The GitHub Actions ecosystem's convention of pinning by tag rather than SHA is the structural weakness that makes every mutable Action tag a potential delivery vector. Organizations that have not already pinned all third-party Actions to SHA references should treat this as an urgent remediation item independent of any specific campaign.</p>
