# Bridge your first topic

Write one file, start two processes, and a topic crosses from an isolated world into yours. Save this as `airlock.yaml`:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json
schema: 1
airlock:
  ipc: { dir: /tmp/airlock, listener: robot }
  worlds:
    robot: {}
    fleet: {}
  directions:
    - from: robot
      to: fleet
      namespace: /my_robot
      topics:
        - topic: /chatter
          type: std_msgs/msg/String
          max_bytes: 65536
          max_hz: 50
```

Then run one `airlock_half` per world, giving each the world's environment. Here the two worlds are just two domain IDs on one machine:

```bash
# terminal 1 — the robot-world half
ROS_DOMAIN_ID=17 ros2 run ros2_airlock airlock_half --world robot -c airlock.yaml

# terminal 2 — the fleet-world half
ROS_DOMAIN_ID=18 ros2 run ros2_airlock airlock_half --world fleet -c airlock.yaml

# terminal 3 — a publisher in the robot world
ROS_DOMAIN_ID=17 ros2 topic pub -r 2 /chatter std_msgs/String 'data: hello'

# terminal 4 — subscribe in the fleet world
ROS_DOMAIN_ID=18 ros2 topic echo /my_robot/chatter std_msgs/msg/String
```

Terminal 4 prints `hello` twice a second. The message was published in domain 17, crossed the airlock's shared-memory transport, and was republished in domain 18 under the `/my_robot` namespace.

## What just happened

**The worlds live in the environment, not in the file.** Each half is an ordinary ROS 2 node in whatever world its process environment describes: `RMW_IMPLEMENTATION`, `ROS_DOMAIN_ID`, and the RMW's own config file (`FASTRTPS_DEFAULT_PROFILES_FILE`, `CYCLONEDDS_URI`, `ZENOH_SESSION_CONFIG_URI`). The example varies only the domain, but the two halves can run different RMW implementations entirely; every ordered pair of fastrtps, cyclonedds, and zenoh is tested. The config file never mentions any of this, which is why changing the fleet network's RMW is a restart of the fleet half and nothing else.

**Both halves read the same file.** `ipc.dir` is where they meet: the control socket and the shared-memory channels live there. On one machine a shared path is enough; in containers, mount the same volume into both and share the IPC namespace (`ipc: service:<peer>` in compose).

**Only what you list exists.** Run `ROS_DOMAIN_ID=18 ros2 topic list` in the fleet world: you see `/my_robot/chatter` and the airlock's own node, and nothing else from the robot world. A direction is a deny-by-default allowlist; a topic not listed is absent from discovery entirely, not merely filtered.

**Data flows only while someone listens.** Stop terminal 4 and the robot-side subscription is torn down; the publisher in terminal 3 is back to zero matched subscribers and does no serialization work. The topic stays advertised in the fleet world the whole time. This lazy activation is the default; entries with latched data (`durability: transient_local`) should set `activation: always` so late joiners get history.

The `type` line is optional. Without it, the airlock discovers the type when a publisher appears, which is what glob entries like `/camera/*` rely on. With it, a publisher with a different type is rejected instead of bridged.

## Check the file before running it

```bash
ros2 run ros2_airlock airlock check airlock.yaml
```

Validation is strict and errors teach: unknown keys, a topic listed in both directions, or an unbounded egress QoS are all rejected with the rule and the fix. Exit code 0 means valid, so scripts and deploy pipelines can gate on it.

## Next

[Add a command topic](add-a-command-topic.md) opens the other direction. [Namespace a robot](namespace-a-robot.md) covers prefixes and TF. If a topic doesn't flow, [Debug a silent topic](debug-a-silent-topic.md) has the ladder.
