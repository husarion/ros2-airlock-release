# Answer everyday questions with the toolbox

The toolbox answers the questions you ask a running wall: what is crossing right now, why a topic is not arriving, what a config change would do, what happened at 14:02, and how to start a new wall. Each tool reads the files the halves already write (status, ledger, history, config) and prints text, or JSON with `--format json`. Start with `trace` when a topic is missing:

```bash
source /opt/ros/jazzy/setup.bash
ros2 airlock trace /chatter
```

```text
trace /chatter (topic)
  ok  name        carried by an exact entry of robot -> fleet (/etc/airlock/airlock.yaml)
  ok  entry       live
  ok  peer        both halves connected
  ok  source      1 publisher(s) in the robot world
  FAIL demand      lazy, nothing subscribes

/chatter is lazy and nothing subscribes to /robot/chatter in the fleet world, so nothing crosses yet; it activates on the first subscriber of /robot/chatter
```

Every tool is both `ros2 airlock <tool>` and `ros2 run ros2_airlock airlock <tool>`. Like [learn](learn-your-allowlist.md), the tools find the config from a running `airlock_half` (or take `-c <file>`), and they must run as the halves' user, because `ipc.dir` is private to it: uid 10001 in the default image, root where the halves run `--user 0`. Inside a half's container that is `docker exec -it <container> bash -lc 'source /opt/ros/jazzy/setup.bash && ros2 airlock <tool>'`. To read copies of the files from another machine, pass `--status <world>=<file>` and `--ledger <world>=<file>`. A tool that needs something an older half does not write says which half lacks it.

## `trace`: why isn't my topic arriving?

`trace <name>` follows one topic or service across the wall hop by hop: the name and the entry that carries it, the peer session, the publishers in the source world, demand in the destination world, the lane on both halves, the destination's subscribers and any QoS mismatch they counted. It stops at the first hop that fails and ends with one sentence: what is wrong and what to change. The exit code is 0 when the name crosses and 1 otherwise. Common endings:

```text
/odometry/filtered (nav_msgs/msg/Odometry) is seen in the robot world and no entry or glob carries it; add it: {topic: /odometry/filtered, type: nav_msgs/msg/Odometry} in robot -> fleet topics
```

```text
/odom is in neither world's ledger; closest: "/odometry/filtered" (nav_msgs/msg/Odometry); check the spelling, or start its publisher
```

```text
/chatter crosses, but it arrives in the fleet world as /robot/chatter; subscribe to (or call) /robot/chatter there
```

Other endings name an `exclude:` item that removes the name from a glob, a hidden or reserved name no glob matches, an entry that expired or waits for a synchronised clock, a refused lane and the rule it needs, a cap that is dropping and the key to raise. `trace` and `airlock doctor` print the same sentences, from the cause table the engine ships (`share/ros2_airlock/causes.yaml`). [Debug a silent topic](debug-a-silent-topic.md) explains each state in depth.

## Did you mean

When an exact entry has matched nothing for a while and the ledger holds a similar name, the tools say so. `ros2 airlock check --live -c airlock.yaml` lists these hints for every entry, `airlock doctor` prints them as information (they never change its verdict), and `trace` prints them on a name that is in neither world:

```text
  did you mean: /odometry/filtered (nav_msgs/msg/Odometry)
```

## `top`: what is crossing right now?

```bash
ros2 airlock top
```

```text
LANE                  DIR             CLASS   STATE           RATE         BW     FRAME   AGE P50   AGE P99      GAP  DROPS
/robot/chatter        robot->fleet    normal  flowing         10.0    120 B/s      12 B     0.4ms     0.9ms    101ms  -

pair  robot: healthy, cpu 0.5%, rss 36.9 MiB, epoch 848ef817f7665165 | fleet: healthy, cpu 0.3%, rss 26.6 MiB, epoch 848ef817f7665165
class fleet: control 0 lanes p99 - gap - capped 0, normal 1 lanes p99 0.9ms gap 101ms capped 0, bulk 0 lanes p99 - gap - capped 0
```

One row per lane: direction, class, state, rate and bandwidth from the counter deltas, the largest frame seen, the delivery age p50 and p99 (how old a sample is when the destination world sees it), the largest gap between two samples in the last 5 s, and drops by cause (`cap` is the bulk byte cap, see [Keep camera traffic away from tf](bounded-bulk-lanes.md)). The footer shows each half's health, CPU and memory, a line per half for each class, and, for a half whose config has an `api:` block, `api: N conn, samples S/s, calls C/min, refused R` (the host request socket's connections, request rates and refusals; see [Read and call the robot from the host](read-and-call-from-the-host.md)). Against a half older than 1.25.0 the age columns show its forwarding delay instead, marked `*`. It refreshes at the status cadence (5 s) until Ctrl-C. `--sort rate|bw|frame|drops|p50|p99|gap|class|state|lane` picks the order, `--class control|normal|bulk` shows one class, `--once` prints one snapshot and exits, and `--format json` prints one snapshot per line.

## `api`: what does this wall carry right now?

```bash
ros2 airlock api
```

```text
DIR           KIND     EXPOSED             SOURCE         TYPE                       CLASS    STATE       CHANNEL   RATE      CAPS                      CARRIED BY                   EVIDENCE
robot->fleet  topic    /robot/chatter      /chatter       std_msgs/msg/String        normal   flowing     -         10 Hz     <= 4.0 KiB                exact (airlock.yaml)         both
robot->fleet  service  /robot/hw/fan       /hw/fan        std_srvs/srv/Trigger       control  advertised  released  -         req 1.0 KiB, resp 4.0 KiB glob /hw/** (airlock.yaml)   both

complete | robot->fleet: 2 items, 1 glob
```

One row per topic, service and action, sorted by direction and name: the name on both sides, the type, the lane's class, the lane state, the service channel (`released` until the first call, then `armed`), the rate, the caps and the entry that carries it, an exact one or a glob with its file. With a glob in the file the config no longer lists what is bridged, so this is the page to read. Since 1.26.0 the `host read` column marks a topic the host sampled through the half's socket within the last status period, and every `api.local` name of a half is a row of its own carried by `local`, with that half's evidence alone (`robot->host`): the host can reach it, the wall never carries it. It is built only from the halves' status files and the running config: it starts no ROS participant and opens no socket. The listener's status is the authority; a row that only the connector's status vouches for reads `unverified`, its evidence column carries a `*` and its names are printed in quotes.

`--format json` prints the document of `schema/api.schema.json` (schema 1), the one the Cockpit's live API page relays. `--direction robot->fleet` and `--kind topic|service|action` filter, `--all` also lists config entries that carry nothing right now (`unmatched`), and `--single-sample` reads once and prints no rates (by default it waits for the next status write, up to three intervals, and takes the rate from the counters). The exit code is 0 when the picture is complete, 3 when it is not: one half has written no status for 15 s or is missing (the other half's rows are printed with `evidence: robot_only` or `fleet_only`), names were left out because a glob hit `max_lanes`, or rows were dropped as malformed. It is 2 when nothing can be read, when the file on disk is no longer the one the halves run (edit and reload first), or when a half is older than 1.24.0 and writes no lane carriers. On 2 and 3 the JSON is still printed, with `complete: false` and the reasons in `problems[]`.

## `check --dry-run`: what would this change do?

Before you reload a changed config, ask the halves what they would do with it:

```bash
ros2 airlock check --dry-run next.yaml -c airlock.yaml
```

```text
dry run of next.yaml:
  [robot] recreated source robot->fleet /chatter -> /robot/chatter (max_hz) [decided at activation: type_hash, shm_budget, peer_typesupport]
  [robot] entry changed topic /chatter (max_hz)
  [robot] kept      2 lane(s) unchanged
  [robot] kept      1 service lane(s) unchanged (calls in flight survive)
  [fleet] recreated destination robot->fleet /robot/chatter (max_hz) [decided at activation: type_hash, shm_budget, peer_typesupport]
  [fleet] entry changed topic /chatter (max_hz)
  [fleet] kept      2 lane(s) unchanged
  [fleet] kept      1 service lane(s) unchanged (calls in flight survive)
offline check: passes (airlock: <merged>/next.yaml is valid (schema 9, epoch bf3eee15107b0fa4, 1 fragments))
  robot->fleet /chatter: matches 1 name(s) in the ledger now (/chatter)
```

Each half runs its own reload planner (`airlock_half --plan`) against the lanes it runs now, so the plan is the one the reload will carry out: lanes added, removed, re-created with the setting that changed, and kept; service lanes the same way, each by its own settings (since 1.24.0 an unchanged service keeps its calls in flight across a reload, and an exact entry that takes over a name a wildcard carries, with the same type and caps, keeps it too); the entries whose outcome is only known at activation (a type mismatch, the shared-memory budget, a type the other half cannot load); a protected lane the change would drop, which the reload refuses. `-c` names the running config (the default is the one a running half uses). `learn --apply` and `open` show the same plan before they act.

## `open` and `close`: a lane for the next ten minutes

```bash
ros2 airlock open /camera/color/image_raw --for 10m
ros2 airlock close /camera/color/image_raw
```

`open` writes a one-entry fragment, `80-open-<name>.yaml`, into the config's include directory with an `expires` deadline, checks it, shows the plan and reloads both halves. The halves close the lane at the deadline by themselves, with no tool alive, and it stays closed across restarts. `close` removes the fragment now; `close --expired` removes every fragment whose entries have all ended. The direction comes from the ledger (published in the listener's world, usually the robot, means out; only subscribed there means in) unless `--to <world>` or `--from <world>` says it; `--type` sets the type when no ledger has it. A lane lasts from 30 s to 24 h.

An `open` that writes into the listener's world, meaning a topic from the fleet or any service or action, is refused unless you add `--i-know`:

```text
airlock open: refused: topic /cmd_vel (fleet -> robot) is risk high: it writes into the robot world; pass --i-know to open it anyway
```

`open` and `close` need what `learn --apply` needs: a config with `include_dir`, both halves on 1.24.0 or later, and a synchronised clock on the robot ([Let the airlock learn your allowlist](learn-your-allowlist.md) explains all three). A config without `include_dir` is treated as managed by something else, and both refuse and name its `managed_by:`.

## `history`: what happened at 14:02?

```bash
ros2 airlock history --since 14:00
```

```text
2026-09-27 19:22:23 [robot] peer_connected lane="" count=1 peer_world="fleet" proto_minor=6 epoch_matched=True
2026-09-27 19:23:02 [robot] lane_state lane="/robot/chatter" count=1 role="tx" from="advertised" to="flowing"
2026-09-27 19:23:16 [robot] trial_added entry_kind="topic" direction="robot->fleet" name="/odometry/filtered" deadline="2026-09-27T19:25:15Z"
2026-09-27 19:23:16 [robot] reload_applied epoch="87730946423c4af5"
2026-09-27 19:23:25 [robot] trial_confirmed entry_kind="topic" direction="robot->fleet" name="/odometry/filtered" deadline="2026-09-27T19:25:15Z"
```

Both halves record what changed: reloads applied and rejected, trials and temporary lanes, lane state changes, refusals, peer connects and disconnects, heal attempts and cap episodes. `history` merges both halves into one timeline. `--lane <name>` keeps one lane, `--since`/`--until` take `14:00`, `-10m` or an RFC 3339 time, and `--around <time>` shows five minutes either side and names the black-box recording that covers the moment, if recording was on.

The history lives next to the status files in `ipc.dir`, two files of at most 5000 events and 2 MiB each. When `ipc.dir` is on a tmpfs such as `/run`, the history is kept in RAM: it survives a half restart and not a reboot, and it never writes to an SD card. Events the other half can cause are merged per lane per kind per minute, and the newest 2000 of the halves' own events (reloads, trials, heal attempts) are always kept, so a misbehaving peer cannot flush them.

## `init`: start a new wall

```bash
ros2 airlock init --dir mywall --robot-rmw rmw_fastrtps_cpp --robot-domain 21 \
  --fleet-rmw rmw_cyclonedds_cpp --fleet-domain 22 --topics /chatter:std_msgs/msg/String
ros2 airlock init --prove --dir mywall
```

```text
wrote mywall/compose.yaml, mywall/airlock.yaml and mywall/airlock.yaml.d/
airlock check: passes (airlock: <merged>/airlock.yaml is valid (schema 9, epoch dbbb64d7889d092c))
PROVED: a message published on /airlock_prove in the robot world (rmw_fastrtps_cpp, domain 21) arrived on /robot/airlock_prove in the fleet world (rmw_cyclonedds_cpp, domain 22) after 2.7 s
```

Without flags `init` asks three questions: the robot world's RMW and domain, the fleet world's, and which topics to carry out. It writes a `compose.yaml` with both halves in their worlds' environments (and a zenoh router when either world uses rmw_zenoh), an `airlock.yaml` at schema 9 with `include_dir: airlock.yaml.d` and a 64 KiB frame cap (`max_bytes`) on every topic that passes `airlock check`, and the empty include directory. The topic list can also come from `--from-topic-list <file>` (the output of `ros2 topic list -t`) or `--from-ledger <file>` (a running half's ledger). `--prove` runs both halves of the written config itself, on a private IPC directory, with a talker in the robot world and a listener in the fleet world, and prints whether a message crossed. Run it where the airlock is installed (the deb, or inside the image). Then `docker compose up -d` in that directory starts the wall, and `ros2 airlock learn` fills the allowlist once the robot runs.

## `migrate`: bring an older config to schema 9

```bash
ros2 airlock migrate airlock.yaml
```

An engine of 1.26.0 or later reads schema 9 only and refuses an older file, naming this command. `migrate` reads any schema from 1 to 8 and writes `airlock.schema9.yaml` beside your file (`--in-place` keeps a `.bak`), prints one line per change and the shared memory the new file reserves, and runs `check` on the result. A schema 8 file only gets its version line changed. From an older file it also writes `class: bulk` and the 4 MiB/s cap on every camera-sized entry, so you can see which lanes are capped. [Move a config to schema 9](move-a-config-to-schema-9.md) lists what it writes and what it leaves to you.
