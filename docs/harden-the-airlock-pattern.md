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

## The deployment is half the pattern

Config bounds what crosses; deployment removes the robot world from the network entirely.

- Run the robot world with no network interface at all. In compose: an anchor container with `network_mode: none` and `ipc: shareable`, every robot-world process joining it via `network_mode: service:<anchor>` and `ipc: service:<anchor>`. The fleet-side half joins the anchor's IPC namespace (that shared `/dev/shm` is the bridge) plus the real network.
- Use a shared-memory-only RMW profile in the robot world, so no robot-world process ever binds a socket, discovery included. The package ships the tuned profile at `share/ros2_airlock/profiles/robot_world/fastdds_shm_only.xml`; point `FASTRTPS_DEFAULT_PROFILES_FILE` at it in every robot-world container.
- Cap both halves' memory with cgroup limits (compose `mem_limit`) on top of the internal caps, and let the supervisor restart them; either half can be killed at any moment, in any order, and the pair re-converges.
- Set an explicit `shm_size` on the shared IPC namespace and keep `ipc.dir` short (it holds Unix sockets, which have a 108-byte path limit).

Domain isolation, `ROS_AUTOMATIC_DISCOVERY_RANGE`, and receive-side RMW caps are accident prevention, not isolation: a peer on your domain can still flood the wire, and traffic on the wire steals CPU from the control loop before any cap applies. The airlock pattern is the topology where that traffic has no path to the robot at all.
