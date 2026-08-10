---
date: "2026-05-28"
title: "Kubernetes diagnostic AI agent"
tags: ["AI", "Kubernetes"]
---

Every conversation about AI agents eventually needs a real project behind it, not just a demo. So I set myself a narrow, deliberately unambitious goal: build something that uses an LLM to do a job I already do by hand — reading `kubectl describe`, `kubectl logs`, and `kubectl get events` output to figure out why a pod is broken — and experiment with what it actually takes to make that reliable.

The result is the [K8s Diagnostic Agent](https://github.com/maximemoreillon/k8s-diagnostic-agent): a conversational agent, built on LangChain and served through a Chainlit UI, that investigates a Kubernetes cluster autonomously and explains what it finds in plain language. Describe the symptom — "pods in the `payments` namespace keep restarting" — and it goes and looks, rather than asking you to paste logs into a prompt.

### What it does

The agent runs a standard ReAct loop: reason, call a tool, observe the result, reason again, repeat until it has enough to answer. The tools are read-only wrappers around the Kubernetes Python client — list namespaces, list pods, describe a pod, fetch logs, list deployments, describe a deployment, list events, get node status, list services. Nine tools, no write access, no ability to restart, scale, or delete anything. For a first agent, giving it hands but not opinions felt like the right boundary.

It recognizes the failure patterns that come up constantly in practice — `CrashLoopBackOff`, `ImagePullBackOff`, `OOMKilled`, pods stuck `Pending` because of resource pressure or unschedulable nodes, failed readiness/liveness probes — and the system prompt nudges it toward the same triage order I'd use myself: check events before logs, because events are cheaper and often already tell you the answer.

### The part that actually mattered: context, not intelligence

The interesting engineering problem here wasn't prompting the model to be smart. GPT is already good at diagnosing a `CrashLoopBackOff` once it has the right facts in front of it. The problem was keeping "the right facts" small enough to fit a context window without losing the signal.

Kubernetes objects are enormous. A raw `kubectl describe pod` dump is mostly noise for this purpose — annotations, managed fields, most of the spec. So none of the tools return raw objects. `describe_pod` returns exactly two things: container states/conditions and resource limits, as trimmed JSON. `list_pods` collapses each pod to one line: name, phase, ready count, restart count. `list_events` filters to `Warning` type by default, because in nearly every real incident the warning events are the whole story and the `Normal` ones are scheduling noise. Logs are capped at 100 lines unless asked for more. Conversation history is capped at the last 10 turns.

None of this is sophisticated — it's the same discipline you'd apply writing a status dashboard for a human. But it's the difference between an agent that stays useful over a multi-turn debugging conversation and one that silently degrades as its context fills with YAML it never needed.

### Deployment

It runs in-cluster: a `ServiceAccount` bound to a `ClusterRole` scoped to read-only access on pods, deployments, services, events, nodes, and namespaces, with the Chainlit UI exposed via a `Service`. Locally it falls back to `~/.kube/config` automatically — same code path, no branching logic beyond the config loader trying in-cluster auth first and catching the exception. Docker Hub builds are wired up via GitHub Actions on every push to `main`, so shipping a change is a git push, not a manual build step.

### Why this project, specifically

AI being useful for cluster debugging isn't really in question at this point. What's not needed is this particular agent: an off-the-shelf Kubernetes MCP server already exposes much the same read-only surface to any MCP-compatible client, and reaching for that would be the pragmatic choice if the goal were a production diagnostic tool. Building my own wasn't about filling a gap — it was about experimenting hands-on with how an agent framework, a tool layer, and a chat UI actually fit together, on a problem domain I already know well enough to judge whether the answers it gives are right. Small enough to finish in about two weeks, real enough to deploy internally, open-source enough to point at.

The spec sketched a v2 — Prometheus and Loki tools to answer latency and error-rate questions alongside the structural ones. I'm leaving that on the shelf: the existing Kubernetes MCP server is the better foundation to extend if I actually want that capability day to day, rather than growing this one further. This project did what it was for.

Code is on GitHub: [github.com/maximemoreillon/k8s-diagnostic-agent](https://github.com/maximemoreillon/k8s-diagnostic-agent)
