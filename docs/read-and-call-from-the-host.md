# Read and call the robot from the host

A script on the robot's host often needs one fact from the driver's world, such as whether the e-stop is latched after a driver restart. Reading it with `ros2 topic echo --once` starts a ROS participant for each read, and a participant killed in the middle of a message can leave the driver's shared-memory world closed to newcomers. From engine 1.26.0 (config schema 9) the half that already lives in that world answers instead, on a local socket the config names. It reads one sample of a lane it carries or calls a service it serves, and it also returns the graph it sees.

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 9
airlock:
  ipc: { dir: /run/airlock, listener: robot }
  worlds:
    robot:
      status_dir: /run/airlock-robot/status
      forensics_dir: /var/lib/airlock-robot
      api:
        socket: /run/airlock-robot/api/api.sock
        calls:
          - { service: /hardware/e_stop_trigger, request: {} }
          - /hardware/e_stop_reset
        retain: [/hardware/e_stop, /autonomous/cmd_vel]
        local:
          services:
            - service: /hardware/e_stop_reset
              type: std_srvs/srv/Trigger
              timeout_s: 2.0
              max_request_bytes: 1024
              max_response_bytes: 4096
              purpose: "the watchdog resets the e-stop after a driver restart"
    fleet: {}
  directions:
    - from: robot
      to: fleet
      namespace: /my_robot
      topics:
        - topic: /hardware/e_stop
          type: std_msgs/msg/Bool
          class: control
          activation: always
          max_bytes: 64
      services:
        - service: /hardware/e_stop_trigger
          type: std_srvs/srv/Trigger
          timeout_s: 2.0
          max_request_bytes: 1024
          max_response_bytes: 4096
    - from: fleet
      to: robot
      namespace: /autonomous
      topics:
        - topic: /cmd_vel
          type: geometry_msgs/msg/Twist
          class: control
          activation: always
          max_bytes: 128
```

```bash
# on the robot's host, as root: the newest e-stop message with its evidence
ros2 airlock sample /hardware/e_stop -c /etc/airlock/airlock.yaml
# just the value, the way `ros2 topic echo --field` prints it
ros2 airlock sample /hardware/e_stop -c /etc/airlock/airlock.yaml --field data
# call a service the robot half serves (its fixed request is sent)
ros2 airlock call /hardware/e_stop_trigger -c /etc/airlock/airlock.yaml
# what the fleet last sent the robot, read in the robot world
ros2 airlock sample /autonomous/cmd_vel -c /etc/airlock/airlock.yaml --field linear.x
# the graph the half sees
ros2 airlock graph -c /etc/airlock/airlock.yaml
```

The sample is the lane's newest message, read by the half inside its own world, with `publishers`, `latched` (the half saw a message older than its reader, which is how a latched topic looks) and `age_s` beside it. `latched` describes the message you got, not the topic: a latched topic whose value changed after the reader started (an e-stop reset, say) answers `latched: false`, because that copy arrived live. To tell an on-change latched topic, where any age is current, from a periodic topic that went silent, read `durability` instead: it is `transient_local` when every publisher of the topic is. The lane itself does not change: a sample is not demand, it starts no flow and moves no counter of the wall. `/hardware/e_stop` is listed under `retain` (from engine 1.26.1), so the half keeps a reader on it all the time and answers from the newest message that reader took, marked `retained: true`, with `age_s` counted from when it arrived; the next section says why a polled latched topic wants that.

## What the socket serves, and to whom

The half creates the socket at `api.socket` when it starts and removes it when it stops. The directory must exist, belong to the half's uid or to root, and be writable by nobody else; the half never creates it, because who can reach the directory is the deployer's statement of who may ask. Put it in a directory mounted into this half's container only: `check` warns when the socket sits under `ipc.dir` on the listener half, since the other half shares that directory.

Who connects decides what they get:

- root (uid 0) and the uids listed in `allow_uids` get the whole block: samples of every lane the half carries, calls of the names in `calls`, and the `api.local` names;
- the half's own uid gets what the wall already gives the other world: samples of carried lanes and calls of the wall's own services named in `calls`, never an `api.local` name (`trust_own_uid: true` gives it the whole block, and `check` warns about that on the listener);
- every other uid is disconnected before a byte of its request is read.

A host call goes through a client the half owns, never through the wall's lane, so it needs no fleet half and arms nothing. Every call is written to `api_<world>.jsonl` in the half's forensics directory with the caller's uid, the request id and the outcome. A request id is executed at most once per half start: a retry that names it gets the first call's outcome back, marked `duplicate`, and the service does not run again. A reply that the half had to make up (the server did not answer in time, no server matched, the half was stopping) says so in `outcome: synthesized` with the reason and whether the request may have executed.

`api.local.services` (and `api.local.topics`) names what the host may reach and the wall does not carry: the fleet never sees these names, the entry needs a `type` and a `purpose`, and doctor lists each one as `api.local_call_enabled` so nobody is surprised that the host can call it. Above, the e-stop reset is such a name: the watchdog may release the latch, the fleet may not.

### Lanes the fleet sends into the robot (engine 1.27.0)

A lane the peer delivers into the half's world can be sampled too, by the name it has in that world: above, the fleet's `/cmd_vel` arrives in the robot world as `/autonomous/cmd_vel`, and that is the name to ask for and to list in `samples` or `retain`. The half reads it through a reader of its own on that name, so it sees what the robot's nodes see, its own publication of the lane included. The fleet's name `/cmd_vel` means nothing in the robot world and answers `not_carried`. The sample still is not demand: a lazy lane nobody in the robot world asks for stays idle, and a sample of it answers `timeout`. A sample never pauses a flowing lane for its real subscribers either; when the last of them leaves, the lane deactivates 1 to 2 s later (engine 1.28.0; 0 to 1 s before it), and up to 2 s later than that if a sample's reader was released just before and is still matched. The name is known from the config from the moment the half starts: before the fleet publishes it at all, a sample answers `no_publisher` (or `timeout`), as for any lane without a publisher, never `not_carried`, and a name listed in `retain` reads `waiting` under `api.retained` and gets its reader once the lane's publisher appears in the robot world (engine 1.27.1; 1.27.0 answered `not_carried` until the fleet first advertised the lane).

### NaN and infinite values (engine 1.27.1)

JSON has no NaN or infinity, so a sample writes a float that holds one as the string `"NaN"`, `"Infinity"` or `"-Infinity"` and sets `nonfinite: true`. A string field could hold the same text, so every sample reply also lists the path of each such float in `nonfinite_paths` (`["temperature", "cell_voltage[3]"]`, at most 64, then `nonfinite_paths_truncated: true`), and a tool that has no `.msg` definitions can tell the two apart. A request with `"nonfinite": "null"` (`airlock_api sample --nonfinite null`) writes those floats as JSON null instead, as rosbridge does; `nonfinite_paths` still says which nulls they are. A `sensor_msgs/BatteryState` whose driver does not know the temperature is the common case:

```bash
airlock_api sample /battery/battery_status --nonfinite null --field temperature   # prints null
```

### A call with a fixed request (engine 1.27.0)

A `calls` item may be `{service: <name>, request: <the request's fields>}`, and an `api.local.services` entry takes the same `request:`. A host call of that name may leave the request out (the fixed one is sent) or send one equal to it once both are filled with the type's defaults; any other request is refused `not_allowed`, nothing reaches the service, and the audit file records the refusal. Above, the e-stop trigger can be pulled from the host but never with arguments someone else chose. The half checks the fixed request against the service's type when it starts and on every reload, and so does `check`; a name whose type the config leaves to discovery is checked at its first call. A name written bare keeps taking any request.

### A budget per caller (engine 1.27.0)

The rate limits under `api.rate` apply to each uid on its own: a tool running as one uid that spends its budget is refused `rate`, while the watchdog running as root still gets its full budget. A uid that has held no connection for 60 seconds starts with a full budget again. The status lists every uid with its tokens and refusals under `api.uids`.

## Watching a retained lane (engine 1.28.0)

A tool that wants to know when the e-stop changes does not have to poll. It opens a watch, and the half writes one line each time the lane's standing reader takes a new message, until the tool closes the connection or the half ends the watch:

```bash
# one line per update: the e-stop's value, as long as it runs
airlock_api watch /hardware/e_stop --socket /run/airlock-robot/api/api.sock --field data
# every line the half writes (updates, keepalives, the closing line), as JSON
airlock_api watch /hardware/e_stop /battery/battery_status --socket /run/airlock-robot/api/api.sock
```

Only a name listed in `api.retain` can be watched: the watch reads the reader the half already keeps for it, so it adds nothing to the robot's world. Any other name is refused `not_watchable`, and the refusal names the `retain` key to add it to. `api.retain` also takes an `api.local.topics` name from 1.28.0 (typed, exact), so a half can let the host watch a topic of its own world that the wall does not carry, and an action's `_action/feedback` and `_action/status` topics can be listed like any carried topic: a dashboard can then follow the progress of goals another client sent. The scope rules are those of a sample: the half's own uid may watch carried lanes, root and `allow_uids` the `api.local` names too.

Each update line carries the message exactly as a sample of the retained lane renders it (`sample`, with `retained: true`, `age_s`, `publishers`, `latched`, `nonfinite_paths`; `nonfinite: "null"` on the request, or `--nonfinite null`, works as for a sample), plus `seq`, the watch's update number from 1, and `coalesced`. Each watched name sends at most `api.watch.max_hz` updates a second (default 10), so a watch of several names may send names x max_hz lines a second and a tool sizes its reading to that: a message taken while the last update is less than 1/max_hz old waits, a newer one replaces it, and the newest leaves when the interval ends, so a value followed by silence still arrives at most one interval late. `coalesced` counts the messages the reader took that were not sent. It is not a count of what the publisher sent, because the reader keeps only the newest message (eight on Humble), and anything published between two takes is never seen. An update leaves within 1/max_hz of the take that brought it: the reader wakes the half as soon as a message arrives.

The first lines of a watch are the current values of the names that hold one (seq 1, 2, ...), so a tool never waits for a latched topic to be published again. When a name holds nothing yet, a keepalive comes right after them. When a watch has sent nothing for `api.watch.keepalive_s` (default 1 s), the half sends one keepalive line that lists each name's state (`waiting` for no reader yet, `none` for a reader without a message, `held`, `blind`), its publishers and the age of the value it holds. Every line carries `keepalive_s`, and a tool that sees no line for three times that treats the half as gone and falls back. A blind half sends `blind` keepalives and no updates; when a participant rebuild has re-created the reader, its next message is an update with the next `seq`, and a latched value comes back marked `latched: true`.

The half ends a watch with a closing line (`kind: closed`) when it stops or its `api:` block goes (`reason: teardown`), when a reload takes a watched name out of `api.retain` (`reload`), or when the tool sends more than `api.max_request_bytes` without a newline, or more than 64 lines (or `api.max_request_bytes` of them) on the watch (`size`); lines a tool sends on a watch are otherwise read and ignored. A tool that stops reading is cut off: when bytes the half wrote have sat unread for three keepalive intervals, the half closes the connection without a closing line, counts it under `api.watch.stalled`, writes it to the audit file `api_<world>.jsonl` with the tool's uid and pid, and doctor warns `api.watch_stalled`. The half asks the kernel how much of what it wrote the tool has not read yet, so a stuck tool is cut off after three keepalive intervals whether its watch is busy or quiet, and a tool that exits or closes its end is seen at once.

A watch connection is no longer one of `api.max_connections` once its first line is read: it takes one of `api.max_watchers` (default 4, one of them kept for root) instead, and a watch beyond that is refused `watchers`, so watches never take the slots calls and samples need. Opening a watch costs one token of the uid's `samples` budget, once. Across all watches, the half renders at most `api.watch.max_updates_per_s` updates a second (default 200), because the same thread answers the e-stop calls; past that, updates coalesce.

`airlock_api watch` exits 0 when the half ended the watch with its closing line, 1 when the half was lost (no line for three keepalive intervals, or end of file without a closing line), 2 when there is no socket to ask, and 3 when the half refused the watch. `--format json` (the default) prints every line exactly as it came, `value` the message of each update (prefixed by its topic when several names are watched), `yaml` each update's sample as a document, and `--field` one field of each update's message.

## Driving an action from the host (engine 1.28.0)

Host automation can send one goal to an action without a ROS participant of its own, follow it, and cancel it. Motion is stopped by the action server and the e-stop, never by a socket closing: closing the connection only asks the server to cancel, and a server may finish the goal anyway.

The action has to be listed in `api.actions` of the half whose world serves it, with the longest a goal may run:

```yaml
worlds:
  fleet:
    api:
      socket: /run/husarion-airlock-fleet/api/api.sock
      actions:
        - action: /navigate_to_pose
          type: nav2_msgs/action/NavigateToPose
          purpose: "the Cockpit map's go here"
          max_duration_s: 600
          goal_constraints: { pose.header.frame_id: map }
```

```bash
# one goal: its accepted line, every status change, the feedback, the result
airlock_api action /navigate_to_pose --socket /run/husarion-airlock-fleet/api/api.sock \
  --goal '{"pose": {"header": {"frame_id": "map"}, "pose": {"position": {"x": 2.0, "y": 1.0}}}}'
# cancel it from another shell, by the goal id the accepted line printed
airlock_api cancel /navigate_to_pose 3f2a... --socket /run/husarion-airlock-fleet/api/api.sock
```

An entry names an action whose server lives in this half's world: an exact action entry of a direction from this world (write only `action` and the bounds; the type is the entry's), or a local action the wall does not carry at all, with its `type` and a `purpose` doctor and the audit print. Nav2 runs in the fleet world, so the fleet half gets an `api:` block of its own for it; its socket lives in a directory only that half sees, and nothing in that block reads the robot world (`check` refuses an action the wall delivers into a world, since its server is in the other one). The engine has no type-erased action client on Humble, so a half on Humble refuses an `api.actions` block of its own world by name at start. `check` on a Humble image refuses it only for the half it is told about: `airlock_half --check --world fleet -c airlock.yaml` (or `ros2 airlock check airlock.yaml --world fleet`) refuses the fleet world's block, and reports another world's block, or every block when no `--world` is given, as the note `api.actions_humble`, so one file shared by a Humble robot half and a Jazzy fleet half checks on both.

Only root may drive an action unless its entry's own `allow_uids` lists more uids; `api.allow_uids` and `trust_own_uid` never extend to actions. `max_duration_s` is required: when it runs out the half cancels the goal and ends the stream with `status: timeout`. `goal:` fixes the goal (a request may omit it, or send it field for field the same; anything else is refused `not_allowed`), and `goal_constraints:` pins single fields while the rest stays free: after the half has built the goal, each listed path must hold its value, else the goal is refused `not_allowed` naming the path. One entry takes one or the other.

The stream is one JSON line per event ([schema](../../schema/api_action.schema.json)): `accepted` with the goal id (32 hex digits) or `accepted: false` with a reason (`rejected`, `no_server` when no server matched within `accept_timeout_s`, nothing sent, `accept_timeout` when the server did not answer in time, the goal then cancelled by its id); then every status change the half sees (`executing`, `canceling`, and the end), never merged; `feedback` at most `api.actions_feedback_max_hz` times a second (default 5), the newest one when several came in between (`coalesced` counts the ones not sent), within the half-wide `api.watch.max_updates_per_s` the watches use too; and one `result` line with `status` `succeeded`, `aborted`, `canceled`, `timeout` or `lost`, after which the half closes the connection. When the half stops with a goal running, it cancels it and ends the stream with `kind: closed`, `reason: teardown`; it does the same with `reason: reload` when a reload takes the action out of `api.actions`, and with `reason: size` when the tool writes more than `api.max_request_bytes` (or 64 lines) on the stream. A goal stream whose tool leaves its lines unread for 10 s is closed, which cancels the goal.

One host goal per action runs at a time: another `action_goal` on it is refused `goal_busy`, naming the active goal's id and owner uid. Never fall back to another client then (`ros2 action send_goal`): that would be the second goal the half refuses. The slot frees when the goal ends, when its cancel is answered `terminated` or `unknown_goal` (for a goal its server confirmed), or when the goal is `lost`: no server matched, or its id left the action's `_action/status`, for `lost_after_s` (default 10), or its cancel was taken and no end followed within `timeout_s + lost_after_s`, when the server may still be ending the old goal while a new one starts. "No server matched" is as DDS reports it: a server that exits cleanly is gone at once, a killed one only after its DDS participant lease (20 s on Fast DDS by default), so a goal on a killed server ends `lost` after that lease plus `lost_after_s`, or `timeout` first when its `max_duration_s` is shorter. A goal the server did not confirm in `accept_timeout_s` keeps the slot until its id has been absent from the status for `lost_after_s`. A request carries an optional `request_id`, as a call does: a retry with the same `request_id` is answered `duplicate` with the first goal's id and outcome, never a second goal. The `request_id` ledger is per action, not per uid: a `request_id` another uid already used reads `duplicate` with that uid's goal.

When a tool's connection closes before the result, the half sends the cancel at once (or as soon as the goal is accepted) and keeps tracking the goal to its end. A half killed with a goal running cancels it at its next start, from the `goal_sent` line it wrote to the audit file `api_<world>.jsonl` before it sent the goal; a half that cannot write that file sends no goal at all (`internal`, or `accepted: false` with `send_failed`). For up to `lost_after_s` after a start, a new goal on an action with such a goal is refused `goal_busy` (`recovering: true`); the cancel itself is sent whenever the action's server is there, until it is answered or the goal's `max_duration_s` has passed. A stopping half cancels its goals without waiting for the answer and the next start cancels them once more, which the server answers `terminated` or `unknown_goal` when they had ended, both recorded `canceled_at_start`. The audit file keeps two segments of 1 MiB: a goal sent before more than 1 MiB of later audit lines is no longer seen by the next start. `action_cancel` cancels a goal by its id; only the uid that sent it, or root, may (another uid gets `not_allowed` for any id, so it cannot probe which goals run). A cancel sent is not a cancel confirmed: the reply carries the server's `return_code` (`none` taken, `rejected`, `unknown_goal`, `terminated`, or `unanswered` within `timeout_s`), and a rejected or unanswered cancel shows as doctor `api.action_cancel_failed`. A lost goal shows as `api.action_lost`.

A goal stream holds one of `api.max_goals` (default 2, one kept for root), never one of `max_connections`, `max_watchers` or `max_inflight`; a goal and a cancel each cost one token of the uid's `calls` budget. The half does not watch the e-stop and does not cancel host goals when it latches: the driver stops motion, and the tool that sent a goal cancels it on its own e-stop path.

`airlock_api action` exits 0 when the goal succeeded, 1 when it ended otherwise (aborted, canceled, timeout, lost, not accepted) or the half was lost, 2 when there is no socket to ask, and 3 when the half refused the goal. `--format json` (the default) prints every line as it came, `value` and `--field` the message of each feedback and the result, `yaml` each feedback and result line as a document. `airlock_api cancel` exits 0 when the server took the cancel or the goal had ended, 1 when it was rejected, unknown or unanswered, 3 on `goal_not_found` or `not_allowed`.

## Every key of the `api:` block

Each world may carry its own `api:` block under `worlds.<world>`; the half of that world serves it. `airlock_half --constants` prints the defaults under `api` ([constants contract](constants-contract.md)).

| Key | Default | Meaning |
| -- | -- | -- |
| `socket` | none, required | Absolute path of the socket, at most 107 bytes with a file name of at most 72. Its directory must exist, belong to the half's uid or root and be writable by nobody else. |
| `allow_uids` | `[]` | Up to 16 uids that get the whole block, as root does. |
| `trust_own_uid` | `false` | Gives the half's own uid the whole block too; `check` warns about it on the listener. |
| `samples` | `carried` | What a sample may read. `carried` admits every lane the half holds: each topic a direction from this world carries (an exact entry or a name one of its globs admits) and, from 1.27.0, each destination name a direction into this world delivers. A list of exact names admits only those, and each must be such a lane. |
| `calls` | `[]` (nothing callable) | Exact service names the host may call: a service entry of a direction from this world (its server lives here) or an `api.local.services` name. An item may be `{service: <name>, request: <fields>}` to fix the request (1.27.0). Actions and globs are not callable. |
| `retain` | `[]` | Up to 16 exact topic names, each with a standing reader for the half's whole run, so a sample costs the world nothing (1.26.1). Each must be sampleable and its entry's `max_bytes` within `max_raw_sample_bytes`. From 1.28.0 also an `api.local.topics` name, and every retained name is watchable. |
| `max_watchers` | 4 (1 to 16) | Watch streams at once (1.28.0); one is kept for root. |
| `max_watch_names` | 8 (1 to 32) | Names one watch may carry. |
| `watch.max_hz` | 10 (1 to 50) | Updates a second per watched name at most. |
| `watch.keepalive_s` | 1 (0.2 to 10) | A keepalive line after this long without a line; a line the tool leaves untaken three times this long closes its watch. |
| `watch.max_updates_per_s` | 200 (10 to 2000) | Updates the half renders a second across all watches; past it, updates coalesce. |
| `max_goals` | 2 (1 to 8) | Goal streams at once (1.28.0); one is kept for root. |
| `actions_feedback_max_hz` | 5 (1 to 20) | Feedback lines a second per goal stream at most. |
| `actions` | `[]` | Up to 16 actions the host may drive: `action`, the required `max_duration_s` (0.1 to 86400), `accept_timeout_s` (5; 0.1 to 60), `timeout_s` (2; 0.1 to 60, a cancel's response and the result after the end), `lost_after_s` (10; 1 to 600), `allow_uids`, a fixed `goal` or `goal_constraints`, and for a local action `type` and `purpose`. Not on Humble. |
| `local.services` | `[]` | Services the host may call that the wall does not carry in this world. Each takes the service entry's keys `service`, `type` (required), `timeout_s`, `max_request_bytes`, `max_response_bytes`, `max_inflight`, `finite`, an optional fixed `request`, and a required `purpose` of up to 255 printable bytes, which doctor and the audit print. |
| `local.topics` | `[]` | Topics the host may sample that the wall does not carry in this world: `topic`, `type` (required, since no lane discovers it), an optional `max_bytes` and the required `purpose`. Only the uids that get the whole block reach them. |
| `max_connections` | 4 (1 to 16) | Connections at once; one slot is kept for root. |
| `max_inflight` | 2 (1 to 8) | Calls in flight across the surface; the next one is refused `inflight`. |
| `max_request_bytes` | 65536 (1024 to 1 MiB) | The longest request line; a longer one is refused `size`. |
| `max_sample_bytes` | 1048576 (4096 to 16 MiB) | The rendered reply of a sample or a graph; past it the reply is cut at an element boundary and marked `truncated`. A request's `max_bytes` may lower it. |
| `max_raw_sample_bytes` | 4194304 (4096 to 64 MiB) | The reader's buffer for one serialized message: a lane whose `max_bytes` is larger answers `size`. |
| `rate.samples` | `{per_s: 5, burst: 10}` | Token bucket for samples, per admitted uid (1.27.0). `per_s` 1 to 1000, `burst` at least `per_s` and at most 1000. |
| `rate.calls` | `{per_s: 2, burst: 4}` | The same for calls. |
| `rate.graph` | `{per_s: 10, burst: 10}` | The same for `graph`. |

A name under `api.local` must be one the wall does not touch in this world, neither carried out of it nor delivered into it, so a local entry can never become a way across the wall that the directions do not name.

## The verbs and the native client

`ros2 airlock sample <topic>`, `ros2 airlock call <service>`, `ros2 airlock graph` and (1.28.0) `ros2 airlock watch <topic>...`, `ros2 airlock action <action>` and `ros2 airlock cancel <action> <goal_id>` take the socket from the config (`-c`, or `AIRLOCK_CONFIG`, or the `-c` of a half running beside them) or from `--socket`, never from a status file: another process of the half's uid can write a status file and point a reader somewhere else. The name is the one in the half's own world: a source name, or the destination name of a lane delivered into it.

From engine 1.27.0 the three verbs run `airlock_api`, a small native program that speaks to the socket without starting Python or ROS (the image has it at `/opt/ros2_airlock/lib/ros2_airlock/airlock_api`). It takes the same arguments with the verb first, `airlock_api sample /hardware/e_stop --socket /run/airlock-robot/api/api.sock`, prints the same output and exits with the same codes. Every GitHub Release also carries it as one statically linked file per architecture, `airlock_api-<version>-linux-amd64` and `airlock_api-<version>-linux-arm64`, each with a `.sha256` beside it. It needs no ROS install and no shared library, so a host installs it by copying that file:

```bash
v=1.27.1; a=arm64
curl -LO https://github.com/husarion/ros2-airlock-release/releases/download/v$v/airlock_api-$v-linux-$a
curl -LO https://github.com/husarion/ros2-airlock-release/releases/download/v$v/airlock_api-$v-linux-$a.sha256
sha256sum -c airlock_api-$v-linux-$a.sha256 && install -m 0755 airlock_api-$v-linux-$a /usr/local/bin/airlock_api
airlock_api --version
```

| Flag | Meaning |
| -- | -- |
| `--world <w>` | the half to ask, when both worlds have an `api:` block |
| `--wait <s>` | `sample`: seconds to wait for a message (default 2, at most 10) |
| `--fresh` | `sample`: read a lane listed in `api.retain` through a new reader of its own instead of the kept copy (1.26.1) |
| `--nonfinite string\|null` | `sample`, `watch`: write a NaN or infinite float as the string `"NaN"`, `"Infinity"`, `"-Infinity"` (default) or as null (1.27.1) |
| `--format json\|yaml\|value` | `json` prints the half's reply line exactly as it came (for `watch`, every line, the default there); `yaml` (default for the others) the sample or call for people; `value` the message only |
| `--field a.b.0` | `sample`, `graph`, `watch`: one field of the message or of the graph |
| `--max-bytes <n>` | `sample`, `graph`, `watch`: lower the reply's size cap |
| `--request '<json>'` | `call`: the request as a JSON object (default: none is sent, and the half fills the type's defaults or sends the entry's fixed request) |
| `--request-id <id>` | `call`, `action`: the id a retry names; without it one is made up and printed on stderr; a retried goal is answered with the first goal, never a second |
| `--timeout <s>` | `call`: at most the entry's `timeout_s` |
| `--goal '<json>'` | `action`: the goal as a JSON object (default: none is sent, and the half fills the type's defaults or sends the entry's fixed goal) |

Exit codes: 0 a sample, a graph, or a call the server answered (a `duplicate` of one included); 1 the request ran and brought no answer (`no_publisher`, `timeout`, `blind`, a synthesized call, a duplicate whose first outcome is unknown); 2 there is no socket to ask (no `api:` block for the world, or nothing at the path), and the message names the fallback; 3 the half refused the request (`scope`, `rate`, `not_carried`, `not_allowed`, `not_served`, `invalid_request`, `not_watchable`, `watchers` and the rest), or closed the connection without a reply because it does not admit your uid. `watch` reads these codes its own way (the section above): 0 is a watch the half closed, 1 a lost half; so do `action` and `cancel` (their section).

## The fallback rule

The socket is a faster path, never the only one. A tool that may restart the half beside it (a driver watchdog, a world recycle) keeps a path that needs no half: when it cannot connect, gets end of file, or a reply `blind`, `busy`, `rate` or `size`, it falls back to `ros2 topic echo` or `ros2 service call` and logs which path answered. When the reply says the request is wrong (`scope`, `not_carried`, `not_allowed`, `not_served`, `invalid_request`, `not_watchable`), it reports that instead of falling back. A reply whose `instance` differs from the previous one comes from a restarted half: its record of request ids is new, so a retry there runs again.

A goal never falls back: on `goal_busy`, `goals` or a stream that ends without a result, a tool reports it and retries through the socket, because another client would send a goal the half does not know and cannot cancel. The rule covers every action stream.

A watch falls back the same way when it cannot connect, when its stream ends without a closing line, or when no line comes for three times `keepalive_s`: it polls with `sample` or follows the topic with `ros2 topic echo` until the half answers again, then opens a new watch. `blind` keepalives are not a reason to fall back: the half is alive and says so. `watchers` means every watch slot is taken: wait and retry, or poll with `sample` meanwhile; it is not a fault of the half. A closing line with `reason: reload` means the name left `api.retain`, which is a configuration change to report, and `teardown` means the half is stopping, after which the tool reconnects as after any restart.

## Status and forensics directories of their own

Both halves write into `ipc.dir` and `forensics.dir` by default, which both containers share. `worlds.<w>.status_dir` and `worlds.<w>.forensics_dir` give one half directories of its own (absolute paths, created 0700 when missing, refused at start with `dirs.permissions` when another uid owns them or anyone else may read or write them): the half's `status_<world>.json` lands in its `status_dir`, its audit rings, black box, violation ring, api audit file and history ring in its `forensics_dir`. Mount each into that half's container only and the other half can no longer write the files a root reader trusts. A reader takes the path from the config: `airlock doctor`, `top`, `api`, `history` and `trace` do, and every one of them also takes `--status <world>=<path>`. The two keys are fixed at start: a reload that moves either is refused in place, naming the key. `check` warns `dirs.shared` when the listener's own directory still lies under `ipc.dir` or the other half's directory, which would close nothing.

## What it costs the control lanes

Nothing on the engine's side of the wall: the requests run on one thread, `alk-api`, at normal priority, which takes no lock a lane's pump takes and never touches a lane. A sample creates and removes one reader inside the half's world, which the DDS itself serializes with the other endpoints of that participant, so a real-time pump publishing at that moment can wait on it; the sample rate limit (5 a second by default) bounds how often, and the release's rig run measures the effect on the tf and control lanes.

The world pays for each sample, though: every participant in it, the driver's nodes included, processes the reader's arrival and departure. On a Lynx (Raspberry Pi 4) one sample of the e-stop cost the driver container 30 to 40 ms of CPU, so a tool that samples continuously at the default limit of 5 a second costs the driver about 17 % of one core, and one a second about 4 %, while `/tf` latency did not move. That is still far cheaper than a `ros2 topic echo --once`, whose fresh participant cost the same driver about 1.3 s of CPU per read.

The remedy for a topic a tool polls, the latched control topics above all, is `api.retain` (engine 1.26.1): list the topic there and the half creates one reader for it once a publisher of it appears and keeps it for its whole run, so a sample of it creates nothing in the world and the driver pays nothing per sample. The reader is one more endpoint in the world for as long as the half runs, which is why the list is explicit (at most 16 names, each an exact topic entry this world carries out or, from 1.27.0, the destination name of one delivered into it, which `samples` allows and whose `max_bytes` fits `api.max_raw_sample_bytes`); `check` lists the retained lanes and that cost. A retained answer is the newest message the reader took, never an older one: before the first message arrives a sample waits for it up to `--wait` and then answers `no_publisher` or `timeout`, and while a participant rebuild has the reader down it answers `blind`. `--fresh` (`"fresh": true` on the socket) still reads through a new reader of its own, at the old cost. For a topic you do not retain, poll no faster than you need: set `api.rate.samples` to `{ per_s: 1, burst: 5 }` on a robot whose tools poll, and read state the status file already carries (health, the lane rows, `last_sample_age_s`) from `graph` instead of a sample.

## Where to look

- the status file's `api` block: whether the socket is bound, the connections, every request and refusal by its code, the `api.local` names with their uses, the surface's own service clients, the retained lanes with their readers' state (`api.retained`), the open watches (`api.watchers`) and the watches' counters (`api.watch`) ([status reference](status-and-health-reference.md#the-host-request-surface));
- each `tx` row's (and, from 1.27.0, each `rx` row's) `samples`, `sample_errors` and `last_sample_age_s`, each serving `services` row's `local_calls`, and `api.uids` for each caller's budget;
- `ros2 airlock api` marks a lane the host sampled in the last status period `host read` and lists every `api.local` name as a row carried by `local`, with the serving half's evidence alone;
- `ros2 airlock top` prints `api: N conn, samples S/s, calls C/min, refused R` per half;
- `ros2 airlock doctor`: `api.socket_unbindable`, `api.socket_dir`, `api.refused_uid`, `api.call_synthesized`, `api.type_unsupported`, `api.thread_stopped`, `api.watch_stalled` (1.28.0), `api.local_call_enabled`, and from `check` `api.socket_under_ipc_dir`, `api.trust_own_uid` and `dirs.shared`.
