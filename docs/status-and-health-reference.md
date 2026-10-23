# Status and health reference

Every half writes `status_<world>.json` in its `ipc.dir` every 5 seconds. This page is the human contract for the values in that file that a dashboard, a supervisor or a script branches on: `discovery.state`, the health verdict and its reasons, the lane states, and every other value list the file carries. The machine contract is `schema/status.schema.json` (status schema 3); a unit test holds every list on this page equal to the schema's and checks that every reason the engine writes appears here, so the page and the code cannot drift apart.

## The promise

- A value on this page is never removed or renamed within a major version of the engine.
- A new value is announced in the CHANGELOG one release before a consumer must handle it. Until then a consumer that meets a value it does not know should treat it as the least healthy reading of that field.
- New fields are additive under status schema 3. A field whose meaning changes gets a new name. One exception, announced in the 1.26.0 CHANGELOG: the delivery-age fields and `max_gap_us` read null instead of 0 for a window without samples (below).
- The numbers a tool computes with (the budget formulas, the schemas, the protocol) are a separate contract: `airlock_half --constants`, described in [the constants contract](constants-contract.md).

## Discovery

`discovery.state` says what an empty or a full `tx` list means: whether this half can see the sources it is configured for.

```yaml
# status: discovery.state
- live
- watching
- blind
- n/a
```

| Value | Meaning |
| -- | -- |
| `live` | At least one configured source entry has been discovered in this half's world. |
| `watching` | Nothing configured has matched yet, and the half sees other nodes in its world: the lazy, healthy reading of an empty `tx`. |
| `blind` | Nothing matched and nobody is visible past the grace window, or a participant rebuild has not yet been cleared by evidence. The half rebuilds its participant on a ladder; health reads `dysfunctional`. |
| `n/a` | A destination-only half: it has no source entries, so it has nothing to discover. |

`discovery.last_poll_s` is the other half of the question: a stalled graph loop leaves it climbing while every other field still looks fresh.

`discovery.cross_check.verdict` is the second participant's word on what the half's own participant misses:

```yaml
# status: discovery.cross_check.verdict
- ok
- deaf
- undelivered
- newcomer_blind
- error
- retired
```

`shm_discovery.state` reports the Fast DDS shared-memory discovery port of the world (`closed` is the world closed to newcomers):

```yaml
# status: shm_discovery.state
- ok
- closed
- unknown
- not_applicable
```

## Health

`health.verdict` is the engine's own roll-up of every rung, so a consumer reads one value instead of re-deriving the rules.

```yaml
# status: health.verdict
- healthy
- degraded
- dysfunctional
```

- `healthy`: nothing below applies.
- `degraded`: the wall works but is not converged or not as asked.
- `dysfunctional`: the wall is not carrying what it should.

A non-healthy verdict carries `health.reason`, the first rung that applies in the order below. The reason is text for a person; where it names a lane, a service, a number or a peer, the angle brackets below stand for that part. Transients (the peer wait at start, a reload's divergence window, an activation that takes seconds) pass through the lower verdicts honestly, so poll with a time budget rather than judging one snapshot.

### Dysfunctional, in rung order

01. The peer speaks too old a protocol: `proto_minor: the peer '<world>' speaks protocol 1.<minor>, below this half's floor 1.<floor> (ipc.min_peer_minor) — run the same image on both halves (protocol 1.8 only since 1.24.0)`. The `(ipc.min_peer_minor)` part appears only when the floor was raised in the config.
02. The peer runs another iceoryx2 pin: `transport_pin: the peer's engine version is unknown (no PEER_INFO); this build is engine <version> on iceoryx2 <pin>, whose pin floor is <floor>: run both halves on the same release; the other pin's leftovers under ipc.dir go with a reboot or the marker cleanup`, or `transport_pin: the peer runs engine '<version>', below the floor; this build is engine ...` with the same tail.
03. The peer reads another config schema (1.26.0): `config_schema: the peer '<world>' (engine '<version>') reads config schema <n>; this half reads <schema> (engine '<version>') — run the same release on both halves (a mixed pair is unsupported; 'ros2 airlock migrate' rewrites the file for the newer one)`. A peer before 1.26.0 states no schema, and `reads config schema <n>` then reads `states no config schema (an engine before 1.26.0)`; with no PEER_INFO inside 3 s of its HELLO, on an engine built without a transport pin, the peer part reads `(engine unknown) states no config schema (no PEER_INFO within 3 s)`. `peer.refused` reads `config_schema`, `peer.peer_info` holds the peer's engine and stated schema, and `airlock doctor` names it `peer.config_schema`.
04. `peer_violations: a session ended for peer violations and the connector is refused for <n> s more (peer.violations names the kinds)`.
05. `control-plane peer not alive`.
06. `config epochs diverged`: the halves run different files.
07. A protected lane refused over its definition: `<n> lane(s) refused (<p> protected): <lane> [<type>] <reason> — <remedy>`.
08. `participant cannot read its world's discovery data: another ROS 2 release's layout (Humble vs Iron+) — <evidence>`.
09. `participant blind in its world`, extended with `: the world is closed to newcomers (world.closed, its shared-memory discovery port is full)` when that is the cause.
10. The cross-check participant sees what this half does not: `participant deaf to <n> configured source topic(s) a cross-check participant can see (<names>)`, or `<n> lane(s) moving bytes that a cross-check participant receives nothing on (<names>)`.
11. `graph sweep stalled`.
12. A lane that is not carrying: `lane <name> <state>` (a degraded or stalled lane), `lane <name> had <n> send(s) refused by the transport — bytes were taken and never reached the peer, which no lane state can show because the state is derived from discovery, not from delivery`, `lane <name> drops every message it is handed: its frame rewriter is unavailable, so the rewrite fails closed and nothing crosses, however healthy the lane state reads`, `lane <name> had <n> message(s) refused by this world's rmw at publish: the bytes crossed the wall and were dropped on this side`, or a protected lane retired as in the degraded list below.
13. `protected service <name> is pinned and not armed past 15 s (service.pinned_unarmed)`.

The contract is the reason text and the order of the rungs, never a rung's number: 1.26.0 inserted `config_schema` as the third rung and the ones below it moved down by one. Match a reason by its leading words (`config_schema:`, `transport_pin:`, `proto_minor:`), not by its position.

### Degraded, in rung order

1. A lane carrying stale data: `lane <name> is <n> ms behind in its own source reader — the world offers it faster than this half takes it, so it carries constantly stale data`, `lane <name> forwards data already <n> ms old — it is keeping up with its reader but what it delivers is stale, so a consumer sees the lag and no ingress counter shows it`, or `lane <name> has no working freshness instrument under <rmw> — ingress_wait cannot rise and forward_age is unusable, so staleness on this lane is unmeasured, not absent` (`<rmw>` reads `this rmw` when the implementation does not name itself).
2. An unprotected lane retired: `lane <name> retired (internal_error)` or `lane <name> retired (peer_invalid)`.
3. `service <name> is pinned and not armed past 15 s (service.pinned_unarmed)`.
4. `protected lane <name> is matching (the definitions differ; the owner is planning)`.
5. An unprotected lane refused over its definition: `<n> lane(s) refused (<p> protected): ...` as above.
6. `reload rejected: the file on disk is not what is running`.
7. The host request surface (1.26.0, a half whose world has an `api:` block): `api thread stopped (api.thread_stopped): <why>` while its thread waits for its restart, or `api socket not bound (api.socket_unbindable): <why>` while the half does not hold the socket (a live process holds the path, the path was unlinked or replaced, the directory is gone). The wall carries either way.
8. `rules rejected: <why>`.
9. `a lane is activating (awaiting the source's ACTIVATED)`.

## Lanes

Each `tx` row is a lane this half takes from its world and sends through the wall; each `rx` row is one it receives and publishes into its world; each `services` row is one service lane in either role.

```yaml
# status: tx[].state
- advertised
- flowing
- no_source
- degraded
- stalled
```

```yaml
# status: rx[].state
- advertised
- activating
- flowing
- no_source
- degraded
- stalled
- unverified
- expired
```

| State | Meaning |
| -- | -- |
| `advertised` | The lane exists and nobody asks for it; nothing is taken. |
| `activating` | A consumer asked; the source half has not confirmed yet. |
| `flowing` | Demanded and carrying. |
| `no_source` | Demanded, and the source world has no publisher for it. |
| `degraded` | Wanted and not carried as asked (a refusal, a retired lane, an unconfirmed activation past its grace). |
| `stalled` | Demanded, a source exists, and nothing moves. |
| `unverified` | The row rests on the peer's word only (the listener has not confirmed it). |
| `expired` | The entry's `expires` time has passed; the lane is closed. |

The source half's own word for an `rx` lane, from its last report:

```yaml
# status: rx[].source_state
- carrying
- degraded
- stalled
- no_source
- idle
- unknown
```

A lane row's `reason`, when present, says why it is not carrying:

```yaml
# status: tx[].reason
- demanded_silent
- shm_budget
- fd_budget
- arm_rate
- wake_capacity
- open_failed
```

```yaml
# status: rx[].reason
- peer_invalid
- demanded_silent
- shm_budget
- fd_budget
- arm_rate
- wake_capacity
- open_failed
```

- `demanded_silent`: demanded, active past the 15 s activation grace and nothing taken for 10 s. A silent source is the world's business; the health does not change.
- `peer_invalid`: retired for peer violations; `retry_in_s` counts down the backoff.
- `shm_budget`, `fd_budget`, `arm_rate`, `wake_capacity`, `open_failed`: a glob lane whose activation the live admission refused; it is retried.

The lane refusal reasons that `type_hash_refused`, a service's `refused_reason`, the glob reports' `skipped[]` and the peer's refusals carry: `hash_missing`, `peer_too_old`, `description_timeout`, `unmappable`, `rules_required`, `convert_denied`, `too_large`, `graph_hash_mismatch`, `unvalidatable`, `server_gone`, `type_changed`, `glob_retired`, `excluded`, `expired`, `shm_budget`, `fd_budget`, `arm_rate`, `wake_capacity`, `dest_taken`, `dest_server_exists`, `slots`.

### Class, cap and delivery

Every topic lane has a class (1.25.0). `control` lanes are served first and may run at real-time priority, `bulk` lanes last and under a byte cap.

```yaml
# status: tx[].class
- control
- normal
- bulk
```

```yaml
# status: rx[].class
- control
- normal
- bulk
```

```yaml
# status: tx[].class_reason
- explicit
- tf
- protected
- service
- type
- max_bytes
- observed
- default
```

```yaml
# status: rx[].class_reason
- explicit
- tf
- protected
- service
- type
- max_bytes
- observed
- default
```

The executor that takes a `tx` lane (`exc` for control and normal, `exb` the bulk executor), and whether a sample is held under the cap:

```yaml
# status: tx[].executor
- exc
- exb
```

```yaml
# status: tx[].cap_held
- 0
- 1
```

The numeric fields beside them: `cap_bytes_per_s`, `cap_fill`, `dropped_cap`, `dropped_cap_bytes`, `class_moves` on `tx` rows; `cap_bytes_per_s`, `cap_bound_bytes_per_s`, `dropped_cap`, `cap_exceeded`, `class_moves`, `delivery_age_p50_us`, `delivery_age_p99_us`, `delivery_age_max_us`, `delivery_age_samples` and `max_gap_us` on `rx` rows. A service row always reads `class: control`.

Since 1.26.0 a window that carried no sample says so by itself: `delivery_age_p50_us`, `delivery_age_p99_us` and `delivery_age_max_us` read null when `delivery_age_samples` is 0, and `max_gap_us` reads null when the window closed no interval (no sample, or the lane's first publish ever: a gap counts in the window of its second publish). Before 1.26.0 they read 0, which looked like a perfect window. `delivery_age_samples` is always an integer and is the field to branch on; a row without it comes from an engine before 1.25.0. The `classes` summary's `delivery_age_p99_us` and `max_gap_us` read null when no `rx` lane of the class carried a sample in the window (or closed an interval, for the gap); its `lanes`, `forwarded` and `dropped_cap` stay integers. `/metrics` leaves out a lane's `airlock_rx_delivery_age_us` and `airlock_rx_delivery_age_max_us` series in such a window, and its `airlock_rx_max_gap_us` series when no interval closed; `airlock_rx_forwarded_total` tells an idle lane. See [bounded bulk lanes](bounded-bulk-lanes.md).

### Real-time pumps

An `rx` row's `rt` is the destination pump's real-time state (1.25.0):

```yaml
# status: rx[].rt
- fifo:<n> # it runs at SCHED_FIFO priority n
- refused # the call was refused; process.rt says why
- ineligible:max_bytes # a control lane above airlock.rt.max_frame_bytes
- null # no airlock.rt in the file, or not a control lane
```

Beside it: `rt_batch` (the frames one wake may take), `rt_run_max_us` (the longest CPU time one wake took), `rt_frames_max` (the most frames taken between two waits), `rt_wakes` and `events_dropped` (records the pump's event ring could not hold; its violations are still counted). The half-wide record is `process.rt` (`requested`, `applied`, `eligible_threads`, `applied_threads`, `error`, `rtprio_limit`), and `airlock doctor` reports a refusal as `process.rt_refused`. See [threads and pinning](threads-and-pinning.md).

### Services

```yaml
# status: services[].role
- server_side
- caller_side
```

```yaml
# status: services[].channel
- released
- arming
- armed
- releasing
- advertised
- activating
- active
- draining
- none
```

The caller side runs `released`, `arming`, `armed`, `releasing`; the serving side `advertised`, `activating`, `active`, `draining`, `released`; an `api.local.services` row reads `none` (1.26.0: it has no channel). A caller-side row carries `last_arm_error` and `last_arm_error_age_s` while the last arm failed; the next successful arm clears both (1.25.0).

```yaml
# status: services[].client
- none
- cached
- live
```

```yaml
# status: services[].origin
- wall
- local
```

Since 1.26.0 every serving row says where its calls come from: `wall` for a service the wall carries, with `local_calls` and `local_synthesized` counting the host's calls of that name through the api socket beside the wire's own `calls` and `errors`; `local` for an `api.local.services` name, a row with no channel and no carrier whose `calls`, `replies`, `timeouts` and `errors` are the host's.

```yaml
# status: services[].carrier.kind
- exact
- glob
```

### Where a lane comes from

```yaml
# status: tx[].provenance
- base
- concern
- overlay
```

```yaml
# status: rx[].provenance
- base
- concern
- overlay
```

```yaml
# status: services[].provenance
- base
- concern
- overlay
```

### QoS

```yaml
# status: tx[].qos_in.reliability
- reliable
- best_effort
```

```yaml
# status: tx[].qos_in.durability
- volatile
- transient_local
```

```yaml
# status: rx[].qos_out.reliability
- reliable
- best_effort
```

```yaml
# status: rx[].qos_out.durability
- volatile
- transient_local
```

## Threads

Every engine thread is listed in `threads` with its role (1.25.0). The main thread keeps the binary's name `airlock_half` and reads role `main`; a library's thread reads `foreign`; an engine thread whose name breaks the scheme reads `off_scheme`, which doctor reports.

```yaml
# status: threads[].role
- main
- sig
- initwd
- ctrlio
- exc
- exb
- heal
- deadman
- cap
- rx
- sd
- ss
- rebuild
- metrics
- blackbox
- audit
- history
- api
- flap
- foreign
- off_scheme
```

```yaml
# status: threads[].class
- control
- normal
- bulk
- null
```

## The host request surface

A half whose world's config carries an `api:` block serves a local socket for host tooling (1.26.0) and reports it under `api`: the configured `socket`, whether the half holds it (`bound`, and `bind_error` when it does not), the connections, the requests by operation, every refusal by its code, the graph replies served and cut at the size cap, the sample and call outcomes, the `api.local` names with their purpose, the surface's own service clients, `instance` (the half's start, the reply header's `instance`), the last uid refused at connect with its age, and its thread. `capabilities` holds `api_socket` while the block is configured. A consumer takes the socket's path from the config, never from this file.

```yaml
# status: api.thread
- running
- stopped
```

`stopped` is the thread waiting for its restart (after 1 s, doubling to 60 s; `restarts` counts them); `airlock doctor` reports it as `api.thread_stopped`, a socket the half does not hold as `api.socket_unbindable`, a uid refused in the last 5 minutes as `api.refused_uid`.

Every admitted uid has its own token buckets (1.27.0): a request is charged to the uid of its connection, so a tool running as one uid that spends its budget is refused `rate` while another uid (root's watchdog, say) is still served. `api.uids[]` lists every uid holding buckets now, in uid order: `connections` it holds, `tokens` (what its `graph`, `samples` and `calls` buckets hold now), `refused_rate` (the requests each bucket refused since it was created) and `idle_s` (the seconds since its last connection closed, null while it holds one). A uid that holds no connection for `api.uid_idle_drop_s` seconds (60, `--constants`) is dropped and starts full again at its next request. `api.refused.rate` still counts every rate refusal of the half; `/metrics` carries `airlock_api_rate_refused_total{uid,op}`.

The `sample` operation (1.26.0) counts every outcome under `api.samples`: `served` (and `truncated`, those of them cut at the reply cap), then one counter per error code (`no_publisher`, `timeout`, `invalid`, `not_carried`, `not_allowed`, `type_unsupported`, `size`, `blind`, `internal`), and `retained` (1.26.1, below). A refusal before the operation runs (`scope`, `busy`, `rate`, `invalid_request`) counts under `api.refused`. Each `tx` row carries `samples`, the host's samples of that lane served, `sample_errors`, the ones that failed by code (a code that never happened is absent), and `last_sample_age_s`, the seconds since the last one was served (null when never; `airlock api` shows a lane read within the last status period as `host_read`); a sample never demands the lane or moves its `forwarded`. Since 1.27.0 a lane delivered INTO the half's world is sampled too, by its destination name in that world (the namespace and the rename applied), through a reader of the surface on that name, so it sees what the world's nodes see, the half's own publication included; its `rx` row carries the same `samples`, `sample_errors` and `last_sample_age_s`, and the surface's readers on the name never count as its demand. `airlock doctor` reports `api.type_unsupported` when a sample asked for a type this image cannot describe. `/metrics` carries `airlock_api_samples_total{topic,outcome}` per carried name.

The `call` operation (1.26.0) counts every outcome under `api.calls`: `served` (the server answered), `synthesized` (the surface made the reply up and says why), `duplicate` (a retry answered from the ledger, executing nothing), then one counter per error code (`not_served`, `not_allowed`, `invalid_request`, `size`, `blind`, `type_unsupported`, `internal`). A refusal before the operation runs (`scope`, `inflight`, `rate`) counts under `api.refused`. `api.last_synthesized` names the newest synthesized reply (its service, reason, age and the count so far); `airlock doctor` reports one in the last 5 minutes as `api.call_synthesized`. The reasons, fixed:

```yaml
# status: api.last_synthesized.reason
- timeout
- no_client
- busy
- teardown
- response_over_cap
- send_failed
```

`timeout` was sent and may have executed; `no_client` (no server matched the surface's client), `busy` (an `api.local.services` entry over its own `max_inflight`) and `send_failed` were never sent; `teardown` (the half stopping, or the name no longer callable after a reload) may or may not have been sent, and the reply's `forwarded` says which; `response_over_cap` executed. `/metrics` carries `airlock_api_calls_total{service,origin,outcome}` and `airlock_api_call_ms{service,p}` (p50 and p99 of the last 64 replies).

```yaml
# status: api.local[].kind
- service
- topic
```

```yaml
# status: api.clients[].origin
- wall
- local
```

```yaml
# status: api.clients[].state
- live
- cached
```

A lane listed in `api.retain` (1.26.1) has a standing reader the half keeps for its life, and a sample of it answers from the newest message that reader took (the reply carries `retained: true`) unless the request asks `fresh: true`. `api.samples.retained` counts the answers served that way (each one is in `served` too), and `api.retained[]` lists every retained lane in the order the config names them: `matched` and `publishers` (the reader's matched publishers), `last_receipt_age_s` (the newest message's receipt, null before the first), `samples` (the answers served from it), `readers_created` (one per creation: the first, a publisher of another durability or reliability joining, a participant rebuild), and its state:

```yaml
# status: api.retained[].state
- waiting
- none
- held
- blind
```

`waiting` has no reader yet: no publisher of the lane is discovered, or its type is not known yet. `none` has a reader and no message yet; a sample then waits for the first up to its `wait_s` and answers `no_publisher` or `timeout` when none comes, never an older value. `held` holds the newest message. `blind` is a reader a participant rebuild tore down that is not back yet; a sample answers `blind`.

## The rest of the file

The peer refusal that stopped a session:

```yaml
# status: peer.refused
- proto_minor
- transport_pin
- config_schema
- null
```

The inventory's view of each topic in the world:

```yaml
# status: inventory.topics[].bridged
- none
- advertised
- activating
- flowing
- no_source
- degraded
- stalled
- unverified
- expired
```

Entries with an `expires` time:

```yaml
# status: expiring[].kind
- topic
- service
- action
```

```yaml
# status: expiring[].state
- live
- expires_clock_unsynced
- expires_clock_suspect
- expired
```

The ledger `learn` reads, and the directories the half writes:

```yaml
# status: ledger.trust
- unverified
- live
- frozen
```

```yaml
# status: dirs[].role
- ipc
- status
- forensics
```

`status` is where this half writes `status_<world>.json`: its world's `status_dir` when the config sets one (1.26.0), else `ipc.dir` (then its warnings are listed once, under `ipc`). `forensics` is the world's `forensics_dir` when set, else `forensics.dir`: the audit rings, the black box, the violation ring and the api audit file land there, and the history ring too when the world sets `forensics_dir` (else it stays in `ipc.dir`). A reader takes a half's status path from the config's `status_dir`, never from the other half's status.
