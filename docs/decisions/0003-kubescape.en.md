# 0003 — Kubescape over Trivy Operator and Falco in the cluster

**In one sentence**: a single operator (Kubescape) covers running-image scanning, posture checks and runtime detection, where the usual combination needs two, which fits a 12 GiB budget.

| Status | Date | Scope |
|---|---|---|
| Accepted | September 2026 | In-cluster vulnerability scanning, posture, runtime detection, runtime VEX |

## Context

Inside the cluster, we need to:

- scan the images that are running;
- check the configuration (CIS controls);
- detect abnormal behaviour at runtime (**a shell opened in a container must raise an alert**);
- ideally, prove that a vulnerable package is never loaded.

The classic pair is Trivy Operator (scanning, posture) plus Falco (runtime rules): two operators, two agents, two rule languages, on a single node capped at 12 GiB.

## Decision

The Kubescape operator replaces both:

- scanning of running images with Grype;
- CIS and NSA controls;
- runtime detection by the eBPF `node-agent`, which compares each container's behaviour with a profile (learned, or hand-written as for `runtime-demo`); alerts are JSON log lines collected by VictoriaLogs;
- runtime OpenVEX documents (experimental) listing packages never loaded, forwarded by `dt-bridge` to Dependency-Track.

## Alternatives set aside

- **Trivy Operator + Falco**: two complete stacks for the same coverage, too heavy for the memory budget.
- **Falco alone**: excellent at runtime, but covers neither scanning nor posture.

## Trade-offs and cost

- The cluster scanner (Grype) is not the CI and Harbor scanner (Trivy): their CVE lists diverge. Dependency-Track decides ([ADR 0008](0008-divergence-grype-trivy.md)).
- Runtime VEX generation is experimental: about two minutes of observation per container, and inconsistent image names (`docker.io` versus `index.docker.io`) that `dt-bridge` normalises.
- A runtime rule stays silent until the container's profile is loaded. A learned profile only exists after the learning period; a hand-written one exists at once. Hence the hand-written profile for the demo.
- Falco's large community rule set is lost. Kubescape's default rules cover the demo cases.

## What would make me change my mind

A need for many fine-grained runtime rules (PCI compliance, for instance): Falco would become relevant again, even at the memory cost.

## In the code

- [`platform/apps/templates/kubescape.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/kubescape.yaml)
- [`platform/workloads/runtime-demo`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/platform/workloads/runtime-demo)
- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
