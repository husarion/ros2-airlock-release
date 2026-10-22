# Keep camera traffic away from tf

A camera or a point cloud moves more bytes in a second than every other topic of a robot together. On a small computer that traffic can hold the threads that also carry `/tf`, and the robot's transforms arrive late or in bursts. From engine 1.25.0 (config schema 8) the airlock sorts every topic lane into a class and serves the classes apart, and it caps the bytes per second of the heavy ones. This page explains the classes, the defaults, how to change them and where to see what a lane is doing.

## Three classes

Every topic lane has one class:

- `control`: `/tf`, `/tf_static`, every entry with `protected: true`, and every service and action. These are served first and never change class.
- `normal`: everything that is neither control nor bulk.
- `bulk`: images, point clouds and other large messages. A bulk lane runs on its own executor thread at a lower priority and is capped in bytes per second.

A lane gets its class from its entry, in this order:

1. `class:` written on the entry.
2. `control` for `/tf` and `/tf_static`, for `protected: true`, and for every service and action.
3. By the entry's own `max_bytes` when it writes one: `bulk` at 256 KiB (`airlock.bulk_threshold_bytes`) or more, `normal` below it. You stated the size, so the type list does not override it: a 16 KiB `sensor_msgs/msg/Image` that drives the lights is `normal`.
4. `bulk` when its type is in the bulk type list: `sensor_msgs/msg/Image`, `sensor_msgs/msg/CompressedImage`, `sensor_msgs/msg/PointCloud2`, `sensor_msgs/msg/PointCloud`, `stereo_msgs/msg/DisparityImage`, `nav_msgs/msg/OccupancyGrid` and `theora_image_transport/msg/Packet`. The list ships as `share/ros2_airlock/bulk_types.yaml`; add your own types in a `bulk_types.d/` directory beside your config, one or more `*.yaml` files of the same shape (`types: [pkg/msg/Name, ...]`).
5. `normal` otherwise.

A glob entry has no type in the file, so each lane it carries is classed when the lane starts, by the type it finds. A `normal` lane whose sample turns out larger than the threshold becomes `bulk` from that sample on (`class_reason: observed` in the status) until the next reload. A lane whose entry writes `class: normal` is never moved.

## The defaults

| What | Default | Key |
| -- | -- | -- |
| Cap of a bulk lane | 4 MiB/s (4194304 B/s) | `airlock.bulk_max_bytes_per_s`, or `max_bytes_per_s` on the entry |
| Size that makes a lane bulk | 256 KiB (262144 B) | `airlock.bulk_threshold_bytes` |
| Threads serving bulk lanes | 1 | `airlock.bulk_threads` (1 to 8; a change needs a restart) |
| Cap of a control or normal lane | none | `max_bytes_per_s` on the entry |

At 4 MiB/s a 1 MiB image flows at 4 Hz and a 256 KiB one at 16 Hz. The number sits between what a Raspberry Pi 4 robot carried without hurting `/tf` (2.6 MB/s) and the rate that stalled it (10.5 MB/s).

The cap keeps the newest sample. When a publisher sends faster than the cap allows, the source half holds the newest sample until the cap admits it and drops the older ones, counting them in `dropped_cap`. It never blocks the publisher and never queues a backlog in shared memory. One sample of the entry's `max_bytes` always fits, and the cap admits a short burst of up to `max(max_bytes, cap / 4)` bytes at full speed. If you want a tighter burst, set `max_bytes` to the topic's real size; `max_frame_bytes` in the status says what that is.

## Change a class or a cap

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 9
airlock:
  ipc: { dir: /run/airlock, listener: robot, queue_depth: 2 }
  bulk_max_bytes_per_s: 4194304 # the default, written out: every bulk lane without its own cap
  worlds: { robot: {}, fleet: {} }
  directions:
    - from: robot
      to: fleet
      namespace: /lynx_0001
      tf: { namespaced: true }
      topics:
        - topic: /odometry/filtered
          type: nav_msgs/msg/Odometry
          max_bytes: 4096
        - topic: /camera/color/image_raw/compressed
          type: sensor_msgs/msg/CompressedImage
          max_bytes: 524288
        - topic: /camera/depth/points
          type: sensor_msgs/msg/PointCloud2
          max_bytes: 2097152
          max_bytes_per_s: 16777216 # 16 MiB/s: this robot has the CPU for it
        - topic: /map
          type: nav_msgs/msg/OccupancyGrid
          max_bytes: 1048576
          max_bytes_per_s: 0 # a map changes rarely: no cap
        - topic: /lights/frame
          type: sensor_msgs/msg/Image
          max_bytes: 16384
          class: normal # a small image that drives the lights: never behind camera traffic
      services:
        - service: /hardware/e_stop_trigger
          type: std_srvs/srv/Trigger
          timeout_s: 2.0
          max_request_bytes: 1024
```

- Raise one lane's cap with `max_bytes_per_s` on its entry. The value is 0 for no cap, or 65536 and more.
- Lift a cap with `max_bytes_per_s: 0`.
- Change the cap of every bulk lane with `airlock.bulk_max_bytes_per_s`.
- Keep a bulk-typed topic out of the bulk class with `class: normal`, or put any topic in it with `class: bulk`.

A cap or class change is applied on reload (SIGHUP) without dropping the lane. Only `bulk_threads` needs a restart.

## See the plan before you deploy

`ros2 airlock check <file> --plan` prints every entry's class, the reason and the cap it runs at. For the file above:

```text
robot -> fleet  topic /odometry/filtered  class: normal  class_reason: max_bytes  cap: unlimited  (promoted to bulk past 262144 B)
robot -> fleet  topic /camera/color/image_raw/compressed  class: bulk  class_reason: max_bytes  cap: 4194304 B/s
robot -> fleet  topic /camera/depth/points  class: bulk  class_reason: max_bytes  cap: 16777216 B/s
robot -> fleet  topic /map  class: bulk  class_reason: max_bytes  cap: unlimited
robot -> fleet  topic /lights/frame  class: normal  class_reason: explicit  cap: unlimited
robot -> fleet  topic /tf  class: control  class_reason: tf  cap: unlimited
robot -> fleet  topic /tf_static  class: control  class_reason: tf  cap: unlimited
robot -> fleet  service /hardware/e_stop_trigger  class: control  class_reason: service
```

A glob prints `class: per-lane`. A bulk entry with `activation: always` is accepted, and `check` notes it as a standing cost: it takes from the start whether or not anyone subscribes.

`ros2 airlock check <file> --plan --format json` prints the same as one JSON document, for a tool that shows the plan before a file is applied; its keys and values are in [the check plan contract](check-plan-contract.md).

## Watch it run

`ros2 airlock top` shows each lane's class, the delivery age (how old a sample is when the destination world sees it, p50 and p99) and the largest gap between two samples in the last 5 seconds, with a line per class below the table. `ros2 airlock top --class bulk` shows only the bulk lanes, `--class control` only the control ones. `ros2 airlock api` lists the class of every topic, service and action it reports.

Each half's status file carries the numbers:

| Field | Row | Meaning |
| -- | -- | -- |
| `class`, `class_reason` | every `tx`, `rx` and `services` row | The lane's class and why it has it |
| `cap_bytes_per_s` | `tx`, `rx` | The lane's cap; 0 for none |
| `dropped_cap`, `dropped_cap_bytes` | `tx` | Samples (and their bytes) the cap dropped at the source |
| `cap_held` | `tx` | 1 while a sample waits for the cap |
| `cap_fill` | `tx` | Bytes the cap would admit right now |
| `executor` | `tx` | `exb` for the bulk thread, `exc` for the others |
| `class_moves` | `tx` | How often the lane moved class while running |
| `delivery_age_p50_us`, `delivery_age_p99_us`, `delivery_age_max_us`, `delivery_age_samples` | `rx` | Delivery age over the last 5 s; the three ages are null when the window carried no sample (`delivery_age_samples` 0) |
| `max_gap_us` | `rx` | Largest interval between two samples in the last 5 s; null when the window carried no sample and in the window of the lane's first sample |
| `dropped_cap`, `cap_exceeded`, `cap_bound_bytes_per_s` | `rx` | Frames the destination dropped because the peer half sent past twice the cap, and how often |
| `classes` | top level | Per class: `lanes`, `forwarded`, `dropped_cap`, `delivery_age_p99_us`, `max_gap_us` (the last two null when no lane of the class carried in the window) |

The history records `class_promoted` when a lane moves to bulk while it runs, and `cap_exceeded` when a peer sends past the bound.

## What doctor says

- `lane.cap_dropping`: the cap dropped more than 10 % of the samples a lane was offered. The lane still delivers the newest sample; if you need more of them, raise `max_bytes_per_s` on its entry or lower the publisher's rate.
- `lane.cap_exceeded_by_peer`: the destination half dropped frames because the other half sent more than twice the lane's cap. A half running the same config never does that, so look at the other half.

## Threads and CPU

The bulk lanes' source callbacks run on the bulk executor's threads, named `alk-exb-<n>`, and their destination pumps are named `alk-rxb-<n>`. Both run at nice +10, so the scheduler prefers the control and normal threads (`alk-exc-<n>`, `alk-rxc-<n>`, `alk-rxn-<n>`) when the CPU is short. A bulk callback takes one sample at a time, so a burst of images cannot hold a thread for long. CPU placement is yours: a pinner can confine `alk-exb-*` and `alk-rxb-*` to the cores you can spare for camera traffic ([Threads and CPU pinning](threads-and-pinning.md)).

## A peer that ignores the cap

The destination half of an inbound lane checks the same cap on arrival, with twice the cap as its bound. A fleet half that sends more is dropped at that bound, each excess counts as a `cap_exceeded` strike, and repeated strikes (17 within 10 s) retire the peer as `peer_invalid`, the same escalation as for invalid frames. While a reload is in flight on one half the bound still drops but never strikes, so lowering a cap cannot cost you the session. A peer cannot raise a lane's class either: each half computes the class from its own file.
