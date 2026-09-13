# Bridge two ROS 2 releases

Your robot runs Jazzy and your fleet still runs Humble, or your fleet moved to Lyrical before the robot did. One airlock joins them: its robot half runs on the robot's release, its fleet half on the fleet's, and the message types that changed between the two releases are converted field by field on the way through. Nothing changes in the config file: it names topics and types, never a release.

The airlock ships one image and one deb per release, all from one engine version:

| Release | Docker image | Deb |
| -- | -- | -- |
| Jazzy | `husarion/ros2-airlock:1.19.0` (also `:latest`) | `ros-jazzy-ros2-airlock` |
| Humble | `husarion/ros2-airlock:1.19.0-humble` | `ros-humble-ros2-airlock` |
| Lyrical | `husarion/ros2-airlock:1.19.0-lyrical` | `ros-lyrical-ros2-airlock` |

A cross-release pair is two containers from two of these images, sharing an IPC namespace and a volume. Save the config from [Bridge your first topic](bridge-your-first-topic.md) as `airlock.yaml` with one more topic, the one whose definition changed on Lyrical:

```yaml
      topics:
        - topic: /chatter
          type: std_msgs/msg/String
          max_bytes: 65536
          max_hz: 50
        - topic: /transition_event
          type: lifecycle_msgs/msg/TransitionEvent
          max_bytes: 4096
          max_hz: 50
```

Then this `compose.yaml` beside it runs a Jazzy robot world (domain 17) and a Lyrical fleet world (domain 18) on one machine:

```yaml
volumes:
  airlock_ipc:

services:
  anchor:                       # holds the IPC namespace the others join
    image: alpine:3
    command: sleep infinity
    ipc: shareable
    shm_size: 512mb

  airlock_robot:                # the robot's release
    image: husarion/ros2-airlock:1.19.0
    ipc: service:anchor
    environment: { RMW_IMPLEMENTATION: rmw_fastrtps_cpp, ROS_DOMAIN_ID: "17" }
    volumes:
      - ./airlock.yaml:/etc/airlock/airlock.yaml:ro
      - airlock_ipc:/tmp/airlock
    command: ["bash", "-c", "source /opt/ros/$$ROS_DISTRO/setup.bash && exec /opt/ros2_airlock/lib/ros2_airlock/airlock_half --world robot -c /etc/airlock/airlock.yaml"]

  airlock_fleet:                # the fleet's release
    image: husarion/ros2-airlock:1.19.0-lyrical
    ipc: service:anchor
    environment: { RMW_IMPLEMENTATION: rmw_fastrtps_cpp, ROS_DOMAIN_ID: "18" }
    volumes:
      - ./airlock.yaml:/etc/airlock/airlock.yaml:ro
      - airlock_ipc:/tmp/airlock
    command: ["bash", "-c", "source /opt/ros/$$ROS_DISTRO/setup.bash && exec /opt/ros2_airlock/lib/ros2_airlock/airlock_half --world fleet -c /etc/airlock/airlock.yaml"]

  talker:                       # stands in for your robot: same image as its half
    image: husarion/ros2-airlock:1.19.0
    ipc: service:anchor
    environment: { RMW_IMPLEMENTATION: rmw_fastrtps_cpp, ROS_DOMAIN_ID: "17" }
    command: ["bash", "-c", "source /opt/ros/$$ROS_DISTRO/setup.bash && ros2 topic pub -r 2 /transition_event lifecycle_msgs/msg/TransitionEvent '{timestamp: 12345000000, transition: {id: 3, label: activate}}' >/dev/null & exec ros2 topic pub -r 2 /chatter std_msgs/String 'data: hello through the airlock'"]

  listener:                     # stands in for your fleet tools: same image as the fleet half
    image: husarion/ros2-airlock:1.19.0-lyrical
    ipc: service:anchor
    environment: { RMW_IMPLEMENTATION: rmw_fastrtps_cpp, ROS_DOMAIN_ID: "18" }
    command: ["bash", "-c", "source /opt/ros/$$ROS_DISTRO/setup.bash && exec ros2 topic echo /my_robot/chatter std_msgs/msg/String"]
```

```bash
docker compose up -d
docker compose logs -f listener            # data: hello through the airlock
docker compose exec listener bash -c 'source /opt/ros/$ROS_DISTRO/setup.bash && ros2 topic echo --once /my_robot/transition_event lifecycle_msgs/msg/TransitionEvent'
```

The talker published `timestamp: 12345000000`, a plain nanosecond count, which is what Humble's and Jazzy's `TransitionEvent` carries. The listener on Lyrical prints `stamp: {sec: 12, nanosec: 345000000}`, because Lyrical's message carries a `builtin_interfaces/Time` there instead. The fleet half read the robot side's definition, saw it differ from its own, and applied the rule the image ships for exactly this change. `/my_robot/chatter` crossed untouched: `std_msgs/msg/String` is the same on every release.

In the airlock's source checkout the same stack is `just demo-cross <a> <b>` for any two of `humble`, `jazzy`, `lyrical`; it prints one verdict line per topic. A `ros2 launch` pair, like the one-command demo in the README, is always single-release: both halves come from the one install you sourced.

## What the doctor says about a pair

Run the doctor in either half's container, with that half's environment:

```bash
docker compose exec airlock_fleet bash -c 'source /opt/ros/$ROS_DISTRO/setup.bash && /opt/ros2_airlock/lib/ros2_airlock/airlock doctor -c /etc/airlock/airlock.yaml'
```

```text
== definitions (proto 1.6) ==
  ok [fleet] 1 lane(s) run CONVERTED on the fleet half (field-level, SPEC-CROSS-RELEASE §5): /my_robot/transition_event
== channels ==
  ok [robot] /chatter -> /my_robot/chatter: flowing
  ok [robot] /transition_event -> /my_robot/transition_event: advertised (no flow: nothing subscribes to the destination topic yet — lazy §3.2)
```

Every topic, service and action the config names ends in one of four states, and the doctor names each one that is not plain `same`:

- **same.** Both halves hold the same definition of the type. The bytes cross untouched, exactly as on a single-release pair. Most types are `same` between any two releases: of the 563 types both the Humble and the Jazzy image carry, 513 are identical.
- **converted.** The definitions differ and one half, the fleet half by default, rewrites each message into the other definition. A field added in the newer release is filled with its default on the way to the older one and dropped on the way back; a renamed field or a changed unit needs a rule, and the image ships rules for the types that changed this way among those Husarion robots expose (the controller manager's services, the lifecycle transition event). The status file lists the dropped and defaulted fields and the rule per lane, and the doctor prints the lane. Conversion costs microseconds per message and runs on the half that is not the `ipc.listener`, the fleet half in every example here, so the robot's side never pays for it.
- **refused.** The definitions differ in a way the airlock will not guess about: a field changed its type or array size, a service or an action (which always need an explicit rule), or an entry you marked `protected` or `convert: deny`. A refused lane is absent from the destination world, and the doctor prints why and what fixes it, with a ready-to-paste rule when a rule is the fix. [Write a type rule](write-a-type-rule.md) walks through that.
- **unchecked.** The other half runs an engine older than 1.18.0, which cannot describe its types; unprotected topics cross as they always did and protected ones are refused. Upgrade both halves and this state goes away.

Topics that merely gained or lost a field convert automatically. Services and actions convert only under a rule that acknowledges every dropped or defaulted field, because a request silently missing a parameter is worse than a service that is visibly absent. You can tighten a topic the same way with `convert: explicit`, or forbid conversion outright with `convert: deny` (both need `schema: 5` in the config).

## The fleet half must match the fleet, not just the robot

The airlock bridges the difference between its two halves. It cannot bridge the difference between the fleet half and the other nodes in the fleet world, and there is one that matters: ROS 2 changed the layout of its own discovery message between Humble and Iron. A Humble node on a domain shared with Jazzy or Lyrical nodes cannot read their discovery information. Data still flows, but the node cannot see the graph, `ros2 node list` comes back empty or fails, and the airlock's fleet half reports itself blind and the deployment dysfunctional. Jazzy and Lyrical nodes read each other fine.

So pick the fleet half's release to match the nodes it will share a domain with: a Humble fleet gets the `-humble` image, a Jazzy or Lyrical fleet gets `:1.19.0` or `-lyrical`. The robot half follows the robot the same way (on Husarion robots it always does). If your fleet is mid-migration and both releases are on the network at once, give each release its own domain ID and run two airlock pairs from the robot, one per fleet domain, each with its own `ipc.dir` and its own fleet half on the matching image. The doctor names this situation: `this world contains participants of another ROS 2 release (Humble vs Iron+ discovery layout)`, with what the half saw, and the half reports itself dysfunctional with the same cause rather than merely blind. Data keeps crossing while it does, and the half does not try to re-create its participant over it, because a new participant of the same release reads the same samples.

## Where the rules live

The image ships its rules in `/opt/ros2_airlock/share/ros2_airlock/type_map.d/`; your own go in a `type_map.d/` directory next to `airlock.yaml`, mounted into both halves at the same path. Both halves read both sets. A change to your rules is picked up on `SIGHUP`, like a config change, and a rule file that fails validation is rejected as a whole while the previous set keeps running, with the reason in the doctor's output.

## On Husarion robots

On Panther and Lynx the robot half runs on the robot's release and Husarion Cockpit will offer the fleet half's release next to the RMW and domain selection on the Network tab, with a live line showing how many topics are the same, converted or refused for that choice. Until then the same switch is a change of the fleet half's image tag in the deployment.

## Next

[Write a type rule](write-a-type-rule.md) turns a refused lane into a converted one. [Check your own types before you deploy](check-your-types-before-you-deploy.md) shows the whole picture for two images of your own before anything runs on a robot.
