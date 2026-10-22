# Move a config to schema 9

From engine 1.26.0 the airlock reads one schema, 9. A file at schema 1 to 8 is refused when a half starts and by `airlock check`, with exit code 2 and the command that rewrites it:

```text
airlock: config invalid: airlock.yaml: schema 8 is no longer accepted (9 only since 1.26.0): run 'ros2 airlock migrate airlock.yaml' (schema/MIGRATION.md)
```

Run it once per file:

```bash
source /opt/ros/jazzy/setup.bash
ros2 airlock migrate airlock.yaml
```

`migrate` writes `airlock.schema9.yaml` beside your file (`--in-place` rewrites the file and keeps a `.bak`, `-o <path>` names the output), and rewrites the fragments of an include directory the same way, into `<include_dir>.schema9/`. It reads every schema from 1 to 8 and goes straight to 9 in one run, so a file from 1.23.0 never needs an older release's tool first. It keeps your comments and the order of your entries. It prints one line for every change, what the new file reserves in shared memory and which entries cost the most, and then runs `airlock check` on the result. Exit code 0 means the new file is written and checks clean; 3 means it is written and something needs your attention, which the output names; 2 means it could not read or write a file. Running it on a schema 9 file changes nothing.

## From schema 8: the version line

Schema 9 means everything schema 8 meant. On a schema 8 file `migrate` changes the `schema:` line of the file and of every fragment and nothing else, and writes no `# migrate:` comment. The two numbers exist so that a half can tell its peer which file format it reads: a 1.26.0 half refuses a peer that reads another one at once and by name (`config_schema` in its status and in `airlock doctor`), instead of running for 30 seconds on two different files.

## From schema 1 to 7: what schema 8 changed

The rest of this page is for a file older than schema 8. `migrate` makes these changes and the version-line change in the same run.

Schema 8 gives every topic lane a class, `control`, `normal` or `bulk`, and gives `bulk` lanes a byte cap of 4 MiB/s by default. A camera or point cloud entry that flowed at its publisher's full rate under schema 7 is now shaped to that cap, and it runs on its own thread after the control traffic. [Keep camera traffic away from tf](bounded-bulk-lanes.md) explains the classes, the cap and how to read them in the status. Because this changes what an existing file does, the new file says it explicitly.

## What `migrate` writes for you

- `schema: 9`, in the file and in every fragment.
- `class: bulk` and `max_bytes_per_s: 4194304` on every exact topic entry that schema 8 classes as bulk, each with a `# migrate:` comment saying why. An entry is bulk when it sets `max_bytes` to 256 KiB or more, or when it sets no `max_bytes` and its type is an image, a point cloud, a disparity image, an occupancy grid or a theora packet (the list is `share/ros2_airlock/bulk_types.yaml`, and your own `bulk_types.d/` beside the config adds to it). An image entry with a `max_bytes` below 256 KiB stays normal and gets nothing. `migrate` lists every such entry. Raise the cap on the ones that need more, or set `max_bytes_per_s: 0` to lift it.
- From a schema 1 to 6 file only: an explicit `max_response_bytes` on every service and action entry that has none, with a `# migrate:` comment saying why. Before schema 7 every service reserved a fixed 10 MiB response channel. Since schema 7 the channel is sized from the entry, and an entry without the key gets its request cap, which is too small for a service that answers with a list. So `migrate` writes the largest response the running halves have seen, rounded up to a power of two, when it can read their status files (`--status robot=<file>` or `--ipc-dir`); otherwise 10 MiB when the file's budget can afford it, and 64 KiB when it cannot. Check the value against what your service really returns. A schema 7 file already has the right default and gets nothing here.
- From a schema 1 to 6 file only: a higher `airlock.max_lanes` with a comment, when the file names more entries than the old value allows (an action counts as five lanes).

## What it cannot write for you

- **A class for a glob.** A glob entry has no type in the file, so each lane it carries is classed when the lane starts, by the type it finds. `migrate` leaves globs alone; `airlock check --plan` prints `class: per-lane` for them and `ros2 airlock top` shows each lane's class while it runs.
- **Whether a bulk-typed entry really is bulk.** A small image that carries a command, such as a light pattern, is bulk by its type. Write `class: normal` on such an entry if it must not wait behind camera traffic.
- **Caps on topics.** Every exact entry's shared memory is reserved when a half starts (see below), and a topic without `max_bytes` reserves 10 MiB per slot. `migrate` lists such topics by cost and leaves the cap to you; the status file's `max_frame_bytes` on a running lane says what the topic really sends, and `migrate --status` prints it for a bulk entry without `max_bytes`.
- **A type on an entry into the listener's world.** Every topic, service and action entry into the protected world names its `type`, and a topic glob into it lists `types: [...]`. `migrate` names the entries that lack one; it never guesses.
- **Durability**, from a file older than schema 6. An outbound lane is volatile unless its entry says `qos: { durability: transient_local }`. Before schema 6 a lane copied the durability of the publisher it found. Add the key to every lane that must latch, such as `/robot_description`, an e-stop state or a map.
- **Globs**, from a file older than schema 6. `*` matches within one name segment and a segment that is exactly `**` matches one or more whole segments. Before schema 6, `/*` matched everything. `migrate` names every glob whose reach changes; write `/**` where you meant everything.

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
        - topic: /camera/image_raw/compressed
          type: sensor_msgs/msg/CompressedImage
          max_bytes: 1048576
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

The same wall at schema 9, after `migrate` and the edits it asked for:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 9
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
          max_bytes: 131072
          qos: { durability: transient_local }
        - topic: /camera/image_raw/compressed
          type: sensor_msgs/msg/CompressedImage
          max_bytes: 1048576
          class: bulk # migrate wrote this: its max_bytes is at or above the bulk threshold
          max_bytes_per_s: 4194304 # and the 4 MiB/s default cap, which the file never had
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

## Keys schema 8 added

| Key | Where | What it does |
| -- | -- | -- |
| `class` | topic entry | `control`, `normal` or `bulk`, over the rules. Not on a service or action entry, which is always `control` |
| `max_bytes_per_s` | topic entry | The lane's byte cap per second, 0 for none, else at least 65536. Default: `airlock.bulk_max_bytes_per_s` on a bulk lane, no cap on the others |
| `bulk_max_bytes_per_s` | `airlock` | The cap of every bulk lane that sets none, default 4194304 (4 MiB/s) |
| `bulk_threshold_bytes` | `airlock` | An explicit `max_bytes` at or above it makes an entry bulk (below it, normal, whatever the type), and a normal lane whose sample is larger becomes bulk while it runs; default 262144 (256 KiB) |
| `bulk_threads` | `airlock` | The threads that serve bulk lanes, 1 to 8, default 1; a change needs a restart |

The keys schema 7 added (`max_response_bytes`, `service_idle_s`, service and action globs, `topic:`/`service:`/`action:` prefixes on an `exclude` item) and the ones schema 6 added (`exclude`, `allow_open_inbound`, `types`, `finite`, `allow_reserved`, `max_inflight`, `expires`, `include_dir`, `managed_by`, `min_peer_minor`, `metrics.allow_untrusted_bind`) keep their meaning. A reserved or hidden destination (`/clock`, `/rosout`, `/parameter_events`, a segment starting with `_`) needs `allow_reserved: true`; an `ipc.dir` or forensics directory owned by another user or readable by others is refused; a `/metrics` bind outside loopback on the connector half needs `metrics.allow_untrusted_bind: true`.

## Fragments in the include directory

With `include_dir: airlock.yaml.d`, both halves read every `*.yaml` in that directory, in byte-wise name order, after the base file. A fragment holds `schema: 9` and `airlock: { directions: [...] }` and nothing else, and its schema must equal the base file's. Its directions merge with the base by the usual rules, so a topic stated twice in one direction is still an error across files. At most 256 fragments of 1 MiB each; a symlink or a group- or other-writable fragment is refused. Any error in any file refuses the whole load, and on SIGHUP the running config stays.

## Both halves move together

An engine older than 1.26.0 refuses a schema 9 file, and a 1.26.0 engine refuses every older one. A 1.26.0 half also refuses to pair with a half that reads another schema, before anything crosses (`config_schema` in its status, its log and `airlock doctor`, naming both schemas and both engine versions); neither half exits, and nothing crosses until both run the same release. A half refuses a peer that speaks an older control protocol the same way (`proto_minor`). Upgrade both halves and move the file in one step. To go back to an older engine, render the file for that engine again (for 1.25.x, `schema: 8` is the only change).
