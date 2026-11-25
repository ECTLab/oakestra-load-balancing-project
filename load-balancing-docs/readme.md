# HALO support guide for the core Oakestra project

These four documents specify this repository's HALO responsibilities: valid NodeEngine telemetry, container CPU allocation, and deployment of matching network-manager images. Weight optimization and WRR belong to the companion networking project; the placement scheduler is not their execution point.

## Baseline

| Item | Baseline |
|---|---|
| Repository | `oakestra-load-balancing-project` |
| Inspected commit | `4081defa67d99f2467fead72aa75c245b1a7b6cc` |
| Review date | `2026-11-25` |
| Algorithm source | Thesis proposed-method chapter: base local cost and spare-capacity root method |
| Delivery status | Documentation only; no HALO implementation or runtime test has been added |

The primary source is [chapter3.tex](latex-reference/chapter3.tex). Observation semantics come from [system-model.tex](latex-reference/system-model.tex); capacity and queue definitions come from [method-cluster.tex](latex-reference/method-cluster.tex).

## Reading order

Cross-project links identify only the shared interface or the other repository's ownership. The latest LaTeX sources, required figures, matching PDF, and integrity manifest are now bundled in [latex-reference](latex-reference/README.txt). Source links stay inside this repository; a separate Dissertation checkout is no longer required. The reference tables use English translations of section headings, exact LaTeX labels, and source-line links.

Reference entry points: [LaTeX master](latex-reference/AUTthesis.tex), [matching PDF](latex-reference/AUTthesis.pdf), and [source-integrity manifest](latex-reference/source-manifest.json).

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

The four existing English guides are retained, with the requested `latex-reference` snapshot added beside them. No HALO runtime implementation is created. Proposed paths remain future work. The dedicated telemetry message avoids requiring legacy resource models to carry routing observations.

## Self-contained implementation boundary

An implementer can write the collector, runtime snapshot extension, telemetry publisher, and deployment integration from these guides without reading the thesis. The low-level plan specifies provider input, normalized output, field validity, concrete message fixtures, and acceptance checks. It does not reproduce the companion repository's solver implementation.

The actual application/gateway observation adapter and deployment addresses must still be supplied for the selected environment. The contract states exactly what that adapter must return; it does not pretend that the current code can count arbitrary encrypted application requests.

## Exact LaTeX references

Section names below are English translations of the bundled headings. Labels are exact identifiers; source-line links identify the corresponding section in this snapshot. A parent section label is used where a subsection has no separate label. Wire formats, class paths, and startup adapters are repository-specific engineering choices.

| Subject in this document | Exact bundled section | Source labels |
|---|---|---|
| Source/container ownership and required observations | [Components, services, and data boundaries](latex-reference/system-model.tex#L19) (system-model.tex:19) | `sec:system-model` |
| Source/container ownership and required observations | [Demand, capacity, and observed load](latex-reference/system-model.tex#L33) (system-model.tex:33) | `eq:source-demand` |
| CPU allocation and linear-capacity inputs | [Linear capacity estimation from CPU consumption](latex-reference/method-cluster.tex#L8) (method-cluster.tex:8) | `eq:container-utilization`, `eq:cpu-linear-capacity` |
| Destination counting and queue inputs | [Request counts from all sources](latex-reference/method-cluster.tex#L33) (method-cluster.tex:33) | `eq:all-source-observation` |
| Destination counting and queue inputs | [Fixed path cost and container queue observation](latex-reference/method-cluster.tex#L92) (method-cluster.tex:92) | `eq:queue-measurement` |
