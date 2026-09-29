# Harden — the airlock pattern

The airlock pattern is the production profile this project exists for: the robot's full API readable, a tiny enumerated command surface writable, everything else nonexistent, and evidence kept at the boundary. This is the whole shape, schema 2:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 2
airlock:
  ipc: { dir: /run/airlock, listener: robot, shm_budget: 268435456 }
  forensics: { dir: /var/lib/airlock, bag_max_bytes: 268435456, bag_split_bytes: 67108864 }

  worlds:
    robot:
      metrics: { bind: "127.0.0.1:9464" }   # protected side only; scrape locally, relay yourself
    fleet:
      diagnostics: false                    # no channel lists or drop counters for reconnaissance

  directions:
    # Outbound: the whole driver API, readable but lazy — data crosses only
    # for topics something in the fleet actually consumes.
    - from: robot
      to: fleet
      namespace: /panther_1234
      topics:
        - topic: /*
          max_bytes: 10485760
          max_hz: 100
        - topic: /robot_description
          activation: always
          qos: { durability: transient_local }
      tf: { namespaced: true, merged: true, shared_frames: [] }
      services:
        - service: /hardware/e_stop_trigger
          type: std_srvs/srv/Trigger
          timeout_s: 2.0
          max_request_bytes: 1024

    # Inbound: exact names, declared types, hard caps, full forensics.
    # Nothing else inbound exists.
    - from: fleet
      to: robot
      namespace: ""
      audit: { ring_bytes: 16777216 }
      topics:
        - topic: /panther_1234/cmd_vel
          rename: /cmd_vel
          type: geometry_msgs/msg/TwistStamped
          qos: { reliability: best_effort, depth: 1 }
          max_hz: 50
          max_bytes: 4096
          record: true
          deadman:
            ms: 500
            hz: 10
            neutral: { twist: { linear: { x: 0.0 } } }
      services: []
```

This profile survived the incident it was designed against: 30 minutes of sustained flood in three attack variants, with the robot's sensor stream holding above 99% of its rate, the robot world invisible throughout, and services still answering. That test is a repeatable gate in the release pipeline.

## The asymmetry, knob by knob

**Outbound is generous, inbound is enumerated.** The glob exposes the whole API but lazily: an unconsumed topic costs nothing, and preallocates nothing. Inbound entries are exact name plus declared type, so the writable surface is a list you can read aloud. Inbound `services` stays empty unless the robot genuinely needs to call outward.

**Every inbound entry is capped three ways.** `max_hz` decimation, `max_bytes` size cap, `depth: 1` queue. Enforced in the fleet-side half and enforced again in the robot-side half, because the fleet-side half is the sacrificial component: a flood may saturate it, but what reaches the robot side is bounded by the caps regardless.

**`diagnostics: false` on the untrusted side.** Channel lists, rates, and drop counters are reconnaissance data. The robot-side half still publishes full diagnostics into the robot world.

**Forensics keep evidence without becoming a target.** The `audit` ring is a bounded on-disk log of everything deny-by-default ever sees inbound: each admitted message's sender GID, every publisher that appeared on a denied name, over-cap drops. `record: true` keeps a size-bounded rosbag2 of exactly the bytes that entered the robot world, post-enforcement, so the caps that protect the robot also bound the evidence. The `deadman` publishes the configured neutral payload at 10 Hz if the command stream dies more than 500 ms, and arms only after the first real command. One caveat: a deadman belongs on a command topic only when the consumer has no dead-source handling of its own. If the topic feeds a twist_mux input, the stream of neutral zeros keeps that source active and locks out lower-priority sources; let the mux's timeout do its job there. Mount `forensics.dir` on a persistent volume, not tmpfs.

**`metrics` binds on the protected side, on localhost.** Your on-robot agent scrapes and relays; no new listener opens toward the untrusted network.

**`shm_budget` caps transport memory.** Channels preallocate `max_bytes × depth`; an activation that would exceed the budget is refused with a diagnostic instead of exhausting `/dev/shm`.

## When the fleet-side half misbehaves

The pattern treats the fleet-side half as sacrificial, so the robot-side half does not trust it either. That half binds the control socket (`ipc.listener: robot`), and since 1.22.0 it checks everything the other half hands it.

**Every message into the robot is validated.** Before a message from the fleet-side half is published into the robot world, the robot half copies it out of shared memory and parses it under its own definition of the type the entry declares. A message that does not parse, or that carries more than a few bytes of padding past its end, is dropped and counted as `dropped_invalid` on the lane. Under schema 6 a message carrying a NaN or an infinity is dropped too (`dropped_nonfinite`), unless the entry sets `finite: false`. Validation proves the bytes are a well-formed message of that type, not that the values are sensible: a well-formed `cmd_vel` of 100 m/s passes, so keep the rate cap, the deadman and your controller's own limits. An inbound type the robot half's image cannot describe is refused on its lane with the reason `unvalidatable` (`airlock check` names such types).

**Inbound lanes have a rate cap even when you set none.** An inbound topic without `max_hz` runs at most 1000 messages per second, with short bursts allowed, and all inbound lanes of a half share 64 MiB/s. `max_hz: 0` removes the per-lane cap for a lane that really needs it. A service answers at most `max_inflight` calls at once (32 by default, a schema 6 key); past that the caller gets an immediate reply whose `message` starts with `airlock: busy`.

**Misbehaviour is counted, and repeated misbehaviour is cut off.** Malformed frames, oversized frames, a type other than the configured one, a squatted shared-memory port, a flood of empty wakeups, invalid names and unknown activations on the control connection are each counted by kind, and each kind is logged at most once per 10 s. The 17th bad frame on one lane within 10 s pauses that lane (reason `peer_invalid`) and retries it after 1 s, doubling up to 60 s. The 65th control-plane violation within 10 s ends the connection; the robot half then refuses the fleet-side half for 1 s, doubling up to 60 s, and resets after five clean minutes. During that backoff the robot world keeps running untouched and health reads `dysfunctional` with the reason `peer_violations`. A connection from an older half that cannot carry data to 1.22.0 (protocol below 1.5, engines 1.15 and older) is refused the same way.

**Where to look.** `airlock doctor` prints `peer.violations` with the kinds, `peer.backoff` with the seconds left, `peer.cred` when the other half runs as a different uid (a warning below schema 6, a refused connection under 6) and `lane.inbound_invalid` for the dropped frames. The status file carries them under `peer` (`violations`, `session_ends`, `session_refusals`, `backoff_s`, `cred`), `/metrics` as `airlock_peer_violations_total{kind}`, and the robot half keeps the last 256 records in `violations_robot.jsonl` in the forensics directory, where a data flood that rotates the black box cannot evict them. None of these need a restart of the robot half: fix or restart the fleet-side half.

**What it does not cover.** Both halves share one shared-memory root and one uid. A half that corrupts the shared-memory transport's own bookkeeping, rather than the messages in it, can still crash the robot-side half; the separate containers and the supervisor's independent restart are the containment there. And a well-behaved fleet-side half still has full authority over every lane and service you allowed, within their caps: the airlock bounds what may cross, it does not decide who may send it.

## The deployment is half the pattern

Config bounds what crosses; deployment removes the robot world from the network entirely.

- Run the robot world with no network interface at all. In compose: an anchor container with `network_mode: none` and `ipc: shareable`, every robot-world process joining it via `network_mode: service:<anchor>` and `ipc: service:<anchor>`. The fleet-side half joins the anchor's IPC namespace (that shared `/dev/shm` is the bridge) plus the real network.
- Use a shared-memory-only RMW profile in the robot world, so no robot-world process ever binds a socket, discovery included. The package ships the tuned profile at `share/ros2_airlock/profiles/robot_world/fastdds_shm_only.xml`; point `FASTRTPS_DEFAULT_PROFILES_FILE` at it in every robot-world container.
- Cap both halves' memory with cgroup limits (compose `mem_limit`) on top of the internal caps, and let the supervisor restart them; either half can be killed at any moment, in any order, and the pair re-converges.
- Set an explicit `shm_size` on the shared IPC namespace and keep `ipc.dir` short (it holds Unix sockets, which have a 108-byte path limit).
- Run both halves as one uid, each in its own container (its own PID namespace), and run nothing else in the robot world as that uid. The image runs as user `airlock` (uid 10001) by default, and `/run/airlock` and `/var/lib/airlock` exist in it owned by 10001 with mode 0700, so a named volume mounted there starts private to the halves. `ipc.dir` and the forensics directory must not be writable by group or others (a half refuses to start, exit 2), and should be owned by the halves' uid with mode 0700 (a warning below config schema 6, a refusal under 6).
- Root is an opt-in: `docker run --user 0` or compose `user: "0"`, for both halves. You need it when the robot world's own processes run as root over Fast DDS shared memory: a participant must write into its peers' shared-memory segments, so a half on another uid sees only part of that world. Husarion UGV OS is such a world: its driver runs as root over shared memory only, so from the UGV OS release that ships engine 1.22.0 it runs both halves `--user 0`. The half says so in its log and status (`world.shm_foreign_owner`, and `airlock doctor` prints it with the remedy). A root half's `ipc.dir` volume must then be root's too: a named volume initialized from the image belongs to 10001, which is a warning below schema 6 and refused under 6 (chown it, or bind-mount a root-owned directory).

Domain isolation, `ROS_AUTOMATIC_DISCOVERY_RANGE`, and receive-side RMW caps are accident prevention, not isolation: a peer on your domain can still flood the wire, and traffic on the wire steals CPU from the control loop before any cap applies. The airlock pattern is the topology where that traffic has no path to the robot at all.
