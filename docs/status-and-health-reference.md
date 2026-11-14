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
- empty
- n/a
```

| Value | Meaning |
| -- | -- |
| `live` | At least one configured source entry has been discovered in this half's world. |
| `watching` | Nothing configured has matched yet, and the half sees other nodes in its world: the lazy, healthy reading of an empty `tx`. |
| `blind` | Nothing matched and nobody is visible past the grace window, or a participant rebuild has not yet been cleared by evidence, and no cross-check has confirmed the world empty. The half rebuilds its participant on a ladder unless a cross-check rules that out; health reads `dysfunctional`, except in the two `unchecked` windows under Health below. |
| `empty` | (1.29.1) Nothing matched and nobody is stably visible past the grace window, and this half's own cross-check, a second participant that confirmed it joined this world, saw no other node in it. An empty world, not a fault: health reads `healthy` with a note, `blind_s` reads 0, `rebuild_suppressed` stays and the half never rebuilds. |
| `n/a` | A destination-only half: it has no source entries, so it has nothing to discover. |

**The empty world** (1.29.1). A half alone in its world reads `healthy` with `watching` through its blind grace, then `unchecked` until its first cross-check answers, then `healthy` with `empty` once that answer confirms the world empty. The grant needs a probe with verdict `ok` that saw no other node at all (`discovery.cross_check.foreign_nodes` 0) and that confirmed it joined this half's world: it saw one of this half's own topics, or this half's own node. It also needs `discovery.foreign_release_layout` false, `shm_discovery.state` not `closed` and `cross_check_s` above 0. Once `empty` stands, a later probe that saw only nodes this half saw itself while the probe ran keeps it (`foreign_nodes` can then be above 0), so a passer-by such as a `ros2 topic list` from another computer does not flip the grade. A node that becomes visible turns the state `watching` or `live` within a sweep and a write, the verdict staying `healthy`, and the next sweep that sees nobody reads `empty` again; a node that stays visible for 5 s ends the blind run and the grant with it. A probe that sees a node this half did not see (after a 5 s hold, in case this half is one sweep behind), finds the half deaf, fails or reads `newcomer_blind`, or a premise that stops holding, turns the half `blind` and `dysfunctional` at the next sweep. Because the cross-check backs off to `cross_check_s` in an empty world (`cross_check_cadence` `no_unseen`), a node this half cannot see is reported up to one `cross_check_s` after it appears: about five minutes by default and up to an hour on `rmw_zenoh_cpp`. For up to 5 s after a probe that saw a node this half did not, a document can carry `empty` beside a non-empty `cross_check.nodes_unseen` and the held `rebuild_suppressed` text, with no `health.reason`. `blind_s` reads 0 for as long as the grant stands, also while a visitor turns the state `watching` or `live`. A grant rests on the next probe answering: a probe still unanswered 20 s past its due moment turns the half `blind` and `dysfunctional` until it answers. A world in which this half sees publishers with no node name behind them (participants of another ROS 2 release, or a bare DDS one) is never graded `empty`, since a probe sees no node there either. A world on another domain, RMW or profile looks exactly like an empty one. Any half can read `empty`, a robot half too: a robot half whose driver world has died (every node in it stopped) reads `healthy` with `empty` once its cross-check confirms nobody is left, where 1.29.0 read `dysfunctional`. To be alerted when a driver stack dies, alert on the robot half's `discovery.state` reading `empty` (or on its lanes), not on its verdict. The api reply header's `discovery.state` reads `empty` too; a sample, a call or a goal in an empty world answers by what the world holds (`no_publisher`, `timeout`, `not_served`), never `blind`.

`discovery.last_poll_s` is the other half of the question: a stalled graph loop leaves it climbing while every other field still looks fresh.

`discovery.cross_check_s` is how often the half runs that second participant, in seconds: `recovery.cross_check_s` when the config sets it, otherwise the default of the half's RMW, 3600 on `rmw_zenoh_cpp` and 300 on every other, and `AIRLOCK_CROSS_CHECK_S` over both (0 means disabled). On Zenoh the default is an hour because each cross-check opens a Zenoh session, and zenoh-c 1.8.0 keeps about 25 KB of every session it closes, so at 300 s a half would grow by about 7 MB a day; at 3600 s that is about 0.6 MB. Set `cross_check_s` yourself to choose another cadence.

`discovery.cross_check_cadence` (1.29.1) is the schedule the cross-check runs on now. `discovery.cross_check_next_s` is the whole seconds until the next probe is requested, not answered: it reads 0 while a probe is requested, running, or answered and not yet applied (an answer is applied at the next 1 s sweep), and after a reload, which asks for a probe at once; it is `null` while the cross-check is disabled. It also reads 0 on a half that is due a probe and has nothing to ask about: its blind clock is not running, every exact rule has matched and no lane is carrying (the resting state of an idle lazy half). No probe is requested then (`cross_check.age_s` keeps growing and `cross_check_cadence` reads `first` or `routine`); the next one is requested at the first sweep that has something to ask about. Both are in the file from the half's first write.

```yaml
# status: discovery.cross_check_cadence
- first
- fast
- routine
- no_unseen
```

- `first`: no probe has answered yet. The first probe of the process comes at a fifth of `cross_check_s`, at least 20 s after start; while the half's blind clock runs (nobody visible in its world) it comes no later than `blind_grace_s` after start (20 s at least), so its answer lands about when the grace ends (60 s instead of 720 s on `rmw_zenoh_cpp`).
- `fast`: every `cross_check_fast_s`, after a probe found a fault and while the half's blind clock runs (nobody else visible in its world) until a probe has found the world empty.
- `routine`: every `cross_check_s`, the rest of the time.
- `no_unseen`: every `cross_check_s`, in a world a probe has found empty. A probe that joined the half's world and saw no other node in it starts it. Every later probe that saw no node this half did not see itself keeps it, so a passer-by both saw (a `ros2 topic list` from another computer) does not end it, and `discovery.cross_check.foreign_nodes` can then be above 0. A probe that sees a node this half did not see, finds the half deaf, fails or reads `newcomer_blind` ends it, and so does a node that stays visible for 5 s; the half is then back on `fast`. While it stands the half does not re-create its participant. An empty world costs 12 probes an hour at the default 300 s instead of about 52, and one an hour on `rmw_zenoh_cpp`. The price: a node this half cannot see is found up to one `cross_check_s` later, 300 s by default and up to an hour on Zenoh. For up to 5 s after a probe that saw a node this half did not, a document can carry `no_unseen` beside a non-empty `discovery.cross_check.nodes_unseen` and the `rebuild_suppressed` text that says the finding is held; the next sweep that still finds the node unseen ends it.

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

`shm_foreign_owner` appears when a half that does not run as root, in a Fast DDS world, finds files in `/dev/shm` that another uid owns and this half may not write. `segments` counts the shared-memory segments (`fastrtps_*`, `fastdds_*`): with them the half joins only part of a world whose participants run as `uid`. `port_mutexes` (1.29.0) counts the ports' named semaphores (`sem.fastrtps_port<N>_mutex`, or `sem.fastdds_*` on Lyrical) and `port_mutex_names` lists the first eight: the half cannot open those ports, and when the discovery port is among them it never joins its world. A port mutex is either a live world's of that uid or the leftover of a participant of that uid killed with SIGKILL, and the half cannot tell which, so its log line and `airlock doctor` (`world.shm_foreign_owner`) name both remedies: the capabilities or one uid for a live world, or, once every DDS process of that uid is gone (root included), `rm -f /dev/shm/sem.fastrtps_* /dev/shm/sem.fastdds_*`. The half never deletes these files itself. The 0-byte `*_port<N>_mutex_holder` files are not counted: the patched Fast DDS in the image creates one per port, never removes it and opens it read-only, so a holder this half may read blocks nothing. One it may not read (another uid's, created 0600 under a strict umask) shuts it out of that port and counts under `segments`. The half judges `/dev/shm` at start, on every rebuild and at most 60 s apart while it runs (1.29.1), so a root driver that starts or restarts after the half is named within about a minute, and a finding that clears leaves the status within the same bound. Its warning is logged when the finding appears or its `uid` changes, never on a count change alone, and an INFO line (`world.shm_foreign_owner: cleared: ...`) names the clearing. A half running as root, or on an RMW other than Fast DDS, never scans.

`rmw_profile_unsupported` appears when a half's process runs `rmw_cyclonedds_cpp` and the Cyclone profile its `CYCLONEDDS_URI` names sets `General/RedundantNetworking` true, which airlock does not support. Each row is `{setting, file, interfaces}`: the setting's path, the file that set it (`<inline>` for inline XML), and how many network interfaces the whole profile lists (its `General/Interfaces/NetworkInterface` entries, or the deprecated `General/NetworkInterfaceAddress` list; 0 when it lists none and Cyclone picks one). With more than one interface, Cyclone DDS 0.10.5 aborts a process using that profile on an assertion (`ddsi_wraddrset.c:488`) when another process on the same host reads a topic it writes, for example a `ros2 topic echo` in the fleet world's environment; the half's own cross-check probe does not trigger it (measured on 2026-11-11). With one, the setting does nothing useful. The half reads the profile once at start and logs one ERROR line naming the file; it works nothing around. Doctor names the field as `world.rmw_profile_unsupported` (a problem) even when the half is crash-looping and doctor runs in another shell, and `airlock check` warns `rmw.profile_unsupported` from its own environment. Remove the setting; `profiles/fleet_world/cyclonedds_two_interfaces.xml` shows how to reach two networks without it.

## Health

`health.verdict` is the engine's own roll-up of every rung, so a consumer reads one value instead of re-deriving the rules.

```yaml
# status: health.verdict
- healthy
- degraded
- dysfunctional
- unchecked
```

- `healthy`: nothing below applies.
- `degraded`: the wall works but is not converged or not as asked.
- `dysfunctional`: the wall is not carrying what it should.
- `unchecked` (1.29.1): a blind half (`discovery.state` `blind`) whose blindness is not yet a fault, and no other rung applies. Read it as pending: neither green nor red. Two windows:
  - its first cross-check of this blind run is pending, from the grace's end until the answer: `world '<world>' looks empty: its first cross-check is pending (within <n> s)`, `<n>` the seconds left of the window. It is bounded: past the probe's due moment plus 20 s with no answer, and after a participant rebuild in the same blind run, the half reads `dysfunctional` as below;
  - before the first `empty` grant, the standing cross-check saw only passing nodes, each of which this half saw too: `world '<world>' looks empty apart from passing nodes: waiting for a cross-check that sees nobody (no fault found)`. It has no bound of its own, since every node such a probe saw, this half saw too (a probe still unanswered 20 s past its due moment, or a participant rebuild in the same blind run, ends it in `dysfunctional`); the next probe that sees nobody grants `empty`, and one that sees a node this half did not, finds the half deaf, fails or reads `newcomer_blind` turns it `dysfunctional`.

A `healthy` verdict carries `health.reason` in one case (1.29.1): while `discovery.state` is `empty`, `world '<world>' is empty: a cross-check participant saw no other node in it (no fault)` when the standing probe saw nobody, or `world '<world>' is empty apart from passing nodes this half also saw (no fault)` when the grant is kept by a probe that saw passers-by, and none for up to 5 s after a probe that saw a node this half has not glimpsed yet (the hold above). None of the four 1.29.1 reasons contains the word "blind".

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
09. `participant blind in its world`, extended with `: the world is closed to newcomers (world.closed, its shared-memory discovery port is full)` when that is the cause. From 1.29.1 an `empty` half skips this rung, and a blind half in an `unchecked` window above skips it too while no rung below applies.
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

An `rx` row's `demanded` says whether a consumer in this world asks for the lane: a subscriber the last demand poll found (one poll a second), or the entry is `activation: always`. A lazy lane deactivates only after two polls in a row find no subscriber (engine 1.28.0, `lazy.deactivate_after_polls` in the [constants](constants-contract.md)), so for the one poll between them the row reads `demanded: false` while its `state` still says the lane flows. For that poll a listener does not grade the row `unverified` and it carries no `demanded_silent` reason, since both apply only to a demanded lane. A lane deactivates 1 to 2 s after its last subscriber leaves, and up to 2 s later while a host sample reader that has just been released is still matched. From 1.29.0 a host sample is demand on a lazy lane delivered into the half's world for its wait, and a retained lane's standing reader is standing demand, so `demanded` reads true for them too. A sample can therefore hold an unconfirmed lane past its 15 s activation grace: two samples back to back of a lane whose source half does not confirm the ACTIVATE leave the row `degraded` (an unconfirmed activation) and the half's health `dysfunctional`, as for any demanded lane the source does not carry; that is the honest verdict for a lane someone asked for.

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

Beside it: `rt_batch` (the frames one wake may take), `rt_run_max_us` (the longest CPU time one wake took), `rt_frames_max` (the most frames taken between two waits), `rt_wakes` and `events_dropped` (records the pump's event ring could not hold; its violations are still counted). The half-wide record is `process.rt` (`requested`, `applied`, `reason`, `eligible_threads`, `applied_threads`, `error`, `rtprio_limit`). `applied` is null when no eligible pump runs. `reason` (1.27.1) is `not_requested`, `no_eligible_pump` (asked for, with no running control pump to apply it to: no control lane into this world, or every one idle), `applied`, `refused:<errno>` (`refused:EPERM`) or `partial`. `airlock doctor` reports a refusal as `process.rt_refused` and the `no_eligible_pump` case as the information `process.rt_no_eligible_pump`. See [threads and pinning](threads-and-pinning.md).

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

The `sample` operation (1.26.0) counts every outcome under `api.samples`: `served` (and `truncated`, those of them cut at the reply cap), then one counter per error code (`no_publisher`, `timeout`, `invalid`, `not_carried`, `not_allowed`, `type_unsupported`, `size`, `blind`, `internal`), and `retained` (1.26.1, below). A refusal before the operation runs (`scope`, `busy`, `rate`, `invalid_request`) counts under `api.refused`. Each `tx` row carries `samples`, the host's samples of that lane served, `sample_errors`, the ones that failed by code (a code that never happened is absent), and `last_sample_age_s`, the seconds since the last one was served (null when never; `airlock api` shows a lane read within the last status period as `host_read`); a sample of a lane this world carries out never demands it or moves its `forwarded`. Since 1.27.0 a lane delivered INTO the half's world is sampled too, by its destination name in that world (the namespace and the rename applied), through a reader of the surface on that name, so it sees what the world's nodes see, the half's own publication included; its `rx` row carries the same `samples`, `sample_errors` and `last_sample_age_s`. From 1.29.0 a fresh sample of such a lane, when it is lazy, is demand for its wait (it wakes an idle lane, which goes idle two polls after the sample's reader is gone), and a retained lane's standing reader is standing demand; the host's action status reader is no demand. The reply's `demand` (`sample` when the sample's reader was the lane's only demand) and `activated_s` (the seconds from the request to the lane's activation, null when it was already carrying) say whether the sample woke the lane, and a timeout of a lane whose activation never confirmed reads `the lane is lazy and idle: this sample asked for it and the source did not confirm within <wait> s` (code `timeout`). `airlock doctor` reports `api.type_unsupported` when a sample asked for a type this image cannot describe. `/metrics` carries `airlock_api_samples_total{topic,outcome}` per carried name.

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
- action
```

```yaml
# status: api.clients[].state
- live
- cached
```

A lane listed in `api.retain` (1.26.1) has a standing reader the half keeps for its life, and a sample of it answers from the newest message that reader took (the reply carries `retained: true`) unless the request asks `fresh: true`. `api.samples.retained` counts the answers served that way (each one is in `served` too), and `api.retained[]` lists every retained lane in the order the config names them: `matched` and `publishers` (the reader's matched publishers), `last_receipt_age_s` (the newest message's receipt, null before the first), `samples` (the answers served from it), `readers_created` (one per creation: the first, a publisher of another durability or reliability joining, a participant rebuild), `takes` (engine 1.29.1: the messages its standing readers took, cumulative; `last_receipt_age_s` is whole seconds, so only `takes` tells a row kept current from a stale one), and its state:

```yaml
# status: api.retained[].state
- waiting
- none
- held
- blind
```

`waiting` has no reader yet: no publisher of the lane is discovered, or its type is not known yet. `none` has a reader and no message yet; a sample then waits for the first up to its `wait_s` and answers `no_publisher` or `timeout` when none comes, never an older value. `held` holds the newest message. `blind` is a reader a participant rebuild tore down that is not back yet; a sample answers `blind`. From 1.28.0 `api.retain` may also name an `api.local.topics` name, whose row reads the same way.

The api thread is event-driven from engine 1.29.1: it wakes for a connection, an operation's descriptor, a retained lane's message, a reload, a participant rebuild, a graph change that concerns a retained name, or the earliest deadline it holds, and at least once a second. `api.turns` counts its wakes and `api.retain_graph_queries` the graph questions it asked for the retained lanes' publishers (one per retained name with a known type, asked only on such a graph change, a reload, a rebuild or the 5 s resync), both cumulative. On an idle half on Jazzy or Lyrical `turns` grows by about 60 a minute and `retain_graph_queries` by about 12 a minute per retained name (engine 1.29.0 turned about 600 times and asked about 600 times per name a minute). While a goal or a call is in flight the thread turns every 100 ms. A retained lane's messages are drained at most every 100 ms with no watch open, so a busy retained lane adds about 600 turns a minute; on Humble, whose message wake is lossy, and for any reader whose rmw offers no wake, the retained lanes are drained on a 100 ms tick that asks the graph nothing, so `turns` grows by about 600 a minute there whenever a retained lane has a reader.

A watch (1.28.0) is a connection that streams the updates of retained lanes. Once its first line is read it no longer counts under `connections` (nor `peak_connections`, nor `airlock_api_connections`): it holds one of `api.max_watchers` instead, so `connections` counts request connections only. `api.requests.watch` counts the watch requests, `api.refused.watchers` the ones refused because every watch slot was taken. `api.watchers[]` lists every open watch: the peer's `uid`, the `names` it watches, the `updates`, `coalesced` takes and `keepalives` it was sent, and `since_s`, how long it has been open. `api.watch` holds the counters:

- `open`, the watches open now, and `opened`, every watch opened since the half started;
- `refused`, the watches refused at open by code: `not_watchable` (a name not in `api.retain`), `watchers` (every slot taken), `scope` (an `api.local` name for a uid with the wall's scope only), `invalid_request`, `rate` and `internal`;
- `updates`, the update lines sent, and `coalesced`, the takes they did not carry; `keepalives`;
- `stalled`, the watches closed because their client stopped reading (bytes left unread for 3 x `api.watch.keepalive_s`, the kernel's count of them), each one written to the audit file `api_<world>.jsonl` with `kind: watch`;
- `closed`, how every watch ended: `eof` (the client closed it), `stalled`, `size` (the client sent more than `api.max_request_bytes` without a newline, or more than 64 lines, or `api.max_request_bytes` of them, on the watch), `teardown` (the half stopped, or its `api:` block went) and `reload` (a reload took a watched name out of `api.retain`);
- `discarded_lines`, the request lines clients sent on an open watch, which the half reads and ignores;
- `last_stalled`, the newest stall (its `uid`, `names`, `age_s` and `count`), null when there was none; `airlock doctor` reports one in the last 5 minutes as `api.watch_stalled`.

`/metrics` carries `airlock_api_watchers` (the open watches) and `airlock_api_watch_updates_total{name}`, and `airlock_api_requests_total` an `op="watch"` series. The history ring records an `api_watch` event when a watch opens and when it closes, coalesced per uid per minute.

A host goal (1.28.0) is a connection whose first line is an `action_goal`: like a watch it leaves `connections` once that line is read and holds one of `api.max_goals` instead (`api.refused.goals` counts the goals refused because every slot was taken; `api.refused.goal_busy` the goals refused because their action already had one). `api.requests.action_goal` and `api.requests.action_cancel` count the requests. `api.actions[]` lists every action of `api.actions`, in its order: its `type`, `origin` (`wall` or `local`), `state` (`idle`; `active`, a host goal holds the action's one slot; `canceling`, the half asked its cancel; `recovering`, a goal a killed run left open is being cancelled at this start and no new goal is accepted until that is done), the active goal's `goal_id` and `owner_uid` (null when idle), and `goals`, every goal by outcome: `succeeded`, `aborted`, `canceled`, `timeout` (its `max_duration_s` ran out and the half cancelled it), `lost` (no server, or no status of it, for `lost_after_s`; or the server did not answer the goal or its cancel), `rejected`, `canceled_on_disconnect` (its client went and the cancel that caused ended it), `canceled_at_start`, `cancel_failed` (a cancel answered `rejected` or `unknown_goal`, or not answered within `timeout_s`; the goal stays tracked), and `send_failed` (a stopping half cancels its goals and leaves their audit trail open for the next start, so no outcome is counted for them). `api.goals` holds the counters:

- `open`, the goal streams open now, and `opened`, every goal the half went on to send since it started;
- `refused`, the goals refused at open by code: `goal_busy`, `not_allowed` (an unlisted action, a uid the entry does not admit, a fixed goal or a goal constraint), `invalid_request`, `rate`, `blind`, `type_unsupported` and `internal`;
- `duplicates`, the retried request ids answered with their first goal;
- `cancels`, the cancel requests the half sent (on a disconnect, a timeout, a host cancel, a stop, a rebuild, a start);
- `last_cancel_failed`, the newest cancel that failed (its `action`, `goal_id`, `reason` `rejected`, `unknown_goal`, `unanswered` or `no_server`, `age_s` and `count`), null when none did; `airlock doctor` reports one in the last 5 minutes as `api.action_cancel_failed`;
- `last_lost`, the newest goal lost (its `action`, `goal_id`, `cause`, `age_s` and `count`); `airlock doctor` reports one in the last 5 minutes as `api.action_lost`.

```yaml
# status: api.actions[].origin
- wall
- local
```

```yaml
# status: api.actions[].state
- idle
- active
- canceling
- recovering
```

```yaml
# status: api.goals.last_cancel_failed.reason
- rejected
- unknown_goal
- unanswered
- no_server
```

`api.clients[]` lists the half's client of each action's `_action/cancel_goal` with `origin: action`. Every goal writes to the audit file `api_<world>.jsonl` with `kind: action` (the call lines carry `kind: call` since 1.28.0): `goal_sent` before the goal leaves, `accepted`, `cancel_sent` with its reason, `cancel_response` with the server's return code, and `terminal` with the outcome; a half reads the file at its start and cancels every goal it shows sent and not ended. `/metrics` carries `airlock_api_action_active{action}` (1 while a host goal holds the action) and `airlock_api_action_goals_total{action,outcome}`, and `airlock_api_requests_total` the `op="action_goal"` and `op="action_cancel"` series. The history ring records an `api_action` event when a goal opens, is refused or ends, coalesced per action per minute.

## Adapters and readiness (engine 1.29.0)

A half with an `api:` block also writes two top-level lists. `extensions[]` has one row for every adapter its `api.adapters` declares and for every undeclared name a request stated in its `adapter` field (at most 16 of those). A row carries the name, the uid, the contract the adapter's newest request stated (`contract`, beside `contract_served`), the age of its last sign of life (`last_seen_s`: 0 while one of its connections is open, null if it never came), its requests and open connections, and its calls and goals by outcome. A sign of life is an open connection or a request, so an adapter that holds a watch open and asks rarely stays alive. An adapter that never came, or one that stopped, is a doctor warning and never changes the half's health.

```yaml
# status: extensions[].kind
- adapter
```

```yaml
# status: extensions[].state
- connected          # a connection bound to the name is open now
- idle               # it came this run and has no connection open now
- absent             # it never came this run
- contract_mismatch  # its newest request stated a host contract this half does not serve (refused `version`)
- uid_mismatch       # its newest request came from another uid than the declared one (refused `not_allowed`, audited)
```

Doctor reads the rows: `extension.absent` (WARN) names a declared adapter that never came, or whose name came from the wrong uid, with its purpose; `extension.stale` (WARN) one that has been `idle` for its `stale_s` (default 60); `extension.contract` (WARN) one whose newest request stated another contract, with both numbers; `api.adapter_unexpected` (INFO) a name nobody declared. The audit file `api_<world>.jsonl` names the adapter of every call, goal and stalled watch in `peer.adapter`, and a refused request from the wrong uid as a `kind: adapter` line.

`readiness[]` has one row per item of the readiness profile `api.readiness` names, empty without one. The half evaluates the profile at every status write from what it already holds: its own graph, its lane rows and the rates it measures on the lanes it carries or retains. It creates no reader and no demand, and the result gates nothing.

```yaml
# status: readiness[].kind
- topic
- service
- action
```

```yaml
# status: readiness[].state
- ok
- missing     # not in the graph, no publisher or server, or a carried lane reads no_source
- untyped     # present under another type
- slow        # a min_hz item measured below its rate
- idle        # a min_hz item on a carried lazy lane nobody demands: no rate to measure
- unmeasured  # a min_hz item on a name the half neither carries nor retains, no rate window yet, or over 10 Hz on a name only retained
```

Doctor warns `readiness.<profile>.<item>` for every row that is not `ok`, with the item's purpose and the `detail`. A profile that does not parse never takes the half down: the half runs without one (at start) or keeps the one it was running (on a reload), `readiness_error` says why, and doctor warns `readiness.profile_invalid`.

## The monitor block

`monitor` (1.29.0, `monitor_schema: 1`) gathers what a resource monitor plots into one object, so a monitor reads one block instead of walking the rows. It is a roll-up of the same status write: every figure is taken from a value the write serialises elsewhere in the file, the block is written last, and nothing of it runs on a lane's path. `/metrics` carries none of it (the rates are derivable there from the counters), and the history ring is unchanged. The block is additive under status schema 3 and not required: a consumer that reads it checks `monitor_schema` and treats an absent block as a pre-1.29.0 half.

Clocks. `monitor.ts_ns` is the top-level `ts_ns`, the same CLOCK_MONOTONIC reading; `monitor.wall_ns` is CLOCK_REALTIME read beside it, for a series kept in wall time. A rate below is the growth of a cumulative counter between this write and the half's previous write, divided by the interval between their `ts_ns`, so it is a rate over about 5 s (a write after a participant rebuild comes sooner, and its interval is the measured one). Every rate is null on a half's first write, since there is nothing to subtract.

- `world`: the half's world, as at the top level.
- `health`: the top-level `health` object copied, `verdict` and `reason` when there is one. The block's `health.verdict` takes the same values:

```yaml
# status: monitor.health.verdict
- healthy
- degraded
- dysfunctional
- unchecked
```

- `cpu_s`, `rss_kib`, `max_rss_kib`: the top-level fields, the same readings: the process's user plus system CPU seconds since start (getrusage), its resident memory now (`/proc/self/statm`) and its peak (`ru_maxrss`), in KiB.
- `threads_cpu_s`: CPU seconds per thread role, cumulative: each thread's user plus system time from `/proc/self/task/<tid>/stat` at this write, summed over the threads of `threads` by their role, so every key is a role of that list (the roles are under Threads, above). A thread that has exited is no longer counted, so a role's figure can fall when a pump ends; a monitor that wants a role's rate takes the difference of two writes and ignores a negative one.
- `shm`: the top-level `shm` block's `effective_budget`, `standing_bytes`, `live_bytes` and `peak_bytes` under the names `budget`, `standing`, `live` and `peak`, in bytes; `peak` is a cumulative maximum since start.
- `lanes`: `total` is the top-level `lanes.count` (topic lanes, service lanes and action parts); `active` counts the `tx` and `rx` rows whose state is `flowing` or `activating`; `unmet_demand` counts the `tx` and `rx` rows carrying reason `demanded_silent` (a lane someone demands whose source has sent nothing for 10 s).
- `rate_per_s`: per second between the two writes, summed over the `tx` and `rx` rows: `forwarded` frames, `bytes` (the rows' `bytes_forwarded`), and `dropped`, one rate per drop counter of the rows under the row's own name (`dropped_rate`, `dropped_size`, `dropped_loan`, `dropped_self`, `dropped_take`, `dropped_convert`, `send_failed`, `ingress_lost`, `dropped_cap`, `publish_failed`, `evicted`, `dropped_malformed`, `dropped_rewrite`, `dropped_invalid`, `dropped_nonfinite`; a name both a `tx` and an `rx` row carry is summed). `trailing_trimmed` is not there: it counts the trimmed bytes of a frame that was delivered, not a drop. `dropped_rate` is the count the min-period limiter dropped, so its rate is drops per second. A row first seen in this write, or whose counter went down (an entry recreated), counts from its next write, so a glob lane that appears or a reload cannot bend the sum. A frame that crosses the wall counts once on each half (in the source half's `tx` row and in the destination half's `rx` row), so a monitor that adds both halves' `forwarded` rates counts every frame twice.
- `worst`: the largest figure over the `rx` rows, each as `{value, lane}` with the row's `dest` (on a tie, one of them: the first offered), both null when no row has the figure:
  - `delivery_age_p99_us`: the rows' `delivery_age_p99_us`, the producer's take-complete stamp to the return of the destination publish, over the last closed 5 s window.
  - `bridge_latency_p99_us`: the rows' `peer_report.latency_p99_us`, the time from the frame's stamp on the source half to its handoff before the publish on this one, over the last window. Both halves read one host's CLOCK_MONOTONIC, so the difference needs no clock between worlds. It is not delivery age (it stops before the publish) and not end to end (it starts at the source half's take, not at the publisher).
  - `max_gap_us`: the rows' `max_gap_us`, the largest interval between two consecutive publishes in the last window.
  - `rt_run_max_us`: the rows' `rt_run_max_us`, the longest run of a pump per wake, cumulative per pump and reset when the lane reactivates (a new pump); a row whose pump has not woken has none.
- `api`: null when the half's world configures no `api:` block. Otherwise `requests_per_s` is the growth of `api.requests` per operation (`graph`, `sample`, `call`, `watch`, `action_goal`, `action_cancel`) per second between the two writes, and `watchers` the number of `api.watchers` rows, the watch streams open at this write.

## The rest of the file

The peer refusal that stopped a session:

```yaml
# status: peer.refused
- proto_minor
- transport_pin
- config_schema
- null
```

The inventory lists every topic and service the half's node sees in its world, its own included. Since 1.29.0 the half and its cross-check probe start without parameter services and without a `/parameter_events` publisher, so `inventory.services` carries no `/airlock_<world>/*parameters*` rows any more; the fields are unchanged.

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
