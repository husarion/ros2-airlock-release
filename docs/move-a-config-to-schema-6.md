# Move a config to schema 6

Schema 6 arrived with engine 1.22.0. A config at schema 1 to 5 keeps its meaning on 1.22.0, so you move when you want what 6 adds: engine-ended trials and temporary lanes, `exclude:` with presets, fragment files the tools can write, globs that respect name segments, and a stricter inbound surface. Ask the engine what the move would change for your file first:

```bash
source /opt/ros/jazzy/setup.bash
ros2 airlock check airlock.yaml
```

```text
note: schema 6 would change: topic '/*' (direction robot->fleet) becomes volatile (today it adapts to the source's durability); add qos: {durability: transient_local} if it must latch
note: schema 6 would change: glob '/*' (direction robot->fleet) matches within one segment ('*') and never a reserved name; '/**' keeps today's reach across '/'
note: schema 6 would change: topic '/robot_description' (direction robot->fleet) becomes volatile (today it adapts to the source's durability); add qos: {durability: transient_local} if it must latch
note: schema 6 would change: topic '/panther_1234/cmd_vel' (direction fleet->robot) writes into the listener's world and needs a 'type'
airlock: airlock.yaml is valid (schema 5, epoch 7a5e03d2525844cd)
```

The `note:` lines never change the exit code, so any script that runs `airlock check` today keeps passing. When the halves are running, `check` also reads their status files (from `ipc.dir`, or `--status <file>`) and lists the actual names each glob would stop matching, and the lanes that latch today so that a subscriber asking for transient_local (rviz on `/map`, for example) would stop matching them.

## Before and after

This schema 5 file is the one the notes above were printed for:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
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
        - service: /hardware/e_stop_trigger
          type: std_srvs/srv/Trigger
          timeout_s: 2.0
    - from: fleet
      to: robot
      topics:
        - topic: /panther_1234/cmd_vel
          rename: /cmd_vel
          max_hz: 50
          max_bytes: 4096
```

The same wall at schema 6:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 6
airlock:
  ipc: { dir: /run/airlock, listener: robot, min_peer_minor: 6 }
  worlds:
    robot:
      metrics: { bind: "127.0.0.1:9464" }
    fleet: {}
  directions:
    - from: robot
      to: fleet
      namespace: /panther_1234
      exclude: ["@ros-noise", "@debug"]       # glob matches only; exact entries are never excluded
      topics:
        - topic: /**                          # `*` no longer crosses `/`
          max_hz: 100
        - topic: /robot_description
          activation: always
          qos: { durability: transient_local } # outbound lanes are volatile unless they say so
      services:
        - service: /hardware/e_stop_trigger
          type: std_srvs/srv/Trigger
          timeout_s: 2.0
          max_inflight: 4                     # calls in flight; past it, a marked busy reply
    - from: fleet
      to: robot
      topics:
        - topic: /panther_1234/cmd_vel
          rename: /cmd_vel
          type: geometry_msgs/msg/TwistStamped # required on every entry into the robot
          max_hz: 50
          max_bytes: 4096
```

## What changes meaning

- **Outbound lanes are volatile unless configured.** Under 1–5 an outbound lane copies the durability of the publisher it found. Under 6 it is volatile unless its entry says `qos: { durability: transient_local }`. Add that to every lane that must latch, such as `/robot_description`, an e-stop state or a map. The `tf` block keeps handling `/tf_static` itself. Lanes into the listener's world (the robot) already work this way on every schema since 1.21.0.
- **Globs are segment-aware.** `*` matches within one name segment, and a segment that is exactly `**` matches one or more whole segments. `/*` is now only the top-level names; write `/**` for everything. `a**` and `***` are refused. A glob never matches a reserved name (`/clock`, `/parameter_events`, `/rosout`); only an exact entry with `allow_reserved: true` carries one.
- **Entries into the listener's world declare their type.** Every topic, service and action entry into the robot names `type`, and a topic glob into it lists `types: [...]`. Both halves enforce it: a publisher or an advertisement of another type is not carried. A glob into the robot with no literal segment at all (`/**`, `/*`) is refused unless its direction sets `allow_open_inbound: true`.
- **A reserved or hidden destination is refused** (`/clock`, `/rosout`, `/parameter_events`, a name with a segment starting with `_`) unless the entry sets `allow_reserved: true`. Under 1–5 it is a warning.
- **`expires` is acted on by the engine.** It takes an RFC 3339 time with a zone, on any entry, and each half ends the entry at that moment by itself. It is still never allowed together with `protected`.
- **The directories and the metrics port are stricter.** An `ipc.dir` or forensics directory owned by another user or readable by others is refused (a warning under 1–5), and so is a `/metrics` bind outside loopback on the connector half unless `metrics.allow_untrusted_bind: true` is set.

## New keys

| Key | Where | What it does |
| -- | -- | -- |
| `exclude` | direction | Topic patterns and presets (`@ros-noise`, `@debug`, or your own in `<config dir>/presets.d/`) removed from the direction's globs. Both halves apply it. `ros2 airlock learn --presets` lists what each preset holds |
| `allow_open_inbound` | direction | Permits a glob with no literal segment into the listener's world |
| `types` | topic glob | The types a glob may carry; required on a glob into the listener's world |
| `finite` | topic, service or action entry | Default `true`: on an entry into the listener's world, a message carrying a NaN or an infinity is dropped and counted (`dropped_nonfinite`). Set `false` for a type that uses NaN on purpose |
| `allow_reserved` | exact topic or service entry | Lets the entry publish a reserved or hidden name |
| `max_inflight` | service entry | Calls in flight, 1 to 1024, default 32; past it a call is answered at once with a reply marked `airlock: busy` |
| `expires` | any entry | The moment the halves end the entry (see above) |
| `include_dir` | `airlock` | A directory of fragment files merged with this config, so `ros2 airlock learn --apply` and `ros2 airlock open` can add entries without touching this file |
| `managed_by` | `airlock` | Informational: who owns a config without `include_dir`. The write tools print it when they refuse |
| `min_peer_minor` | `airlock.ipc` | The oldest protocol minor the listener accepts from the other half. `6` refuses halves older than 1.17.0 |
| `allow_untrusted_bind` | `worlds.<w>.metrics` | Permits a non-loopback `/metrics` bind on the connector half |

A 1–5 file that uses any of these keys is refused, and the error names `schema: 6`.

## Fragments in the include directory

With `include_dir: airlock.yaml.d`, both halves read every `*.yaml` in that directory, in byte-wise name order, after the base file. A fragment holds `schema: 6` and `airlock: { directions: [...] }` and nothing else; its directions merge with the base by the usual rules, so a topic stated twice in one direction is still an error across files. At most 256 fragments of 1 MiB each; a symlink or a group- or other-writable fragment is refused. Any error in any file refuses the whole load, and on SIGHUP the running config stays. Mount the config's directory into both halves, not only the file. The tools write their fragments here: `50-learn-<time>.yaml` from learn and `80-open-<name>.yaml` from `open`.

## What changed on every schema

Some checks tightened in 1.22.0 whatever the schema, because they only refuse broken configs: a key stated twice in one mapping; names that fail the ROS validators; type names outside the `package/msg|srv|action/Name` form; topic names over 255 bytes and type names over 159; a `max_hz` outside 0 to 10000, a `timeout_s` outside 0.001 to 3600, a `max_bytes` or `max_request_bytes` outside 1 byte to 1 GiB, or a float that is not finite; `tf.merged` with `namespace: "/"`; an `ipc.dir` so long that the control socket path passes 107 bytes; a group- or other-writable `ipc.dir` or forensics directory. On every schema a wildcard never matches a hidden segment (one starting with `_`) any more; `airlock check` lists the running lanes that fix ends, with exit code 3. And an inbound lane with no `max_hz` now runs at most 1000 messages per second ([Harden: the airlock pattern](harden-the-airlock-pattern.md)).

## Both halves must understand 6

An engine older than 1.22.0 refuses a schema 6 file as newer than it understands. That is deliberate: a config that depends on `exclude`, `expires` or fragments fails closed on an old half instead of silently carrying more than you meant. Upgrade both halves first, then move the file. A tool that renders configs for several robots should emit the oldest schema every engine it addresses understands. On Husarion robots Husarion Cockpit decides when to render schema 6.
