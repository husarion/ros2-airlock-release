# Move a config to schema 7

From engine 1.24.0 the airlock reads one schema, 7. A file at schema 1 to 6 is refused when a half starts and by `airlock check`, with exit code 2 and the command that rewrites it:

```text
airlock: config invalid: airlock.yaml: schema 5 is no longer accepted (7 only since 1.24.0): run 'ros2 airlock migrate airlock.yaml' (schema/MIGRATION.md)
```

Run it once per file:

```bash
source /opt/ros/jazzy/setup.bash
ros2 airlock migrate airlock.yaml
```

`migrate` writes `airlock.schema7.yaml` beside your file (`--in-place` rewrites the file and keeps a `.bak`, `-o <path>` names the output), and rewrites the fragments of an include directory the same way. It keeps your comments and the order of your entries. It prints one line for every change, what the new file reserves in shared memory and which entries cost the most, and then runs `airlock check` on the result. Exit code 0 means the new file is written and checks clean; 3 means it is written and something needs your attention, which the output names; 2 means it could not read or write a file. Running it on a schema 7 file changes nothing.

## What `migrate` writes for you

- `schema: 7`, in the file and in every fragment.
- An explicit `max_response_bytes` on every service and action entry that has none, with a `# migrate:` comment saying why. Before schema 7 every service reserved a fixed 10 MiB response channel. Schema 7 sizes it from the entry, and an entry without the key gets its request cap, which is too small for a service that answers with a list. So `migrate` writes the largest response the running halves have seen, rounded up to a power of two, when it can read their status files (`--status robot=<file>` or `--ipc-dir`); otherwise 10 MiB when the file's budget can afford it, and 64 KiB when it cannot. Check the value against what your service really returns.
- A higher `airlock.max_lanes` with a comment, when the file names more entries than the old value allows (schema 7 counts an action as five lanes).

## What it cannot write for you

- **Caps on topics.** Schema 7 reserves every exact entry's shared memory when a half starts (see below), and a topic without `max_bytes` reserves 10 MiB per slot. `migrate` lists such topics by cost and leaves the cap to you; the status file's `max_frame_bytes` on a running lane says what the topic really sends.
- **A type on an entry into the listener's world.** Every topic, service and action entry into the protected world names its `type`, and a topic glob into it lists `types: [...]`. `migrate` names the entries that lack one; it never guesses.
- **Durability.** An outbound lane is volatile unless its entry says `qos: { durability: transient_local }`. Before schema 6 a lane copied the durability of the publisher it found. Add the key to every lane that must latch, such as `/robot_description`, an e-stop state or a map.
- **Globs.** `*` matches within one name segment and a segment that is exactly `**` matches one or more whole segments. Before schema 6, `/*` matched everything. `migrate` names every glob whose reach changes; write `/**` where you meant everything.

## Before and after

A schema 5 file:

```text
schema: 5
airlock:
  ipc: { dir: /run/airlock, listener: robot }
  worlds:
    robot:
      metrics: { bind: "127.0.0.1:9464" }
    fleet: {}
  directions:
    - from: robot
      to: fleet
      namespace: /panther_1234
      topics:
        - topic: /*
          max_hz: 100
        - topic: /robot_description
          activation: always
      services:
        - service: /controller_manager/list_controllers
          type: controller_manager_msgs/srv/ListControllers
          timeout_s: 5.0
          max_request_bytes: 1024
    - from: fleet
      to: robot
      topics:
        - topic: /panther_1234/cmd_vel
          rename: /cmd_vel
          max_hz: 50
          max_bytes: 4096
```

The same wall at schema 7, after `migrate` and the edits it asked for:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 7
airlock:
  ipc: { dir: /run/airlock, listener: robot, queue_depth: 2 }
  worlds:
    robot:
      metrics: { bind: "127.0.0.1:9464" }
    fleet: {}
  directions:
    - from: robot
      to: fleet
      namespace: /panther_1234
      exclude: ["@ros-noise", "@debug"]
      topics:
        - topic: /**
          max_hz: 100
        - topic: /robot_description
          activation: always
          max_bytes: 1048576
          qos: { durability: transient_local }
      services:
        - service: /controller_manager/list_controllers
          type: controller_manager_msgs/srv/ListControllers
          timeout_s: 5.0
          max_request_bytes: 1024
          max_response_bytes: 65536 # a list answers with more than its request; migrate wrote 10 MiB here, 64 KiB is plenty
    - from: fleet
      to: robot
      topics:
        - topic: /panther_1234/cmd_vel
          rename: /cmd_vel
          type: geometry_msgs/msg/TwistStamped
          max_hz: 50
          max_bytes: 4096
```

## Shared memory is reserved when a half starts

Every channel of an exact entry is reserved when the file loads: `(cap + 24) x (queue_depth + 3)` bytes for a topic and `(cap + 32) x (max(queue_depth, max_inflight) + 3)` for each of a service's request and response channels, plus a fixed cost per channel that `ros2 run ros2_airlock airlock_half --constants` prints. `airlock check` prints the total as a note, with the largest entries. A file whose reservation is larger than `ipc.shm_budget` is refused with exit code 2 and the entries named, largest first; lower their caps or `queue_depth`, or share them through a glob, which is admitted while it runs and never reserved. A half also refuses to start when the reservation is larger than its host's `/dev/shm`. Docker gives a container 64 MiB of `/dev/shm` by default, so in Docker give the halves a larger one (`docker run --shm-size=1g`, or `shm_size: 1gb` on the container that owns the shared IPC namespace in compose); the refusal prints both numbers and the size to ask for.

`airlock.max_lanes` counts every lane one half holds, an action as five, and the exact entries are counted first. A file whose exact entries alone need more lanes than `max_lanes` is refused.

## New keys

| Key | Where | What it does |
| -- | -- | -- |
| `max_response_bytes` | service or action entry | The largest response the lane carries. Default: the entry's request cap; an action's `send_goal` and `get_result` take 65536 and `cancel_goal` keeps 4096 |
| `service_idle_s` | `airlock` | Seconds without a call before a service's channels are released, 0 to 3600, default 30, 0 for never |
| `service:` or `action:` glob | services, actions | A pattern over the service or action name, with `types`, `max_matches` and a `timeout_s` of at least 0.5 s; a glob into the listener's world needs both `types` and `max_matches` |
| `topic:`, `service:`, `action:` prefix | an `exclude` item | Limits the item to one kind; a bare item applies to every kind |

The keys schema 6 added (`exclude`, `allow_open_inbound`, `types`, `finite`, `allow_reserved`, `max_inflight`, `expires`, `include_dir`, `managed_by`, `min_peer_minor`, `metrics.allow_untrusted_bind`) keep their meaning. A reserved or hidden destination (`/clock`, `/rosout`, `/parameter_events`, a segment starting with `_`) needs `allow_reserved: true`; an `ipc.dir` or forensics directory owned by another user or readable by others is refused; a `/metrics` bind outside loopback on the connector half needs `metrics.allow_untrusted_bind: true`.

## Fragments in the include directory

With `include_dir: airlock.yaml.d`, both halves read every `*.yaml` in that directory, in byte-wise name order, after the base file. A fragment holds `schema: 7` and `airlock: { directions: [...] }` and nothing else, and its schema must equal the base file's. Its directions merge with the base by the usual rules, so a topic stated twice in one direction is still an error across files. At most 256 fragments of 1 MiB each; a symlink or a group- or other-writable fragment is refused. Any error in any file refuses the whole load, and on SIGHUP the running config stays.

## Both halves move together

An engine older than 1.24.0 refuses a schema 7 file, and a 1.24.0 engine refuses every older one. The two halves also speak one control protocol, 1.8, and a half refuses a peer that speaks an older one (`proto_minor` in its status and in `airlock doctor`). Upgrade both halves and move the file in one step. To go back to an older engine, render the file for that engine again.
