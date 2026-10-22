# Threads and CPU pinning

Every thread the airlock engine starts has a stable name, so `top -H`, `ps -L`, `/proc/<pid>/task/*/comm` and a CPU pinner can tell a lane pump from the executor and from the threads your RMW starts. This page lists the names, says which threads sit on the latency path, and says what a pinner should and should not do with them.

## The scheme

A Linux thread name holds 15 characters. Every engine thread is named `alk-<role>[<class>][-<n>]`:

- `<role>` says what the thread does (the table below).
- `<class>` is one letter on a destination pump only: `c` control, `n` normal, `b` bulk, the class of the lane it serves.
- `<n>` is the lane's index on a pump, or the worker's number on an executor. A lane keeps its index while it lives and a retired lane's index is reused. The longest name is 12 characters (`alk-rxb-4095`).

This is version 1 of the scheme. `airlock_half --constants` prints it as `"thread_names": 1`, and a rename within a major release bumps that number and is announced in the changelog.

The main thread is the one exception: it keeps the binary's name `airlock_half`, so `pgrep airlock_half`, `pidof`, `killall` and journald's `_COMM` keep finding a half. The engine starts no other thread under that name.

| Name | Role | On the latency path |
| -- | -- | -- |
| `airlock_half` | the main thread: start, reload and stop | no |
| `alk-sig` | consumes SIGINT, SIGTERM and SIGHUP, bounds the stop | no |
| `alk-initwd` | ends a start that wedges in the RMW (exits once started) | no |
| `alk-ctrlio` | the control plane between the two halves (leases, demand, lane state) | indirectly: a lane activates through it |
| `alk-exc-<n>` | executor worker n: source takes, timers, publisher and subscriber callbacks of `control` and `normal` lanes | yes, the source hop of every lane |
| `alk-exb-<n>` | bulk executor worker n: the callbacks of `bulk` lanes | bulk traffic only |
| `alk-rxc-<n>`, `alk-rxn-<n>`, `alk-rxb-<n>` | destination pump of lane n: shared memory to the DDS publish | yes, the destination hop; `c` lanes first |
| `alk-sd-<n>` | caller-side service pump of lane n (responses back to your client) | yes, for that service |
| `alk-ss-<n>` | serving-side service pump of lane n (requests to your server) | yes, for that service |
| `alk-heal` | discovery health, the cross-check and the participant rebuild | no |
| `alk-rebuild` | bounds a participant rebuild (lives only during one) | no |
| `alk-deadman` | publishes a deadman lane's neutral value when its stream stops | yes, for deadman lanes |
| `alk-cap` | reserved for the per-lane byte cap | no |
| `alk-metrics` | serves `/metrics` | no |
| `alk-blackbox` | writes the black box bag | no |
| `alk-audit` | writes the audit ring | no |
| `alk-history` | writes the history ring | no |
| `alk-api` | the host request socket: the graph, one sample, one call; off unless the world's config has an `api:` block (1.26.0) | no |
| `alk-flap` | a test lever only, never on a deployment | no |

Threads your RMW, DDS or the tracer start keep their own names (Fast DDS `dds.*`, Cyclone DDS `recv`, `tev`, `gc`, `dq.*`, Zenoh's runtime threads, LTTng's `*-ust`). A thread a library starts without naming it, such as rcl logging's flusher or Fast DDS's discovery listener, carries `ros-lib`. None of these are the engine's, and the airlock does not rename them.

## What the status says

Each half's status file lists its threads, refreshed on every write:

```json
"threads": [
  {"name": "airlock_half", "tid": 1, "role": "main", "class": null, "lane": null},
  {"name": "alk-rxn-0", "tid": 87, "role": "rx", "class": "normal", "lane": "/lynx/odometry/filtered"},
  {"name": "dds.ev.0", "tid": 41, "role": "foreign", "class": null, "lane": null}
]
```

`role` is the scheme role, `foreign` for an RMW or DDS thread, and `off_scheme` for a thread carrying an `alk-` name the engine did not give it, or `airlock_half` on any thread but the main one. A thread inherits its creator's name, so an engine thread somebody forgot to name shows up under its parent's name, and the status calls it `off_scheme`. `airlock doctor` reports each one as `half.thread_off_scheme`. That is a bug in the engine build, and a pinner matching by name would put that thread in the wrong group. The `tx`, `rx` and `services` rows carry `thread`, the name of the pump serving that lane while one runs. A source lane (`tx`) has no pump of its own because its takes run on `alk-exc-*`, so its `thread` is always null.

## Pinning

CPU placement belongs to the deployment. On UGV OS the pinner places threads by name. The intended use:

- Keep `alk-rxc-*`, `alk-exc-*`, `alk-sd-*` and `alk-ss-*` on the CPUs your control traffic gets.
- Confine `alk-exb-*` and `alk-rxb-*`, the bulk executor and the bulk pumps, to the CPUs you can spare for camera-sized traffic.
- Leave the rest (`alk-heal`, `alk-metrics`, `alk-history`, `alk-audit`, `alk-blackbox`, `alk-api`) wherever is cheapest. `alk-api` always runs at normal priority, never SCHED_FIFO, and takes no lock a real-time pump holds.
- Never raise a thread's scheduling policy or priority from outside. The engine owns the policy of its threads.

Affinity is the deployment's decision. The scheduling policy of the real-time eligible pumps is the engine's, from its config, as the next section says.

## Real-time priority

The engine can run the destination pumps of `control` topic lanes at SCHED_FIFO, so a `/tf` frame or a command that has crossed the wall is published into its world ahead of everything that runs at normal priority. You turn it on in the config:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 9
airlock:
  ipc: { dir: /run/airlock, listener: robot, queue_depth: 2 }
  rt: { pump_fifo_priority: 5 } # pump_batch: 8 and max_frame_bytes: 65536 are the defaults
  worlds: { robot: {}, fleet: {} }
  directions:
    - from: fleet
      to: robot
      topics:
        - topic: /cmd_vel
          type: geometry_msgs/msg/TwistStamped
          max_bytes: 4096
          max_hz: 50
          class: control
```

| Key | Range | Default | What it does |
| -- | -- | -- | -- |
| `pump_fifo_priority` | 1-49 | absent: no real-time priority | The SCHED_FIFO priority of the eligible pumps. Use 5-10: below the kernel's threaded interrupts (50) and a driver's real-time threads (60), with room for your own real-time work. `airlock check` warns above 10. |
| `pump_batch` | 1-64 | 8 | The frames a real-time pump takes in one wake before it waits up to 1 ms, which hands the CPU to the other work on it. |
| `max_frame_bytes` | 4096-1048576 | 65536 | The largest `max_bytes` a real-time pump serves. The time a pump holds the CPU grows with the frame, so a `control` lane above it runs without real-time priority (`rt: ineligible:max_bytes` on its row, and `airlock check` warns). |

Only the `alk-rxc-*` pumps are eligible: the pumps of `control` topic lanes (`/tf`, `/tf_static`, a `protected: true` entry, an explicit `class: control`, an action's feedback and status) whose `max_bytes` is within `max_frame_bytes`. The executors, the service pumps (`alk-sd-*`, `alk-ss-*`), the `normal` and `bulk` pumps and every other engine thread stay at normal priority. That means the source hop of a control lane, the take on `alk-exc-*`, is not real-time: the class split protects it, and the priority covers the hop from shared memory into the destination world. The tf block's lanes carry the block's `max_bytes`, 64 KiB unless `tf.max_bytes` says otherwise, so with the default `max_frame_bytes` they are eligible; a tf block set above `max_frame_bytes` runs without real-time priority and `airlock check` warns.

A reload that changes `airlock.rt` applies the new policy to every eligible pump at once, and a lane whose class changes moves in or out of SCHED_FIFO with it. Nothing restarts.

### Permission

The container needs an `rtprio` ulimit at least as high as the priority: `docker run --ulimit rtprio=<n>`, or `Ulimit=rtprio=<n>` in a Podman quadlet. CAP_SYS_NICE also works; the engine asks for neither and adds no capability. On a host that still uses cgroup v1 with `CONFIG_RT_GROUP_SCHED`, the container's cgroup also needs a `cpu.rt_runtime_us` budget; the engine reports that refusal the same way. UGV OS uses cgroup v2.

When the call is refused, the half logs it once at WARN with the error and the limit, and runs on without real-time priority. The status says so:

```json
"process": {"uid": 10001, "gid": 10001, "caps_effective": [], "fds": 40, "fd_limit": 1048576,
  "rt": {"requested": 5, "applied": 0, "eligible_threads": 2, "applied_threads": 0,
         "error": "EPERM (Operation not permitted)", "rtprio_limit": 0}}
```

`applied` is the priority when every eligible pump runs at it, and 0 otherwise. `airlock doctor` reports `process.rt_refused` with the fix. Each `rx` row carries `rt` (`fifo:5`, `refused`, `ineligible:max_bytes`, or null for a lane that is not real-time), `rt_batch`, and two figures from the pump itself: `rt_run_max_us`, the longest stretch of CPU time it spent in one wake, and `rt_frames_max`, the most frames it took between two waits (at most `pump_batch`). `chrt -p <tid>` shows the policy from outside; the tids are in the status's `threads` list.

### Sharing CPUs with normal work

A SCHED_FIFO thread runs before every normal thread on its CPU until it sleeps. The engine bounds what a real-time pump does per wake (at most `pump_batch` frames, then a wait of at most 1 ms when more are queued) and its hot path takes no lock a normal thread holds: it counts, and leaves the logging, the history and the peer strikes to `alk-history`. `rcl_publish` into the destination world is the one shared path it keeps. On kernel 6.12 and later the fair deadline server also guarantees normal threads a share of each CPU whatever the real-time threads do; on older kernels the engine's bound is the only protection, beside the kernel's real-time throttle (`sched_rt_runtime_us`, 95 % of each second). If your halves share their CPUs with other containers (on UGV OS, the camera, rosbridge and the web server), measure them together: the release's rig and bench rows record how late a 1 ms sleep of a normal task wakes beside a flooded real-time pump, and that is the number to compare against.
