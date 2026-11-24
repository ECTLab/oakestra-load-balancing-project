# HALO support guide for the core Oakestra project

These four documents specify this repository's HALO responsibilities: valid NodeEngine telemetry, container CPU allocation, and deployment of matching network-manager images. Weight optimization and WRR belong to the companion networking project; the placement scheduler is not their execution point.

## Baseline

| Item | Baseline |
|---|---|
| Repository | `oakestra-load-balancing-project` |
| Inspected commit | `4081defa67d99f2467fead72aa75c245b1a7b6cc` |
| Review date | `2026-11-24` |
| Algorithm source | Thesis proposed-method chapter: base local cost and spare-capacity root method |
| Delivery status | Documentation only; no HALO implementation or runtime test has been added |

The primary source is [local chapter3.tex](<../../../../Dissertation/HALO/latex/قالب پایان نامه امیرکبیر/Latex template 2018/chapter3.tex>). Observation semantics come from [local system-model.tex](<../../../../Dissertation/HALO/latex/قالب پایان نامه امیرکبیر/Latex template 2018/system-model.tex>); capacity and queue definitions come from [local method-cluster.tex](<../../../../Dissertation/HALO/latex/قالب پایان نامه امیرکبیر/Latex template 2018/method-cluster.tex>).

## Reading order

Cross-project links use the two repositories' `lb_develop` branches so they also work on GitHub. Thesis links are local-only references into the sibling `Dissertation` workspace; neither repository contains those LaTeX files. Their equations and reference example are reproduced in the companion algorithm document, while opening a thesis-source link requires the original workspace layout.

| Existing document | Purpose |
|---|---|
| [algorithm.md](algorithm.md) | What this project measures and supplies, and where weight computation begins |
| [skeleton-implementation-high-level.md](skeleton-implementation-high-level.md) | Data flow, NodeEngine ownership, independent HALO monitoring, and deployment |
| [skeleton-implementation-low-level.md](skeleton-implementation-low-level.md) | Existing and proposed files, telemetry contract, configuration, and acceptance checks |

The complete cost function, Lagrange conditions, root allocation, integration, and worked example are in the [network algorithm document](https://github.com/ECTLab/oakestra-net-load-balancing-project/blob/lb_develop/load-balancing-docs/algorithm.md). This guide does not duplicate that solver inside the core orchestrators.

## Repository boundary

```text
This repository:
  NodeEngine -> CPU allocation/counters -> HALO telemetry -> MQTT
  Compose / installation -> matching network-manager images and worker binaries

Companion networking repository:
  cluster solver <-> root shares -> final table -> edge WRR
```

| Core component | HALO responsibility |
|---|---|
| NodeEngine | Read actual container CPU allocation/usage and valid request observations; publish window summaries |
| `cluster_manager` | Continue lifecycle and resource management; do not host the network cost solver |
| `system_manager` | Continue service and cluster management; do not host the network root-share calculator |
| Compose and worker installation | Select compatible images and binaries built from the intended checkouts |
| Scheduler and resource-abstractor | Preserve placement and resource responsibilities; do not register HALO routing as a placement algorithm |

This repository's Compose files declare the network managers, but their implementation lives in the [network repository](https://github.com/ECTLab/oakestra-net-load-balancing-project/blob/lb_develop/load-balancing-docs/readme.md). That guide also specifies the proposed cluster/root class paths.

## Actual gaps in this checkout

[Resources](../go_node_engine/model/Service.go) currently carries `cpu_percent` as a string. The containerd monitoring implementation normalizes CPU usage by host core count, not by the container's allocated share. HALO needs a cumulative CPU-counter delta and the actual allocation for that container.

`Service.Vcpus` exists, but the inspected OCI construction path does not apply that field as a CPU constraint. An SLA value alone does not establish an enforced runtime allocation; enforcement and readback must be completed and tested.

The existing resource report has no exact request count, request outcome, or queue-wait measurement. Connections and packets are not substitutes. An application or request-gateway observation provider must be integrated. Opaque encrypted traffic without such instrumentation cannot supply the exact model inputs from the current data alone.

## File and change boundary

Only these four existing documents are populated; no additional documentation file is needed. Proposed implementation paths are future work and have not been created. HALO telemetry uses a dedicated message over the existing MQTT connection, without making legacy resource models carry raw routing observations.
