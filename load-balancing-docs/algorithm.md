# Producing HALO inputs in the core project

This repository supplies valid observations for every source and service container. The cluster network manager turns them into capacities and weights. The solver is specified in the [network algorithm](https://github.com/ECTLab/oakestra-net-load-balancing-project/blob/lb_develop/load-balancing-docs/algorithm.md); this document defines the measurements and their meaning.

## Measurement ownership

```text
Container CPU counters + enforced allocation
Request observations at source and destination
                  |
                  v
NodeEngine HALO collector: one valid window
                  |
                  v
nodes/<worker>/halo/telemetry -> cluster network manager
```

The contract follows the thesis [system model](latex-reference/system-model.tex) and the capacity, all-source counting, and queue sections of [method-cluster.tex](latex-reference/method-cluster.tex).

| Input | Observation point | Consumer |
|---|---|---|
| Allocated container cores | Enforced runtime configuration and effective CPU constraints | Capacity estimate for that destination instance |
| CPU consumption | Cumulative CPU counter for the entire container cgroup | Window delta and linear capacity |
| Attempt starts and outcomes | Logical request source or instrumented request gateway | Source demand and path failure penalty |
| Arrivals and processing completions | Destination container, covering all senders | Observed load and processing capacity |
| Enqueue and service start | Actual application queue | Optional empirical queue correction |
| Ping | NetManager in the companion repository | Path cost; NodeEngine does not derive ping from CPU or request duration |

## Identities

`service_key` is the complete unique service name. An instance is identified by service, instance number, and deployment revision. A host is identified by worker ID, not solely by a mutable IP address.

A source may be just an edge ID. If calling instances on the same edge need separate policies, `source_key` also identifies the caller. An ephemeral connection port is not part of the algorithm's source identity.

An edge without a target-service container may still originate demand for that service. `B2` in the thesis is such a source. It publishes source observations, but no fabricated destination capacity record.

## Window and clocks

The default HALO window is configurable `5s`. Record the actual duration: assuming exactly five seconds for a delayed window produces the wrong rate. UTC timestamps identify the interval and age; a monotonic clock measures elapsed time.

The existing [NodeEngine monitoring cycle](../go_node_engine/internal/daemon/nodeengined.go) is `2s`. Do not silently change it for HALO. The new collector constructs its own window from counter deltas. NetManager's ping period is configured independently.

If the HALO window does not align with legacy samples, averaging several CPU percentages is not a replacement for the exact CPU delta. Read cumulative counters at the two window boundaries, or use valid snapshots with those boundaries.

## Container allocation and CPU consumption

Supply these values for the same container and window:

| Quantity | Definition |
|---|---|
| $g_{si}$ | Reliably supplied runtime allocation in cores; the mathematical model permits fractional shares |
| $T^{CPU}_{si}$ | Cumulative CPU-counter difference, in core-seconds |
| $N^{proc}_{si}$ | Processing completions in that window, from all local and foreign senders |

Mean consumed cores are $\bar c=T^{CPU}/\Delta$ and utilization relative to allocation is $u=T^{CPU}/(g\Delta)$. Consuming `0.5` core gives utilization `0.5` under a `1`-core allocation and `0.25` under a `2`-core allocation. Host-wide CPU percentage is different.

The network manager computes $y=N^{proc}/\Delta$, $\gamma=N^{proc}/T^{CPU}$, and $\mu=\gamma g$. It then applies the margin once: $U=(1-\varepsilon)\mu$. NodeEngine does not subtract the margin from raw measurements or publish a guessed spare capacity instead of them.

Allocated shares must fit the host's usable CPU budget. A cgroup quota alone is not a guaranteed reservation: reliable supply and absence of overcommit also matter. Background-load terms and sequential coupling between services do not replace enforcement of the allocation assumed by the model.

The current `Vcpus` field is an integer. Fractional capacity in the mathematics does not establish fractional support in the current SLA/runtime contract. The first implementation may enforce existing integer shares and report the effective value; fractional SLA support is a separate change.

## Source and destination request counts

At a source, count each started attempt once in `attempts_started`. A retry is a new attempt, distinguishable from its original application request. Receiving that attempt at its destination does not create another source demand.

At the destination, `arrivals` counts all arrivals and `processed` counts processing completions. Include successful and unsuccessful processing that actually ran; exclude rejection before execution. A queued request can have an arrival without a processing completion.

If the destination already has a complete counter, do not add sender-reported totals again. Starts and completions in a single window need not match because attempts cross window boundaries.

Source accounting uses authoritative whole-source `completed_succeeded` and `completed_failed`, including outcomes whose destination could not be attributed. It must satisfy:

$$open_{end}=open_{start}+attempts_{started}-succeeded-failed.$$

Here `succeeded` and `failed` mean those whole-source totals, not the sum of a partial `outcomes[]` breakdown. An outcome without a chosen destination contributes to the failure total, but not to a fictitious instance failure ratio. The wire contract carries it separately. An attempt awaiting a response stays open until its timeout or other terminal event; it is not failed early.

## Outcomes and duration

For a known path, `succeeded` and `failed` refer to outcomes completed during the window. The cluster computes the ratio $F/(C+F)$; started-attempt count is not its denominator. The observation source must define success, application error, timeout, and retry semantics, with a configuration revision.

Measure duration at the source with a monotonic clock, from attempt start to successful or failed completion. A duration sum and completion count are sufficient for the mean; raw per-request records need not be transmitted. Open attempts are not included in completed-duration averages.

No processing-time field is required inside the API response. Request boundaries still need instrumentation. Packet-level networking, particularly with TLS or multiplexing, cannot by itself identify those boundaries or application failures. An application or gateway observation provider is therefore a required integration.

## Queue measurements

`queue_waiting_end` counts waiting requests; `in_service_end` counts requests being processed. An open connection or active request is not necessarily queued. On every service start, add the complete enqueue-to-start duration to the wait sum.

For `service_starts > 0`, mean queue wait is `queue_wait_sum_seconds / service_starts`. A request enqueued in an earlier window contributes its complete wait when it starts service in the current window. This cohort differs from processing completions.

Without queue instrumentation, mark these fields unavailable rather than inventing measured zeros. The network manager omits only the empirical queue residual; the fractional base-cost term remains.

The network manager constructs the $M/M/1$ reference from total destination arrival rate. NodeEngine does not calculate it. Request duration minus ping is not supplied as a queue-wait measurement.

## Validity and container changes

Every container has a deployment revision and CPU-allocation revision. A revision change, counter reset, migration, or agent restart requires a new baseline. Negative CPU deltas, nonfinite values, and zero-duration windows are invalid.

Zero processing and zero CPU can be valid observations, but they are insufficient for a new capacity estimate. The network manager may use a still-valid estimate for the same configuration or a startup measurement. Zero consumption must not produce infinite capacity.

Track validity per measurement. Missing queue data does not invalidate valid CPU data, while valid CPU alone does not replace missing processing counts. Application readiness must come from a valid source, not be equated with a live PID or successful ping.

## Output and the next stage

Publish the window on `nodes/<worker>/halo/telemetry` using NodeEngine's existing MQTT client. This is a proposed new message; the existing `ReportServiceResources` does not produce it. The envelope and arrays are defined in the [low-level contract](skeleton-implementation-low-level.md).

Raw CPU and queue observations go only to the owning cluster network manager. The networking repository subsequently sends spare capacity after local reservation, plus overflow, to the root. Average cluster CPU or the general `15s` resource aggregate is not a substitute for that service-specific summary.

The receiver consumes this normalized telemetry contract. Its weight calculations and forwarding implementation are outside this repository. NodeEngine does not calculate cost functions, derivatives, Lagrange multipliers, or final routing weights.

## Reference inputs and required cases

For the thesis replay, use a `5s` window. Started counts in source order `A1,A2,B1,B2,C1,C2` are `(612,408,180,120,288,192)`. The five destination records contain:

| Destination | Allocated cores | Processing completions | CPU usage, core-seconds |
|---|---:|---:|---:|
| A1 | 2 | 450 | 9 |
| A2 | 1 | 270 | 4.5 |
| B1 | 2 | 285 | 5.7 |
| C1 | 2 | 352 | 7.04 |
| C2 | 1 | 237 | 3.95 |

These are hypothetical inputs. `B2` has only a source record. The consumer must obtain capacities `(100,60,100,100,60)` from these measurements; NodeEngine does not replace the measurements with those estimates. The final matrix and capacity checks are in the [network example](https://github.com/ECTLab/oakestra-net-load-balancing-project/blob/lb_develop/load-balancing-docs/algorithm.md).

Measurements must remain meaningful with no traffic, changed CPU allocation, failed requests, pre-service rejection, cross-window attempts, and container removal. Without a real observation provider, only contract tests are possible; do not report a complete live HALO implementation.

## Exact LaTeX references

Section names below are English translations of the bundled headings. Labels are exact identifiers; source-line links identify the corresponding section in this snapshot. A parent section label is used where a subsection has no separate label. Wire formats, class paths, and startup adapters are repository-specific engineering choices.

| Subject in this document | Exact bundled section | Source labels |
|---|---|---|
| Demand, capacity, and observed load | [Demand, capacity, and observed load](latex-reference/system-model.tex#L33) (system-model.tex:33) | `eq:source-demand` |
| Ping, failures, and outstanding attempts | [Path, failure, and queue observations](latex-reference/system-model.tex#L53) (system-model.tex:53) | `eq:ping-mean`, `eq:failure-observation` |
| Container CPU and processing inputs | [Linear capacity estimation from CPU consumption](latex-reference/method-cluster.tex#L8) (method-cluster.tex:8) | `eq:container-utilization`, `eq:processed-throughput`, `eq:cpu-linear-capacity` |
| Processing from all senders | [Request counts from all sources](latex-reference/method-cluster.tex#L33) (method-cluster.tex:33) | `eq:all-source-observation` |
| Optional destination queue observation | [Fixed path cost and container queue observation](latex-reference/method-cluster.tex#L92) (method-cluster.tex:92) | `eq:queue-measurement`, `eq:queue-residual` |
| Producer replay fixture | [Inputs and calculation order](latex-reference/method-example.tex#L15) (method-example.tex:15) | `tab:example-input-hosts`, `tab:example-input-sources` |
