---
title: "The AI agent as confused deputy: a 2026 attack class"
date: 2026-04-29
heroImage: "/art/ai-agent-confused-deputy-pattern.png"
description: "Four recent CVEs reveal the AI agent as confused deputy: privileged process, attacker-controlled input. The class named, mapped, and defended."
author:
  name: falco365
  github: falco365
tags:
  - AI agents
  - confused deputy
  - threat modeling
  - capability security
  - isolation
  - security architecture
---

<p>A confused deputy is a program that holds privileges, accepts requests from a less-privileged caller, and gets tricked into using its privileges on the caller's behalf. The original example dates from 1988. In 2026, the deputy is the AI coding agent — and a recent cluster of CVEs makes the class concrete.</p>

###### The pattern

<p>The unprivileged caller is the input the agent operates on: a GitHub repository, a user prompt, a downloaded model file, a <code>package.json</code>. The privileges the deputy holds are the things that make the agent useful in the first place — cloud credentials, the ability to run shell commands, the ability to publish packages, the ability to load and execute model weights. The same property that makes agents productive makes them a confused-deputy magnet.</p>

<blockquote>The agent is not malicious. The agent is doing its job. It got handed an input, and the input told it to do something the agent's privileges allowed.</blockquote>

###### CVE-2026-34040 — Docker authorization plugin bypass

<ul role="list">
<li><strong>Deputy</strong> — the AI coding agent's Docker daemon.</li>
<li><strong>Attacker-influenced input</strong> — a malicious GitHub repository the agent has cloned and is operating on.</li>
<li><strong>Privileges held</strong> — Docker API access, cloud credentials mounted at container start, kubeconfig.</li>
<li><strong>Confusion mechanism</strong> — the malicious repository contains instructions, in README, devcontainer config, or postinstall hook, that cause the agent itself to execute a crafted Docker API call that bypasses the authorization plugin.</li>
</ul>

<p>The pattern in its purest form. The agent is doing what it was asked to do; the request was attacker-controlled.</p>

###### CVE-2026-7733 — LangChain PythonREPLTool escape

<ul role="list">
<li><strong>Deputy</strong> — the LangChain <code>PythonREPLTool</code>.</li>
<li><strong>Attacker-influenced input</strong> — a prompt that reaches the REPL through the agent's normal control flow.</li>
<li><strong>Privileges held</strong> — Python execution in the agent's process.</li>
<li><strong>Confusion mechanism</strong> — a missing check on <code>__import__</code> lets a prompt-controlled call break out of the documented sandbox.</li>
</ul>

<p>The REPL was designed under the assumption that prompts are bounded data. They aren't. They are code-equivalent input that should not have been treated as data in the first place — true of every prompt-driven tool, not just this one. The CVE is the specific bug; the class is "tools designed under the wrong threat model."</p>

###### CVE-2026-12091 — npm postinstall maintainer takeover

<ul role="list">
<li><strong>Deputy</strong> — <code>npm install</code>.</li>
<li><strong>Attacker-influenced input</strong> — a corrupted version of a legitimate package, signed by the legitimate maintainer because the maintainer's account was taken over.</li>
<li><strong>Privileges held</strong> — code execution at install time, often as the developer user with full env-var, SSH-key, and git-credential access.</li>
<li><strong>Confusion mechanism</strong> — there isn't one. <code>postinstall</code> does what <code>postinstall</code> always did. The change is who controls the script.</li>
</ul>

<p>The cleanest mapping. The deputy doesn't have a bug. The pattern is the bug — running attacker-controlled code with developer privileges at every install. Patching <code>npm</code> doesn't help; the fix has to be structural.</p>

###### CVE-2026-5760 — SGLang malicious GGUF

<ul role="list">
<li><strong>Deputy</strong> — the GGUF model loader.</li>
<li><strong>Attacker-influenced input</strong> — a downloaded weight file.</li>
<li><strong>Privileges held</strong> — code execution in the inference server's process.</li>
<li><strong>Confusion mechanism</strong> — parser flaw lets a malicious model file achieve RCE via the loader.</li>
</ul>

<p>The trust boundary used to be "downloaded binary." Anyone running an inference server understood that running attacker-supplied binaries was dangerous. The new boundary is "downloaded weights" — and that boundary moved without anyone announcing it. Most operators still treat GGUF files as data, not code-equivalent.</p>

###### Why 2026 specifically

<p>The confused-deputy pattern is forty years old. What changed is two things, both about privilege.</p>

<p>First, privilege-bearing agents are now common. Two years ago, an AI agent was a chatbot in a sandbox with no real authority. Today, agents routinely hold long-lived cloud credentials, GitHub publish tokens, npm publish tokens, kubeconfigs, and the ability to write to file paths that matter. Agents are useful because of those privileges. They are dangerous because of them too.</p>

<p>Second, agent input surfaces are inherently untrusted. Agents are designed to work on user-supplied inputs — GitHub repositories from anywhere, prompts from anywhere, model weights downloaded from anywhere. The agent's job is to accept those inputs and act on them. That makes every untrusted-input boundary in the stack a confused-deputy candidate by default.</p>

<p>Combine the two and the picture is clear. An entire generation of infrastructure has been built on the assumption that "running in a Docker sandbox" was sufficient isolation, paired with input surfaces that include arbitrary attacker-controlled code and prompts. The four CVEs are not unusual events. They are the predictable consequence.</p>

###### Privilege narrowing is the structural fix

<p>The single biggest mistake in agent security architecture is treating "isolation" as a binary property of the runtime. It isn't. It's a property of the credential and capability surface the agent reaches into. Long-lived admin credentials mounted into agent sandboxes are an anti-pattern; the mounted credentials become the privilege surface that a successful confused-deputy attack reaches.</p>

<ul role="list">
<li><strong>Short-lived, scope-narrowed tokens issued per-task.</strong> Generate the token when the user dispatches the task; revoke it when the task completes.</li>
<li><strong>Tokens that can do only what the task needs.</strong> Per-task IAM roles in cloud, per-task GitHub fine-grained PATs, per-task npm publish tokens scoped to a single package.</li>
<li><strong>Read-only by default.</strong> Make write privileges an explicit, narrow exception.</li>
</ul>

###### Isolation that survives agent subversion

<p>Shared Docker on a host with cloud credentials is not a security boundary for untrusted code. CVE-2026-34040 makes this explicit, but it was true before the bug was disclosed.</p>

<ul role="list">
<li><strong>Per-task isolation domains</strong> — Firecracker microVMs, gVisor sandboxes, per-task Kubernetes pods with their own IAM principal.</li>
<li><strong>Design the isolation domain as the unit of compromise.</strong> A fully compromised domain still cannot reach anything you care about.</li>
<li><strong>Stepping-stone pattern for production</strong> — agent → tightly-scoped intermediary service → production. The intermediary enforces the policy the agent doesn't.</li>
</ul>

###### Treat all agent inputs as untrusted code-equivalent

<p>A repository from an internal user is not safer than one from an external user, in security terms. The internal user might have been phished. Their account might have been taken over. An attacker-controlled <code>package.json</code> looks identical regardless of who pushed it.</p>

<ul role="list">
<li><strong>No per-source allowlisting</strong> based on "trusted internal." Every input flows through the same untrusted-input pipeline.</li>
<li><strong>Static analysis on incoming repositories</strong> before agent invocation — <code>package.json</code> install hooks, dev-container configs, GitHub Actions workflows, <code>.cursorrules</code> / <code>.windsurfrules</code>, and any file the agent will read as instructions.</li>
<li><strong>Prompt input is code-equivalent.</strong> If a prompt can reach a tool that executes code, the prompt is an RCE surface.</li>
</ul>

###### Detect confused-deputy attempts even when patched

<p>The patch fixes the specific bug. The pattern continues. Detection has to be runtime, not advisory-driven.</p>

<ul role="list">
<li><strong>Log every privileged operation</strong> the agent performs and check it against the user-issued task. Cloud-API calls outside the task scope are evidence of confused deputy.</li>
<li><strong>Watch for credential-path reads</strong> — <code>~/.ssh</code>, <code>~/.aws</code>, <code>~/.npmrc</code>, kubeconfig paths — by agent processes. Almost never legitimate.</li>
<li><strong>Egress monitoring</strong> on agent isolation domains. Outbound traffic to anything other than the package registry, model registry, and explicitly-allowed APIs is suspicious.</li>
</ul>

###### What this means going forward

<p>The confused-deputy class is going to grow. Every new tool that agents can invoke is a candidate. Every new credential type that gets mounted into agent runtimes is a candidate. The question for security teams is not whether their agents will be used as deputies — they will. The question is whether the privilege surface the deputy can reach is narrow enough that a successful confusion is recoverable.</p>

<p>Capability-based design is forty years old too. It came back into fashion at the right time.</p>
