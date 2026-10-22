# Expose a service

A service server in one world becomes callable from the other by listing it under the direction whose `from` world hosts the server. Here the robot's e-stop becomes callable from the fleet world:

```yaml
    - from: robot
      to: fleet
      namespace: /my_robot
      topics:
        - topic: /chatter
          type: std_msgs/msg/String
          max_bytes: 65536
          max_hz: 50
      services:
        - service: /hardware/e_stop_trigger
          type: std_srvs/srv/Trigger
          timeout_s: 2.0
          max_request_bytes: 1024
```

```bash
# fleet world: call the robot's service on its namespaced name
ROS_DOMAIN_ID=18 ros2 service call /my_robot/hardware/e_stop_trigger std_srvs/srv/Trigger
```

The call crosses the wall, executes on the robot's real server, and the response comes back. Unlisted services do not exist in the fleet world: `ros2 service list` there shows only what you exposed.

## The rules services follow

**Direction means server location.** `services` under `from: robot` exposes robot-world servers to fleet-world callers. The reverse direction's `services` list would expose fleet-world servers to the robot, which the production profile leaves empty.

**No call ever hangs.** `timeout_s` is the ceiling on every call. Upstream server down, peer half dead, entry removed by a reload, plain timeout: every failure path returns an error reply to the caller within bounded time. A dead robot-side server can never wedge a fleet-side caller's future.

**A reply the airlock had to invent says so, and you must read it.** ROS services carry no error channel, so a call the bridge could not complete comes back as a default-initialized response — for `std_srvs/srv/Trigger` that is `success=false`, which on its own is exactly what a server refusing looks like. So every invented reply puts the reason in the response's `message` field, prefixed `airlock:`, and says that the request WAS forwarded and may have been executed:

```text
response:
std_srvs.srv.Trigger_Response(success=False, message='airlock: TIMEOUT after 2.0 s waiting for the server's reply — the request was forwarded and MAY HAVE BEEN EXECUTED; this is not a refusal, verify the robot's state before retrying')
```

Treat that as "unknown", never as "no" — this bit us on a real robot, where an e-stop trigger the caller was told had failed had in fact latched. Automation calling a safety service should check the `message` prefix (or the robot's actual state) before deciding a command did not take effect. The halves count these per service in `status_<world>.json` and `/metrics` (`airlock_service_timeouts_total`), and `airlock doctor` fails while any are recorded — a service that times out regularly needs a longer `timeout_s` or a faster server, not a caller that learns to ignore it.

**Rename an exposed name that would collide.** `rename: /localization/local/enable` on a service entry changes only what the fleet world sees: the service appears as `namespace + rename` instead of `namespace + service` (schema 3). Use it when the fleet side runs its own service under the robot's default name — an off-robot navigation stack with its own `localization/*` EKF services is the canonical case; the robot's become `/my_robot/localization/local/*` and stop colliding. The robot-world name in `service:` is untouched, and exposed names must be unique within a direction (two entries landing on one exposed name fail validation).

**Calls in flight are bounded, and none is lost inside the bound.** `max_inflight` (default 8) is how many calls to one service may be open across the wall at once. A call past it gets an immediate reply whose `message` starts with `airlock: busy` and says the request was NOT forwarded, so it is safe to retry; both halves enforce the bound. Up to the bound, a burst of calls loses nothing, even when the robot's CPU stalls the bridge for a moment: each service's request and response channels hold `max(queue_depth, max_inflight)` messages, so a plain burst of ten calls at `queue_depth: 2` no longer loses one in about thirty-six. That memory has a price: a channel costs `(cap + 32) x (max(queue_depth, max_inflight) + 3)` bytes, so raise `max_inflight` only for a client that really keeps more calls open (a parameter dump, a reconfigure), and keep the caps small. A request that waited in the channel longer than the caller's `timeout_s` is skipped instead of run, because its caller already got the timeout reply; the halves count these as `dropped_stale` and `airlock doctor` mentions them as information.

**Requests and responses are bounded.** `max_request_bytes` rejects oversized requests before they cross, and `max_response_bytes` (schema 7) sizes the response channel and caps what a server may answer. It defaults to the request cap, which is too small for a service that returns a list, so state it on those: `max_response_bytes: 65536` on `list_controllers`, for instance. A response above the cap is dropped and the caller gets the timeout reply. Services are for commands and parameters, not bulk data; the shared-memory fast path is for topics.

**A service holds shared memory only while it is used.** A service's channels are created by its first call and released after `airlock.service_idle_s` (default 30 s, 0 to 3600, 0 for never) without a call and with nothing in flight, on both halves. Until then `airlock api` shows the channel as `released` and the service costs no memory and no thread. The price is a cold first call, which has to open the channels and, on the robot side, create its client: about 55 to 90 ms to arm on a NUC, on top of the call itself, and the next call within the idle time is as fast as before. A service that must answer at once, such as an e-stop, should not wait for that: give the entry `protected: true` or a `timeout_s` under 0.5 s, and its channels are created at start and kept. Keep in mind that a caller on the fleet network can tell a cold call from a warm one, so it learns whether someone called a service within the last idle time; `service_idle_s: 0` or a pinned entry removes that.

**Reloading keeps services working.** A reload of a changed file, or a restart of one half against the other, does not drop a service whose settings did not change: its calls in flight survive, and `airlock check --dry-run` lists such lanes as kept. A changed cap, type or `max_inflight` re-creates that one service.

**Docker's `/dev/shm` must hold the standing set.** The channels of every exact entry are reserved when a half starts, and a half refuses to start when they do not fit in its `/dev/shm`, naming both sizes. Docker gives a container 64 MiB by default, so give the container that owns the shared IPC namespace more: `docker run --shm-size=1g`, or `shm_size: 1gb` in compose. Services found through a glob are not part of that set; they are admitted while the halves run, within `ipc.shm_budget`.

**The type's support library must be installed in both halves' containers.** Topics cross as raw bytes with no type code involved, but services convert between wire and in-memory representation at each world edge, which loads the type's support library at runtime. Standard ROS types ship with the image. For your own interface packages, extend the image with your interface debs (`FROM husarion/ros2-airlock` plus an `apt install` of your packages). A service whose typesupport is missing is skipped with a diagnostic, never fatal, and `airlock doctor` points at exactly this.

## Expose many services with a glob

A `service:` value with a `*` carries every service whose name matches, found while the halves run. `*` matches within one name segment and a segment that is exactly `**` matches one or more whole segments; a name with a hidden segment (one starting with `_`) never matches. An entry written for the same service name always wins over a glob, with its own settings:

```yaml
      services:
        - service: /controller_manager/*
          types: [controller_manager_msgs/srv/ListControllers, controller_manager_msgs/srv/SwitchController]
          max_matches: 20
          timeout_s: 5.0
          max_request_bytes: 1024
          max_response_bytes: 65536
```

An action glob works the same way, under `actions` with `action:` and `types: [pkg/action/Name]`; it matches an action only when all five of its parts exist, and each match counts as five lanes.

- **No `type:`.** The type is read from the server. `types` limits which types the glob carries, and `max_matches` (1 to 65535) is the most names it carries at once. Names past it, and names past `airlock.max_lanes`, are skipped and counted, never refused as a file; `ros2 airlock api` and `trace` say which.
- **The first matching glob in file order wins a name,** and `exclude:` removes names from globs. An item starts with `service:` or `action:` to apply to that kind only, so `exclude: ["service:/**/set_parameters"]` leaves topics alone; a bare item applies to every kind.
- **A glob cannot rename.** A name crosses under the direction's namespace as it is. Where the other side expects bare names, as UGV OS's robot nodes do, write exact entries with `rename:`; `airlock check` prints a note on every inbound glob under a namespace.
- **A glob carries a name only once it can see every server behind it.** A matched service appears about a second after its server does. A name that two programs offer with different types is left out and named in the status, and so is a name whose server sits beside a client of another type, since the half cannot tell which type the server answers with. A half whose world mixes a Humble program into a Jazzy or Lyrical world cannot see node names (`ros2 airlock doctor` says `discovery.foreign_release`), so its globs carry nothing there: write exact entries for that world.
- **A glob costs nothing until a call.** Its channels are reserved only when the service is called, so `airlock check` reports a glob as unbounded by the file and the live count comes from `airlock api`. Make sure the shared memory budget has room for the `max_matches` you ask for.
- **Inbound globs deserve care.** A glob under `from: fleet` exposes servers of the fleet world inside the robot world, and any process in the fleet world can create a server whose name matches the pattern, so the airlock will expose it. The loader requires `types` and `max_matches` on it, and refuses a pattern without a literal segment unless its block sets `allow_open_inbound: true`. Use an inbound glob only for a namespace nothing else in the fleet world may use.

## Actions

Actions work the same way, under an `actions` list. A Nav2 goal sent in the fleet world executes on the robot, with live feedback, status, and cancel:

```yaml
      actions:
        - action: /navigate_to_pose
          type: nav2_msgs/action/NavigateToPose
          timeout_s: 60.0
```

The airlock bridges the goal, result, and cancel services plus the feedback and status topics as one unit, and correlates goals across the wall by their UUID. Fleet-side tooling sees a genuine action server on `/my_robot/navigate_to_pose`; Open-RMF fleet adapters work through it unmodified. Note that `timeout_s` bounds every bridged call of the action, and the result request completes only when the goal itself finishes, so set it longer than your longest expected goal. The Nav2 test configurations use 60 s.
