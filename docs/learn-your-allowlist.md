# Let the airlock learn your allowlist

The airlock carries nothing a config does not name, and writing that list by hand is the tedious part. Learn mode writes the first draft for you. Each half keeps a ledger of what its own world does, and `learn` turns it into a proposed allowlist with the evidence for every line. You review it and switch it on, for a few minutes first if you like. Run it where the halves' files are, on a running pair:

```bash
source /opt/ros/jazzy/setup.bash
ros2 airlock learn
```

```text
robot world, observed for 2 min (ledger live since 19:27)

  TO FLEET  (data flows out)
  ✓ /battery/battery_status        sensor_msgs/msg/BatteryState     1 pub · 2 min     unbounded (engine default)
  ✓ /hardware/e_stop               std_msgs/msg/Bool                1 pub · latched   8 B fixed · transient_local · lock suggested
  ✓ /odometry/filtered             nav_msgs/msg/Odometry            1 pub · 2 min     unbounded (engine default)
  · /camera/color/image_raw        sensor_msgs/msg/Image            1 pub · 2 min     unbounded (engine default) · heavy, lazy

  WRITES INTO ROBOT  (commented out: uncomment each one yourself)
  ! /cmd_vel          from fleet   geometry_msgs/msg/TwistStamped   1 sub · 2 min     unbounded (engine default) · deadman 200 ms · max_hz 50

  hidden: 15 plumbing names (@ros-noise and friends), 1 already configured  (--why-not <name>)
  offline check: passes (airlock: <merged>/airlock.yaml is valid (schema 6, epoch 39d0b0cd122bcb1f, 1 fragments))

  write the proposal:   ros2 airlock learn --write learned.yaml
  try it for 5 minutes: ros2 airlock learn --apply --trial 5m
```

`✓` is proposed and switched on, `·` is proposed but left for you to decide, and `!` is written into the proposal commented out. `ros2 airlock <verb>` and `ros2 run ros2_airlock airlock <verb>` are the same tool; this page uses the first.

## Where it runs and what it reads

`learn` needs no flags on a machine or in a container where a half runs: it finds the config from the running `airlock_half` (or `-c <file>`, or `AIRLOCK_CONFIG`) and reads both halves' files from the config's `ipc.dir`. It must run as the halves' user, because `ipc.dir` is private to them: uid 10001 in the default image, root where the halves run `--user 0`. In Docker that is simply:

```bash
docker exec -it <robot half container> bash -lc 'source /opt/ros/jazzy/setup.bash && ros2 airlock learn'
```

It reads each half's ledger (`ledger_<world>.json`) and status, the config and its fragments, the engine's presets and the knowledge base that classifies names, and the image's type catalog. It creates no ROS participant and subscribes to nothing. The ledger is always on: each half records the names, types and publisher and subscriber counts its world shows, from the graph sample it already takes every 5 s, so there is no session to start. A name needs to be seen in 3 samples before it is proposed (`--min-evidence`), and the ledger writes each name's first samples as they come, so a fresh pair has its first proposal after about 15 seconds. The ledger survives a half restart in the same world and starts fresh when the half comes up on another RMW, domain, profile, release or boot.

## What it proposes, and what it never switches on

Learn looks from the listener's world (the robot in the usual setup; `--home <world>` picks the other). A topic with a publisher there is a candidate to carry out. A topic that something there subscribes to while nothing there publishes it is a candidate to carry in.

Only topics that carry data OUT, at low risk, are proposed switched on. Everything that writes into a world stays commented out, whichever direction the config would put it in:

- An inbound topic writes into the robot. You uncomment it yourself, after reading the safe settings learn already filled in.
- Every service and action is commented out in both directions, because a call writes into two worlds: the request or goal lands where the server runs, and the response or feedback lands where the caller runs. Learn labels each one by its effect ("callable from fleet", "robot calls fleet"); on Humble, which cannot count servers and clients, it says the side is unknown.

The knowledge base also fills in settings from what the ledger saw:

- A topic whose publisher is transient_local gets `qos: { durability: transient_local }`, and small latched state gets `activation: always`.
- A name that looks safety-related (`e_stop`, `estop`, `safety`, `emergency`) gets a lock suggestion, written as a comment. Learn never writes `protected: true` into a proposal: a trial entry cannot carry it, and removing a protected entry later needs a restart of both halves. Add it yourself once you keep the entry.
- A velocity command into the robot gets `max_hz: 50` and a 200 ms `deadman` with a zero neutral.
- Images, point clouds and occupancy grids stay lazy and are left for review.
- Sizes come from the types, never from probing: a fixed-size type such as `std_msgs/msg/Bool` gets an exact `max_bytes`, and anything with a string or a sequence in it (every type with a `Header`) gets no cap and is flagged "unbounded", so the engine's 10 MiB default applies.

Learn also stays within `ipc.shm_budget`: candidates that would exceed it are proposed commented out, with the budget to add.

## What it hides, and how to ask about it

Plumbing never reaches the table. The engine ships three presets, which you can list with `ros2 airlock learn --presets`:

- `@ros-noise`: `/rosout`, `/parameter_events` and Humble's lifecycle `transition_event` topics, plus what learn omits but a config cannot express: every node's parameter, type-description and logger-level services, every hidden name (a segment starting with `_`), everything of anonymous CLI and launch nodes, and the halves' own nodes.
- `@debug`: the controller_manager statistics, introspection and activity topics and `diagnostics_agg`. Shown with `--include debug`.
- `@lifecycle`: the lifecycle management services. Shown, commented out, with `--include lifecycle`.

Names the config already carries, types this image cannot load, and names seen with more than one type are left out as well. Ask about any name:

```bash
ros2 airlock learn --why /hardware/e_stop     # the evidence and the rule behind a candidate
ros2 airlock learn --why-not /rosout          # why a name is left out
```

## Keep the proposal as a file

`--format yaml` prints the proposal as a schema 6 fragment, one comment per entry with its evidence and rule; `--write learned.yaml` writes it after running the engine's own offline check over your config plus the proposal. A proposal that fails that check is a bug in learn mode, so `--write` refuses it. Paste the entries you want into your config, or move the file into the config's include directory (next section). `--format json` prints the same proposal for other tools (`schema/learn.schema.json`).

## Try it for five minutes

To apply from the tool, the config must be schema 6 ([Move a config to schema 6](move-a-config-to-schema-6.md)) and name an include directory, a folder of fragment files the halves read beside the base file. These are the two lines that matter:

```yaml
schema: 6
airlock:
  include_dir: airlock.yaml.d   # relative to the config file; the tools create it
```

Mount the config's directory into both halves, not just the file, so both see the same fragments. `ros2 airlock init` writes a config and a compose file set up this way ([the toolbox](use-the-toolbox.md)). Then:

```bash
ros2 airlock learn --apply --trial 5m
```

The tool writes the switched-on entries, and only those, into `airlock.yaml.d/50-learn-<time>.yaml` with an `expires` deadline on each, prints the reload plan for both halves, asks, sends SIGHUP to both halves and waits until both run the new config. On a terminal it then counts down: `k` keeps the entries, `r` removes them now, `q` walks away. The same actions later:

```bash
ros2 airlock learn --keep      # the newest trial: its entries no longer expire (a hitless reload)
ros2 airlock learn --revert    # the newest trial: remove it now
```

Walking away is safe. The halves end an expiring entry at its deadline on their own, with no tool alive, and an entry whose deadline has passed stays off across a restart or a reboot. A trial lasts from 30 s to 24 h. Without a terminal, `--apply` needs `--yes`.

Expiring entries need a synchronised clock. Until the kernel reports the clock synchronised (chrony or systemd-timesyncd does that), an expiring entry stays off, and a deadline more than 7 days ahead of the half's clock is ignored; `ros2 airlock trace <name>` says which. So a robot that boots with a wrong clock can never hold a trial open too long.

## When `--apply` refuses

- **The config has no `include_dir`.** The tool treats it as managed by something else and never rewrites it. It prints the config's `managed_by:` value if it has one. On Husarion robots the config is rendered by Husarion Cockpit: add lanes from its Lanes page instead. Otherwise add `include_dir` as above, or use `--write` and edit the config yourself.
- **A half lacks a capability.** Trials need both halves on 1.22.0 or later (the status lists `include_dir` and `expires`); the tool names the half that lacks it.
- **The tool cannot signal the halves.** From another PID namespace (a separate container), the fragment is written, the tool prints the reload command to run (`docker kill -s HUP <container>` for each half) and exits 3.

## More ways to use it

- `--since now` marks a moment. Drive the robot, toggle the e-stop, switch camera modes, then run `learn --since <the printed time>`: names first seen after the mark are highlighted. `--since` also takes `14:02`, `-10m` or an RFC 3339 time.
- `--watch` redraws the table every 5 s and marks names that appeared since the first draw.
- `--peer-catalog <file>`: when the other half runs another ROS 2 release, pass its image's `share/ros2_airlock/type_catalog.json`, and each candidate says whether its type is the same there, converted by a rule, or needs a rule ([Bridge two ROS 2 releases](bridge-two-ros2-releases.md)).
