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

A lane the peer delivers into the half's world can be sampled too, by the name it has in that world: above, the fleet's `/cmd_vel` arrives in the robot world as `/autonomous/cmd_vel`, and that is the name to ask for and to list in `samples` or `retain`. The half reads it through a reader of its own on that name, so it sees what the robot's nodes see, its own publication of the lane included. The fleet's name `/cmd_vel` means nothing in the robot world and answers `not_carried`. The sample still is not demand: a lazy lane nobody in the robot world asks for stays idle, and a sample of it answers `timeout`. The name is known from the config from the moment the half starts: before the fleet publishes it at all, a sample answers `no_publisher` (or `timeout`), as for any lane without a publisher, never `not_carried`, and a name listed in `retain` reads `waiting` under `api.retained` and gets its reader once the lane's publisher appears in the robot world (engine 1.27.1; 1.27.0 answered `not_carried` until the fleet first advertised the lane).

### NaN and infinite values (engine 1.27.1)

JSON has no NaN or infinity, so a sample writes a float that holds one as the string `"NaN"`, `"Infinity"` or `"-Infinity"` and sets `nonfinite: true`. A string field could hold the same text, so every sample reply also lists the path of each such float in `nonfinite_paths` (`["temperature", "cell_voltage[3]"]`, at most 64, then `nonfinite_paths_truncated: true`), and a tool that has no `.msg` definitions can tell the two apart. A request with `"nonfinite": "null"` (`airlock_api sample --nonfinite null`) writes those floats as JSON null instead, as rosbridge does; `nonfinite_paths` still says which nulls they are. A `sensor_msgs/BatteryState` whose driver does not know the temperature is the common case:

```bash
airlock_api sample /battery/battery_status --nonfinite null --field temperature   # prints null
```

### A call with a fixed request (engine 1.27.0)

A `calls` item may be `{service: <name>, request: <the request's fields>}`, and an `api.local.services` entry takes the same `request:`. A host call of that name may leave the request out (the fixed one is sent) or send one equal to it once both are filled with the type's defaults; any other request is refused `not_allowed`, nothing reaches the service, and the audit file records the refusal. Above, the e-stop trigger can be pulled from the host but never with arguments someone else chose. The half checks the fixed request against the service's type when it starts and on every reload, and so does `check`; a name whose type the config leaves to discovery is checked at its first call. A name written bare keeps taking any request.

### A budget per caller (engine 1.27.0)

The rate limits under `api.rate` apply to each uid on its own: a tool running as one uid that spends its budget is refused `rate`, while the watchdog running as root still gets its full budget. A uid that has held no connection for 60 seconds starts with a full budget again. The status lists every uid with its tokens and refusals under `api.uids`.

## The verbs and the native client

`ros2 airlock sample <topic>`, `ros2 airlock call <service>` and `ros2 airlock graph` take the socket from the config (`-c`, or `AIRLOCK_CONFIG`, or the `-c` of a half running beside them) or from `--socket`, never from a status file: another process of the half's uid can write a status file and point a reader somewhere else. The name is the one in the half's own world: a source name, or the destination name of a lane delivered into it.

From engine 1.27.0 the three verbs run `airlock_api`, a small native program that speaks to the socket without starting Python or ROS (the image has it at `/opt/ros2_airlock/lib/ros2_airlock/airlock_api`). It takes the same arguments with the verb first, `airlock_api sample /hardware/e_stop --socket /run/airlock-robot/api/api.sock`, prints the same output and exits with the same codes. Every GitHub Release also carries it as one statically linked file per architecture, `airlock_api-<version>-linux-amd64` and `airlock_api-<version>-linux-arm64`, each with a `.sha256` beside it. It needs no ROS install and no shared library, so a host installs it by copying that file:

```bash
v=1.27.0; a=arm64
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
| `--nonfinite string\|null` | `sample`: write a NaN or infinite float as the string `"NaN"`, `"Infinity"`, `"-Infinity"` (default) or as null (1.27.1) |
| `--format json\|yaml\|value` | `json` prints the half's reply line exactly as it came; `yaml` (default) the sample or call for people; `value` the message only |
| `--field a.b.0` | `sample`, `graph`: one field of the message or of the graph |
| `--max-bytes <n>` | `sample`, `graph`: lower the reply's size cap |
| `--request '<json>'` | `call`: the request as a JSON object (default: none is sent, and the half fills the type's defaults or sends the entry's fixed request) |
| `--request-id <id>` | `call`: the id a retry names; without it the verb makes one up and prints it on stderr |
| `--timeout <s>` | `call`: at most the entry's `timeout_s` |

Exit codes: 0 a sample, a graph, or a call the server answered (a `duplicate` of one included); 1 the request ran and brought no answer (`no_publisher`, `timeout`, `blind`, a synthesized call, a duplicate whose first outcome is unknown); 2 there is no socket to ask (no `api:` block for the world, or nothing at the path), and the message names the fallback; 3 the half refused the request (`scope`, `rate`, `not_carried`, `not_allowed`, `not_served`, `invalid_request` and the rest), or closed the connection without a reply because it does not admit your uid.

## The fallback rule

The socket is a faster path, never the only one. A tool that may restart the half beside it (a driver watchdog, a world recycle) keeps a path that needs no half: when it cannot connect, gets end of file, or a reply `blind`, `busy`, `rate` or `size`, it falls back to `ros2 topic echo` or `ros2 service call` and logs which path answered. When the reply says the request is wrong (`scope`, `not_carried`, `not_allowed`, `not_served`, `invalid_request`), it reports that instead of falling back. A reply whose `instance` differs from the previous one comes from a restarted half: its record of request ids is new, so a retry there runs again.

## Status and forensics directories of their own

Both halves write into `ipc.dir` and `forensics.dir` by default, which both containers share. `worlds.<w>.status_dir` and `worlds.<w>.forensics_dir` give one half directories of its own (absolute paths, created 0700 when missing, refused at start with `dirs.permissions` when another uid owns them or anyone else may read or write them): the half's `status_<world>.json` lands in its `status_dir`, its audit rings, black box, violation ring, api audit file and history ring in its `forensics_dir`. Mount each into that half's container only and the other half can no longer write the files a root reader trusts. A reader takes the path from the config: `airlock doctor`, `top`, `api`, `history` and `trace` do, and every one of them also takes `--status <world>=<path>`. The two keys are fixed at start: a reload that moves either is refused in place, naming the key. `check` warns `dirs.shared` when the listener's own directory still lies under `ipc.dir` or the other half's directory, which would close nothing.

## What it costs the control lanes

Nothing on the engine's side of the wall: the requests run on one thread, `alk-api`, at normal priority, which takes no lock a lane's pump takes and never touches a lane. A sample creates and removes one reader inside the half's world, which the DDS itself serializes with the other endpoints of that participant, so a real-time pump publishing at that moment can wait on it; the sample rate limit (5 a second by default) bounds how often, and the release's rig run measures the effect on the tf and control lanes.

The world pays for each sample, though: every participant in it, the driver's nodes included, processes the reader's arrival and departure. On a Lynx (Raspberry Pi 4) one sample of the e-stop cost the driver container 30 to 40 ms of CPU, so a tool that samples continuously at the default limit of 5 a second costs the driver about 17 % of one core, and one a second about 4 %, while `/tf` latency did not move. That is still far cheaper than a `ros2 topic echo --once`, whose fresh participant cost the same driver about 1.3 s of CPU per read.

The remedy for a topic a tool polls, the latched control topics above all, is `api.retain` (engine 1.26.1): list the topic there and the half creates one reader for it once a publisher of it appears and keeps it for its whole run, so a sample of it creates nothing in the world and the driver pays nothing per sample. The reader is one more endpoint in the world for as long as the half runs, which is why the list is explicit (at most 16 names, each an exact topic entry this world carries out or, from 1.27.0, the destination name of one delivered into it, which `samples` allows and whose `max_bytes` fits `api.max_raw_sample_bytes`); `check` lists the retained lanes and that cost. A retained answer is the newest message the reader took, never an older one: before the first message arrives a sample waits for it up to `--wait` and then answers `no_publisher` or `timeout`, and while a participant rebuild has the reader down it answers `blind`. `--fresh` (`"fresh": true` on the socket) still reads through a new reader of its own, at the old cost. For a topic you do not retain, poll no faster than you need: set `api.rate.samples` to `{ per_s: 1, burst: 5 }` on a robot whose tools poll, and read state the status file already carries (health, the lane rows, `last_sample_age_s`) from `graph` instead of a sample.

## Where to look

- the status file's `api` block: whether the socket is bound, the connections, every request and refusal by its code, the `api.local` names with their uses, the surface's own service clients, the retained lanes with their readers' state (`api.retained`) ([status reference](status-and-health-reference.md#the-host-request-surface));
- each `tx` row's (and, from 1.27.0, each `rx` row's) `samples`, `sample_errors` and `last_sample_age_s`, each serving `services` row's `local_calls`, and `api.uids` for each caller's budget;
- `ros2 airlock api` marks a lane the host sampled in the last status period `host read` and lists every `api.local` name as a row carried by `local`, with the serving half's evidence alone;
- `ros2 airlock top` prints `api: N conn, samples S/s, calls C/min, refused R` per half;
- `ros2 airlock doctor`: `api.socket_unbindable`, `api.socket_dir`, `api.refused_uid`, `api.call_synthesized`, `api.type_unsupported`, `api.thread_stopped`, `api.local_call_enabled`, and from `check` `api.socket_under_ipc_dir`, `api.trust_own_uid` and `dirs.shared`.
