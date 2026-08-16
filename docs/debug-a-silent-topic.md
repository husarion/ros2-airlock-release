# Debug a silent topic

A topic you expected in the destination world is missing, or present but silent. Start with the doctor, in a shell with the same environment as one of the halves:

```bash
ros2 run ros2_airlock airlock doctor -c airlock.yaml
```

```text
== env (this shell's world) ==
  ok RMW_IMPLEMENTATION = rmw_fastrtps_cpp
== config ==
  ok parsed; worlds: robot, fleet; ipc.dir: /run/airlock
== ipc dir ==
  ok /run/airlock exists (shared volume mounted in both halves?)
  ok control socket present: /run/airlock/control.sock (listener half up?)
  ok /dev/shm free: 412 MiB (channels preallocate max_bytes x depth; size it for the shm budget, §3.1)
== halves ==
  ok half 'robot': status age 2 s (written every 5 s; stale = hung or dead)
  ok half 'robot': control-plane peer alive
  ok half 'robot': config epochs matched (mismatch = halves read different files, §3.6)
== channels ==
  ok [robot] /scan -> /panther_1234/scan: advertised (no flow: nothing subscribes to the destination topic yet — lazy §3.2)
warn [fleet] /panther_1234/cmd_vel -> /cmd_vel: flowing DROPPING {'dropped_size': 3181} (rate = max_hz cap; size = max_bytes cap)
```

The doctor checks each half's environment, the IPC directory, the control plane, and every channel's state and drop counters, and prints why for anything unhealthy. If it says `doctor: healthy` and the topic still looks silent, the problem is usually the consumer side; the ladder below runs from most to least common.

## The ladder

**1. Nothing is subscribed in the destination world.** Bridging is lazy: `advertised` is the healthy resting state and means the topic exists in the destination world but no one there subscribes yet, so no data crosses. `ros2 topic echo <destination name> <type>` makes it flow. If it must flow with no subscribers (latched topics that late joiners need), set `activation: always` on the entry.

**1b. The channel says `degraded`, not `advertised`.** This is the state that used to hide behind the healthy resting one: the topic is advertised, something in the destination world IS subscribed, and activation keeps failing — so nothing can ever cross. The doctor names the attempt count and the transport error, and the half repeats the warning every minute. The usual cause is a transport capacity limit (§3.1): `ipc.shm_budget`, or the wake service, whose capacity is frozen when it is created — a config grown by a `SIGHUP` reload needs BOTH halves restarted to resize it.

**2. The entry doesn't match, or matches in the wrong direction.** Names in a direction are `from`-world names, before the namespace prefix. The prefix applies on republish, so the entry says `/scan` even though the fleet world sees `/panther_1234/scan`. Run `ros2 run ros2_airlock airlock check airlock.yaml` after every edit; it catches structural mistakes offline.

**3. A cap is dropping.** The `DROPPING` counters name the cap: `dropped_size` means messages exceed `max_bytes` (a common surprise on cameras and point clouds; the default is 10 MiB but the hardened profile sets far less), `dropped_rate` means `max_hz` decimation, and `evicted` means drop-oldest under overload, which is by design and only the newest data survives.

**4. The type doesn't agree.** A typed entry rejects a publisher of a different type, and a topic seen with two different types in the source world is skipped entirely; both raise a warn-once diagnostic in the half's log. In merged TF mode there is a stricter variant: frame rewriting fails closed, so a stamped type whose introspection typesupport is missing in the container drops every message and the doctor reports `dropped_rewrite` with the fix (install the interface package in the half's image).

**5. QoS doesn't match the consumer.** The airlock adapts its publisher QoS from the source endpoint unless overridden. A best-effort bridged topic will not deliver to a subscriber demanding reliable. `ros2 topic info -v <topic>` in the destination world shows both endpoints' QoS; add a `qos:` override to the entry if the consumer can't be changed.

**6. The halves aren't paired.** The doctor's `halves` section tells you: a missing status file means that half isn't running, a missing control socket means the `ipc.listener` half isn't up or the two halves see different `ipc.dir` volumes, and an epoch mismatch older than the 30 s grace window means the halves are reading genuinely different files — converge them and `SIGHUP` both.

**7. `/dev/shm` is too small, or the budget refused the channel.** Channels preallocate `max_bytes × depth` at activation. In containers the default 64 MiB `/dev/shm` disappears fast under big `max_bytes`; set `shm_size` explicitly, and watch for the budget-refused diagnostic when the sum of active channels would exceed `ipc.shm_budget`.

## Tooling can lie to you

Two `ros2` CLI habits matter when your terminal hops between worlds. The CLI daemon caches the graph per environment it was started in, so after switching `ROS_DOMAIN_ID` or RMW in a shell, `ros2 topic echo` may claim it cannot determine a topic's type: pass the type explicitly (as the snippets on these pages do) or run `ros2 daemon stop` after switching. And remember an echo is itself a subscriber: observing a lazy topic activates it, so "it only flows when I look at it" is not a bug, it is the design working.
