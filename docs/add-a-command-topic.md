# Add a command topic

Commands flow against the sensor stream: from the fleet world into the robot world. Add a second direction to the config from [Bridge your first topic](bridge-your-first-topic.md):

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

    - from: fleet
      to: robot
      namespace: ""
      topics:
        - topic: /my_robot/cmd_vel
          rename: /cmd_vel
          type: geometry_msgs/msg/TwistStamped
          qos: { reliability: best_effort, depth: 1 }
          max_hz: 50
          max_bytes: 4096
```

Restart the halves (or send both `SIGHUP` for a live reload), then drive from the fleet world and watch the robot world receive it:

```bash
# fleet world: publish a command on the robot's namespaced name
ROS_DOMAIN_ID=18 ros2 topic pub -r 5 /my_robot/cmd_vel geometry_msgs/msg/TwistStamped \
  '{twist: {linear: {x: 0.5}}}'

# robot world: the driver's unnamespaced topic
ROS_DOMAIN_ID=17 ros2 topic echo /cmd_vel geometry_msgs/msg/TwistStamped
```

## Reading the entry

**Names are source-world names.** `topic:` is the name in the direction's `from` world. Fleet participants publish on `/my_robot/cmd_vel` because that is the robot's public name out there; `rename: /cmd_vel` is what the robot's driver actually subscribes to. The inbound direction has `namespace: ""` because the robot world is not namespaced.

**Exact name, declared type.** Globs are allowed everywhere, but on a direction facing a network you don't fully control, list commands individually with their type. The attack surface stays enumerable, and a publisher offering a different type on the name is rejected instead of forwarded.

**Caps are the contract.** `max_hz` decimates, `max_bytes` drops oversized messages, and both are enforced in the sending half and enforced again on receipt. `depth: 1` with `best_effort` is the right shape for a command stream: a stale command is worse than a dropped one, and nothing inbound can queue up behind a slow consumer. Unbounded reliable history is not expressible in the config at all.

**One name, one direction.** The same topic name cannot appear in both directions; validation rejects it. That rule is what makes bridge loops structurally impossible.

If losing this command stream mid-motion matters, the production profile adds a `deadman` to the entry: the robot-side half publishes a configured neutral payload (zero velocity) when the stream dies. That and the audit trail are covered in [Harden — the airlock pattern](harden-the-airlock-pattern.md).
