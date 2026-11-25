# Low-level HALO support plan for the core project

This document maps only this repository's code. Linked files exist; new paths in backticks are proposals. The wire contract matches the [network low-level plan](https://github.com/ECTLab/oakestra-net-load-balancing-project/blob/lb_develop/load-balancing-docs/skeleton-implementation-low-level.md).

## Existing execution map

```text
nodeengined.go
  -> runtime manager
  -> MQTT InitMqtt
  -> StartServicesMonitoring(MONITORING_CYCLE = 2s)
  -> runtime.ResourceMonitoring
  -> model.Resources
  -> ReportServiceResources
  -> nodes/<worker>/jobs/resources
  -> cluster_manager typed models -> resource database
```

| Existing file | Verified state | Required change |
|---|---|---|
| [model/Service.go](../go_node_engine/model/Service.go) | Integer `Vcpus`; string `Resources.Cpu` percentage | Preserve legacy field meanings; add separate HALO types |
| [jobs/ServicesResourcesUpdater.go](../go_node_engine/jobs/ServicesResourcesUpdater.go) | Starts runtime monitoring with a supplied interval | Preserve this loop; HALO has an independent window collector |
| [virtualization/RuntimeDispatcher.go](../go_node_engine/virtualization/RuntimeDispatcher.go) | Constructs runtimes and exposes monitoring | Expose optional HALO snapshot capability |
| [runtime/runtime.go](../go_node_engine/virtualization/internal/runtime/runtime.go) | Existing monitoring interface exposes `ResourceMonitoring` | Add a separate optional capability, not a mandatory method for every runtime |
| [containerd/ContainersManagement.go](../go_node_engine/virtualization/internal/containerd/ContainersManagement.go) | PID/child CPU percentages normalized to the host; no `Vcpus` enforcement in inspected OCI path | Apply/read back HALO-managed CPU allocations; integrate whole-cgroup counters |
| [mqtt/MqttClient.go](../go_node_engine/mqtt/MqttClient.go) | Existing client and `jobs/resources` publication | Add HALO report publication without another client |
| [internal/daemon/nodeengined.go](../go_node_engine/internal/daemon/nodeengined.go) | `2s` monitoring cycle and MQTT startup | Start one HALO collector after identity/MQTT are ready |
| [config/config.go](../go_node_engine/config/config.go) | `/etc/oakestra/conf.json` configuration | Optional HALO settings; validate interval and observation source |

`getTotalCpuUsageByPid` sums child-process percentages and divides by host core count. The fallback also divides by host core count. Neither value is automatically the thesis utilization $u=T^{CPU}/(g\Delta)$, nor an exact measurement over the intended HALO boundaries.

## Proposed new files

| Path | Required content |
|---|---|
| `go_node_engine/model/HALOTelemetry.go` | `HALOTelemetryWindow`, `HALOSourceMetrics`, `HALOOutcome`, `HALODestinationMetrics`, per-measurement validity, and resource snapshots |
| `go_node_engine/jobs/HALOTelemetryCollector.go` | Window/baseline management, request-provider integration, reset checks, and report construction; keep `ObservationProvider` contract here |
| `go_node_engine/virtualization/internal/containerd/HALOMetrics.go` | Whole-container cumulative CPU and effective allocation readback, with instance/revision identity |

These files have not been created. Keep the proposed types and methods in those three locations rather than adding a new telemetry service. Future tests belong beside the relevant implementation.

Do not implement `HALOClusterAlgorithm` or `HALORootAlgorithm` in this repository. Their paths are specified in the [network design](https://github.com/ECTLab/oakestra-net-load-balancing-project/blob/lb_develop/load-balancing-docs/skeleton-implementation-high-level.md). The scheduler's `placement.Algorithm` concerns placement, not HALO traffic distribution.

## CPU allocation in containerd

1. Validate the requested allocation for a HALO-managed service against the host's allocatable CPU budget, after reserving control resources. Check the sum of allocated shares rather than only an individual request.
2. Apply the existing `Service.Vcpus` allocation as an OCI CPU constraint. With quota/period, record their ratio in cores. The period is a runtime setting, not an extra cost-function parameter.
3. Read the effective configuration back from the running container/cgroup. Quota, cpuset, and parent constraints may restrict the effective share; do not report the nominal value above that bound.
4. Read cumulative CPU consumption for the whole container cgroup, not only its main PID or some children. Detect cgroup v1/v2 units and convert to seconds.
5. Attach container identity, deployment revision, and CPU-allocation revision. Changing any of them requires a suitable new baseline; an incompatible old capacity estimate must not silently remain valid.

Quota is an upper bound, not by itself a CPU reservation. The simplified independent-container model assumes a reliably supplied share. Competing-container tests and overcommit checks are required; adding a background-load term would not replace those checks.

The mathematical model permits fractional shares, but the current `Vcpus` field is integer. The first implementation may use enforced integer shares. Fractional SLA support is not an initial requirement and must not be guessed from an integer field.

## Optional runtime capability

Define a proposed `HALOResourceReader` snapshot capability in the runtime contract. The dispatcher exposes it only when supported. Implement containerd first while leaving legacy `ResourceMonitoring` intact.

A snapshot contains cumulative CPU seconds, effective allocated cores, instance identity, deployment/allocation revisions, and validity. The collector differences two snapshots into a window; the runtime and collector must not both difference the same data.

An unsupported runtime reports `unsupported` with a reason. Do not package host CPU or semantically different VM measurements as container CPU. Extending support requires a separate measurement/allocation mapping, without breaking existing unikernel or crosvm deployments.

## Request-observation provider

The proposed `ObservationProvider` must supply observations with explicit window boundaries. It may adapt an actual application or gateway tool, but this checkout has no ready endpoint or provider implementation to assume.

Select a configured measurement source and implement its mapping to the following contract. This is a remaining measurement requirement. An enabled HALO switch alone does not provide exact request counts.

| Category | Required provider output |
|---|---|
| Source demand | Service, `source_key`, started attempts, outstanding counts at both boundaries |
| Path outcomes | Actual known destination identity, successes/failures, completed-duration sum/count |
| Unrouted failures | Failed completions before destination identification, without a fictitious destination |
| Destination processing | Total arrivals and processing completions from all senders |
| Queue, when instrumented | Enqueue-to-start wait sum, service-start count, waiting and in-service counts |
| Validity | Window, observation-definition revision, measurement coverage, and missing-data reason |

Cumulative measurements need an atomic snapshot or a defined cut. Provider window deltas are not differenced again. Specify whether the provider supplies cumulative snapshots or deltas; the final MQTT representation always contains window deltas.

Define application success/error/timeout semantics for each service. An attempt has one terminal outcome: a late response after timeout must not create a second completion. Retries receive new attempt identities while remaining distinguishable from the original application request.

An incoming foreign request contributes to destination `arrivals`/`processed`, not another source demand for the same attempt. If the service calls a different service, that call is a separate source attempt for the called service.

Queue observations are unavailable without instrumentation. Waiting time is not response duration minus ping, and open connections are not queued requests. Source duration measurement does not require an internal processing-time field in the API response.

Path outcomes require correlation with the instance actually selected. A service VIP or service name alone does not reveal the routed destination. If the provider cannot establish that relationship, do not attribute failure to a guessed instance; report unavailable/unobserved path data and retain the age of any valid prior observation.

## Proposed NodeEngine configuration

Add an optional `halo` section to existing configuration. These fields are proposals and are absent from the current `ConfFile`.

| Field | Behavior |
|---|---|
| `enabled` | Disabled by default; preserve legacy behavior when absent |
| `window_seconds` | Default `5`; positive and finite; also report actual window duration |
| `observation_sources` | Bind services/gateways to real providers and versioned outcome definitions |
| `max_observation_age_seconds` | Explicit freshness bound recorded in deployment settings |

Ping configuration belongs to NetManager; do not duplicate `ping_interval_seconds` here. $P_s$, $\kappa_s$, $\varepsilon$, source order, and solver tolerance are network-manager settings, not telemetry-source settings.

Use worker identity from the existing handshake and service identity from deployment data. Start the collector after MQTT is ready and stop it with the daemon. Measurement reads and publishing must not block the deployment-control loop.

## Constructing a window

1. At the beginning boundary, capture CPU and observation-source baselines. A new instance has no preceding valid complete window.
2. At the end boundary, measure actual monotonic duration. Difference CPU/counters only for matching identities and allocation revisions.
3. Detect negative deltas, resets, zero duration, configuration changes, and nonfinite values. Mark affected measurements invalid with reasons rather than inventing zero.
4. Construct source and destination counts separately. Do not add sender outcomes to a complete destination processing counter.
5. Check outstanding-attempt accounting, including failures without a destination. The cluster network manager calculates per-destination failure ratios from source-side completed outcomes, not from destination processing counts.
6. Keep CPU, processing, request, and queue validity separate. Omit only missing optional measurements; do not conceal mandatory capacity-input gaps.
7. Build a message with producer ID, epoch, and sequence; publish with the existing MQTT client. Duplicate reception must not add counts twice.
8. Retain appropriate baselines for the next interval. Do not accumulate raw request histories for delivery to the root.

Publish identity/status even during idle windows, with appropriate empty role arrays. Measured zero differs from a failed provider or absent container. Idle-instance visibility must not depend on NetManager selection count.

## Shared MQTT contract

A new publication function in [MqttClient.go](../go_node_engine/mqtt/MqttClient.go) calls `publishToBroker("halo/telemetry", payload)`. The existing function adds `nodes/<worker>/`, producing `nodes/<worker>/halo/telemetry`, not a `net/` topic.

This message is new. Preserve legacy `jobs/resources` and its `2s` cycle. The companion network manager must add the subscription; changing NodeEngine alone does not activate reception.

| Envelope field | Meaning |
|---|---|
| `schema_version` | Contract version; proposed initial value `1` |
| `producer_id`, `producer_epoch`, `sequence` | Worker identity, execution epoch, and increasing sequence within that epoch |
| `window_start`, `window_end` | UTC observation boundaries |
| `duration_seconds` | Actual positive duration in seconds |
| `source_metrics`, `destination_metrics` | Independent arrays; either may be empty |

Epoch is an opaque identity, not a lexically sortable timestamp. Receivers must recognize restarts/active producers. Sequence numbers order messages within their epoch. Replaying a message preserves its identity.

| Source field | Meaning |
|---|---|
| `service_key`, `source_key` | Full service name and logical source identity |
| `attempts_started`, `open_start`, `open_end` | Demand and open-attempt counts at both boundaries |
| `completed_succeeded`, `completed_failed` | Authoritative totals of every completed attempt, including outcomes with unknown route attribution |
| `duration_sum_seconds`, `duration_count` | Whole-source completed-duration aggregate; optional path aggregates do not replace it |
| `unrouted_failed` | Failed completion without a known destination; whole-source accounting only |
| `outcomes[]` | Known-destination outcome groups |
| Outcome: `cluster_id`, `instance_number`, `host_id`, `deployment_revision` | Actual destination and deployment identity |
| Outcome: `succeeded`, `failed` | Successful and failed completions |
| Outcome: `duration_sum_seconds`, `duration_count` | Duration sum and count for the corresponding completed cohort |
| `validity`, `missing_reason` | Measurement groups: `requests`, `path_outcomes`, and `durations`; outcome-definition revision is recorded in provider configuration |

Source accounting uses `completed_succeeded` and `completed_failed`, not a possibly incomplete path breakdown. `unrouted_failed` is a subset of the failure total and is not added again. Whole-source duration aggregates include terminal attempts independently of route attribution; unknown routes are not assigned to fake instances.

| Destination field | Meaning |
|---|---|
| `service_key`, `instance_number`, `host_id` | Service-container and host identity |
| `deployment_revision`, `cpu_allocation_revision` | Instance execution and CPU-allocation revisions |
| `cpu_allocated_cores`, `cpu_used_core_seconds` | Effective core allocation and window CPU delta |
| `arrivals`, `processed` | All arrivals and actual processing completions from every sender |
| `service_starts`, `queue_wait_sum_seconds` | Service-start cohort and its total queue wait |
| `queue_waiting_end`, `in_service_end` | Waiting and currently served requests at the window end |
| `ready`, `validity`, `missing_reason` | Application readiness and per-measurement validity; queue data may independently be unavailable |

Counts are nonnegative integers; time is seconds and CPU usage is core-seconds. Represent missing fields with explicit status or null, not an unexplained numeric zero. CPU validity must also cover verification of the applied allocation.

The following hypothetical message illustrates source-only edge `B2`. Identifiers are examples; this edge has no destination CPU record:

```json
{
  "schema_version": 1,
  "producer_id": "worker-b2",
  "producer_epoch": "boot-example-1",
  "sequence": 10,
  "window_start": "2026-11-25T09:00:00Z",
  "window_end": "2026-11-25T09:00:05Z",
  "duration_seconds": 5,
  "source_metrics": [
    {
      "service_key": "campus.default.parking.default",
      "source_key": "worker-b2",
      "attempts_started": 120,
      "open_start": 0,
      "open_end": 0,
      "completed_succeeded": 120,
      "completed_failed": 0,
      "unrouted_failed": 0,
      "duration_sum_seconds": 2.4,
      "duration_count": 120,
      "validity": {
        "requests": "valid",
        "path_outcomes": "valid",
        "durations": "valid"
      },
      "outcomes": [
        {
          "cluster_id": "cluster-b",
          "instance_number": 3,
          "host_id": "worker-b1",
          "deployment_revision": "deployment-b1-1",
          "succeeded": 120,
          "failed": 0,
          "duration_sum_seconds": 2.4,
          "duration_count": 120
        }
      ]
    }
  ],
  "destination_metrics": []
}
```

The consumer obtains demand `24` from `120/5`; `B2` remains a source, not a processing destination. The duration value only illustrates the contract and adds no new term to the base objective.

## Component call contracts

These are proposed public boundaries for this repository. They do not move a network solver into NodeEngine.

| Component call | Required input | Output |
|---|---|---|
| Runtime capability `ReadHALOSnapshot` | Current runtime/container inventory | Cumulative CPU seconds, effective cores, instance/deployment/allocation identities, and per-field validity. No rates or weights. |
| Public runtime-manager wrapper | Runtime type and target instances | The model-level snapshot, or explicit unsupported/error result; callers outside `virtualization/internal` do not import private runtime packages. |
| Provider `ReadWindow` | Actual start/end boundaries and configured service/source/instance bindings | A normalized request window with source totals, optional route breakdown, destination processing counts, and optional queues. It contains deltas, not an ambiguous mixture of counters and deltas. |
| Collector window close | Matching CPU boundary snapshots plus provider window | `HALOTelemetryWindow`: identity envelope, two role arrays, and validity groups. No capacity estimate, safety margin, or routing table. |
| MQTT `ReportHALOTelemetry` | One complete telemetry window | Publication outcome using the existing client and `halo/telemetry` suffix. A timeout is not reported as successful delivery. |
| Collector stop | Daemon cancellation | Stop timers/provider reads, release local state, and avoid new publication. It does not stop unrelated deployment services. |

The provider adapter may internally read cumulative counters or an event stream, but must normalize them before returning `ReadWindow`. Its cut must match the requested window. If exact matching is unavailable, mark the affected measurement invalid/unavailable; do not silently combine different durations.

A whole-source total is authoritative even when route attribution fails. Enforce `open_end = open_start + attempts_started - completed_succeeded - completed_failed`. Known-route successes/failures may be a subset of those totals. `unrouted_failed` is already included in `completed_failed`.

With complete route coverage, `sum(outcome.succeeded) = completed_succeeded` and `sum(outcome.failed) + unrouted_failed = completed_failed`. Otherwise, the nonnegative differences are unattributed outcomes. Mark `path_outcomes` unavailable/invalid so the consumer does not derive a biased fresh path-failure ratio. Do not lose valid source demand because route attribution is missing.

For valid complete source-duration coverage, `duration_count = completed_succeeded + completed_failed`. This supports a source mean even when destination attribution is missing. A zero count gives no new mean. Partial or missing duration coverage must not be labeled a complete measured mean.

### Whole-container CPU fixture

For one valid `5s` window, suppose the container CPU counter moves from `100` to `109` core-seconds and its verified allocation remains `2` cores. The CPU delta is `9`, not `109`, and not a percentage of the host. The following destination-only wire message illustrates the normalized result.

```json
{
  "schema_version": 1,
  "producer_id": "worker-a1",
  "producer_epoch": "boot-example-a1",
  "sequence": 10,
  "window_start": "2026-11-25T09:00:00Z",
  "window_end": "2026-11-25T09:00:05Z",
  "duration_seconds": 5,
  "source_metrics": [],
  "destination_metrics": [
    {
      "service_key": "campus.default.parking.default",
      "instance_number": 1,
      "host_id": "worker-a1",
      "deployment_revision": "deployment-a1-1",
      "cpu_allocation_revision": "cpu-a1-1",
      "cpu_allocated_cores": 2,
      "cpu_used_core_seconds": 9,
      "arrivals": 455,
      "processed": 450,
      "service_starts": 452,
      "queue_wait_sum_seconds": 9.04,
      "queue_waiting_end": 5,
      "in_service_end": 2,
      "ready": true,
      "validity": {
        "cpu": "valid",
        "processing": "valid",
        "queue": "valid",
        "readiness": "valid"
      }
    }
  ]
}
```

The CPU and completion values reproduce the thesis input for `A1`. Arrival and queue values here are illustrative provider data, not additional thesis measurements. The collector publishes raw window inputs; it does not publish the derived capacity.

### Validation and lifecycle decisions

| Condition | Required producer behavior |
|---|---|
| Known observed service has no traffic | Emit its explicit zero-count record. Empty role arrays mean the role is absent, not that every omitted service has zero demand. |
| Observation provider missing or failing | Keep identity and report unavailable measurements with reasons. Do not manufacture request/processing counts from flows. |
| CPU input valid but request processing unavailable | Publish their separate validity; the consumer cannot estimate new capacity from CPU alone. |
| Queue unavailable | Null/omit the affected optional values with unavailable status; do not set measured zeros. |
| Container/allocation revision changes mid-window | Invalidate that instance's mixed window and establish a new baseline. Other instances may still have valid records. |
| Observation definition changes | Reset incompatible measurement baselines, version the provider configuration, and invalidate incompatible cached observations before normal publication. |
| MQTT not connected yet | Do not dereference an uninitialized client. Keep bounded pending state or explicitly record a failed publication. |
| Retrying a window | Reuse the same producer/epoch/sequence and body; do not assign a new identity to the same delta. |
| Provider snapshot gives only a PID CPU percentage | Do not label it a whole-container core-second delta. |

Use validity values `valid`, `unavailable`, `invalid`, and `unsupported`. Source groups include `requests`, `path_outcomes`, and `durations`; destination groups include `cpu`, `processing`, `queue`, and `readiness`. Counts are integers, not booleans; durations and CPU/rate values must be finite. Reject JSON NaN/Infinity.

On restart, registration/reconnect must identify the active producer epoch to the telemetry receiver and retire the old one. Sequence numbers alone do not order different opaque epochs. This startup adapter is part of telemetry delivery; it is not implemented by merely adding a new struct.

For concurrent container deployment, allocation bookkeeping must include pending as well as running reservations. Reserve atomically, release on failed deployment/removal, and reconstruct from verified runtime state after restart. This is resource-allocation bookkeeping, not the removed cross-service background term in the HALO cost function.

### Incomplete route-attribution fixture

Three attempts start and complete: two succeed and one fails before destination selection. Only one successful attempt has a known destination. The source totals and mean remain valid, but fresh per-path failure estimation is unavailable.

```json
{
  "schema_version": 1,
  "producer_id": "worker-a1",
  "producer_epoch": "boot-example-a1",
  "sequence": 11,
  "window_start": "2026-11-25T09:00:05Z",
  "window_end": "2026-11-25T09:00:10Z",
  "duration_seconds": 5,
  "source_metrics": [
    {
      "service_key": "campus.default.parking.default",
      "source_key": "worker-a1",
      "attempts_started": 3,
      "open_start": 0,
      "open_end": 0,
      "completed_succeeded": 2,
      "completed_failed": 1,
      "unrouted_failed": 1,
      "duration_sum_seconds": 0.12,
      "duration_count": 3,
      "validity": {
        "requests": "valid",
        "path_outcomes": "unavailable",
        "durations": "valid"
      },
      "missing_reason": {
        "path_outcomes": "One completed successful attempt has no confirmed destination identity."
      },
      "outcomes": [
        {
          "cluster_id": "cluster-b",
          "instance_number": 3,
          "host_id": "worker-b1",
          "deployment_revision": "deployment-b1-1",
          "succeeded": 1,
          "failed": 0,
          "duration_sum_seconds": 0.04,
          "duration_count": 1
        }
      ]
    }
  ],
  "destination_metrics": []
}
```

The demand is `3/5` attempts/s, the complete source-duration mean is `0.12/3` seconds, and outstanding accounting is `0 + 3 - 2 - 1 = 0`. Summing only the known-route outcome would incorrectly leave two attempts open.

## Why core manager models need no mandatory extension

[NodeJobResourceMessage](../cluster_orchestrator/cluster-manager/src/cluster_manager/models/mqtt.py) and [JobInstanceResources](../cluster_orchestrator/cluster-manager/src/cluster_manager/models/job.py) parse legacy messages. [update_instance_resources](../cluster_orchestrator/cluster-manager/src/cluster_manager/clients/job_management.py) then copies selected fields into `JobInstance`.

Consequently, `extra="allow"` at the beginning does not guarantee end-to-end storage of new fields. This plan does not extend that chain for HALO. The dedicated message reaches the network manager while general resource models retain their existing contract.

If a future public API must expose HALO metrics, explicitly update both models and the update function as a separate requirement. That display/storage feature is not needed for the base routing algorithm.

## Deploy the companion network fork

The [root](../root_orchestrator/docker-compose.yml) and [cluster](../cluster_orchestrator/docker-compose.yml) Compose files select official network images. HALO requires images built from the companion checkout and selection through the existing local overrides.

Image names match the existing [root override](../root_orchestrator/override-local-service-manager.yml) and [cluster override](../cluster_orchestrator/override-local-service-manager.yml). These files currently select local images; their existence does not mean those images contain HALO.

After implementation, build from this repository root using the sibling checkout:

```powershell
docker build -t local_root_service_manager ../oakestra-net-load-balancing-project/root-service-manager/service-manager
docker build -t local_cluster_service_manager ../oakestra-net-load-balancing-project/cluster-service-manager/service-manager
```

These commands were not executed for this documentation delivery. A successful image build also does not replace a running service; deploy the selected images and verify their versions.

| Connection | Current Compose setting |
|---|---|
| Network root | `MY_PORT=10099`; cluster peer uses `ROOT_SERVICE_MANAGER_PORT=10099` |
| Cluster network manager | `MY_PORT=10110`; the core cluster manager uses that port |
| Cluster MQTT | `10003`; reuse for HALO publication |
| Network root/cluster Mongo | `10008 / 10108`; distinct from general resource databases |

Proposed `/api/net/halo/capacity` and `/api/net/halo/decision` routes live in the network project on those existing ports. Do not add another service, port, or scheduler here. Host-network deployments need reachable configured addresses.

Keep default Compose behavior unchanged; select images and HALO settings through existing overrides. `override-no-network.yml` is incompatible with running the network part of HALO because it disables the required managers.

## Worker installation and version alignment

[InstallOakestraWorker.sh](../scripts/InstallOakestraWorker.sh) downloads both official release binaries. Running it with an official version does not install this fork's HALO artifacts. Select locally built artifacts for experiments and record both versions.

Build NodeEngine from this repository and NetManager from its companion. This repository does not generate NetManager from NodeEngine code. Align `schema_version`, settings, both binaries, and both manager images for a deployment.

[go.mod](../go_node_engine/go.mod) specifies `go 1.24.0`. Container-runtime builds/tests require the repository's Linux environment and dependencies. Documentation edits do not establish a successful Windows binary test or a running laboratory deployment.

The [root](../scripts/StartOakestraRoot.sh) and [cluster](../scripts/StartOakestraCluster.sh) startup scripts may download official configuration/images. Keep checkout and override selection explicit: selecting this repository's branch does not install the sibling networking checkout automatically.

During implementation, update [SKILLS/troubleshoot-oakestra.md](../SKILLS/troubleshoot-oakestra.md) for new configuration, binary behavior, and the topic. This documentation revision changes the four guides and adds the requested LaTeX reference snapshot; runtime code remains unchanged.

## Future acceptance tests

| Test | Expected result |
|---|---|
| CPU allocation/usage | Read enforced share; account for main process and descendants through the entire cgroup |
| Resource contention | Verify reliable allocation under competing containers/control load; host CPU is not container utilization |
| Independent windows | Preserve legacy `2s`; produce actual `5s` or configured HALO intervals |
| Reset/deployment change | No negative delta or mixed-container window becomes a fabricated capacity |
| Allocation change | New $g$ changes revision/baseline; do not mix observations from different allocations |
| All senders | Count local and foreign processing once at the destination |
| Open attempts | Starts, outcomes, and outstanding counts reconcile across boundaries |
| Failure/retry | One timeout completion, no second late-response completion, independent retry attempts |
| Partial route attribution | Complete source totals reconcile; unknown destinations do not create biased path ratios or false open attempts. |
| Pre-service rejection | Count the appropriate arrival/failure but not nonexistent processing |
| Queue | Waiting separate from in-service; enqueue-to-start duration; missing instrumentation marked unavailable |
| Zero CPU/processing | Report genuine zeros without creating capacity through division by zero |
| Source-only edge | Publish `B2`-like demand without a fake destination record |
| MQTT/restart | Explicit duplicate, stale, disconnected, and new-epoch handling; bounded publication queue |
| HALO disabled | Legacy deployment, reports, and runtimes continue to work |
| Thesis replay | Five capacity-input records and six demand records reproduce the reference inputs without unit changes |
| Deployment | Matching binaries/images; network root `10099`, cluster `10110` |

A two-repository integration test must show that this telemetry reaches the network manager and that the network manager, not NodeEngine, computes capacity and weights. Synthetic observations test a contract; they do not establish a functioning real request provider.

Report flow shares separately from request shares. A persistent connection may send many requests to one destination; collecting exact request counts alone does not fix that datapath limitation. Request-layer prerequisites are in the [network readme](https://github.com/ECTLab/oakestra-net-load-balancing-project/blob/lb_develop/load-balancing-docs/readme.md).

## Exact LaTeX references

Section names below are English translations of the bundled headings. Labels are exact identifiers; source-line links identify the corresponding section in this snapshot. A parent section label is used where a subsection has no separate label. Wire formats, class paths, and startup adapters are repository-specific engineering choices.

| Subject in this document | Exact bundled section | Source labels |
|---|---|---|
| Window and source-demand contract | [Demand, capacity, and observed load](latex-reference/system-model.tex#L33) (system-model.tex:33) | `eq:source-demand` |
| Failure and duration semantics | [Path, failure, and queue observations](latex-reference/system-model.tex#L53) (system-model.tex:53) | `eq:failure-observation` |
| Effective CPU allocation and usage | [Linear capacity estimation from CPU consumption](latex-reference/method-cluster.tex#L8) (method-cluster.tex:8) | `eq:container-utilization`, `eq:cpu-linear-capacity` |
| Destination totals | [Request counts from all sources](latex-reference/method-cluster.tex#L33) (method-cluster.tex:33) | `eq:all-source-observation` |
| Queue-start cohort | [Fixed path cost and container queue observation](latex-reference/method-cluster.tex#L92) (method-cluster.tex:92) | `eq:queue-measurement` |
| Telemetry fixture | [Inputs and calculation order](latex-reference/method-example.tex#L15) (method-example.tex:15) | `tab:example-input-hosts`, `tab:example-input-sources` |
