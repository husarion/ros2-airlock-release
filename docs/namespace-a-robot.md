# Namespace a robot

One `namespace` per direction prefixes everything that direction republishes, so a robot whose internal graph is unnamespaced appears in the fleet world as `/panther_1234/...`:

```yaml
    - from: robot
      to: fleet
      namespace: /panther_1234
      topics:
        - topic: /*
          max_bytes: 10485760
          max_hz: 100
        - topic: /odometry/filtered
          rename: /odom
          max_hz: 50
      tf:
        namespaced: true
        merged: true
        shared_frames: []
```

Every topic, service, and action in the direction gets the prefix: `/scan` becomes `/panther_1234/scan`, `/hardware/e_stop_trigger` becomes `/panther_1234/hardware/e_stop_trigger`. `rename` composes with it: `/odometry/filtered` appears as `/panther_1234/odom`. Inside the robot world nothing changes; namespacing on one host buys nothing, so the robot graph stays plain and the identity lives entirely at the boundary.

This is how a fleet of identical robots shares one network: every robot runs the same internal graph with the same names, each robot's airlock applies its own serial as the namespace, and the fleet world sees `/panther_1234/scan` next to `/lynx_5678/scan` with no collisions.

## TF: two conventions, both supported

TF is where namespacing usually breaks, because frame names travel inside message payloads, not in topic names. The `tf` block bridges `/tf` and `/tf_static` in either or both of two conventions:

**`namespaced: true`** republishes the pair as `/panther_1234/tf` and `/panther_1234/tf_static` with the payload byte-identical: frames stay `base_link`, `odom`, robot-local. This is what per-robot consumers want. An Open-RMF fleet adapter, or any Nav2-style stack pointed at one robot, reads this pair and applies its own prefixing convention.

**`merged: true`** republishes into the fleet-global `/tf` and `/tf_static` with every `frame_id` and `child_frame_id` prefixed: `panther_1234/base_link`. Because a frame mentioned in TF must match the frame in sensor headers, merged mode also rewrites `header.frame_id` on every bridged stamped topic in the direction. This is the convention that lets one rviz session display N robots at once, each a distinct subtree of one tree: distinct topics, distinct TF subtrees, a RobotModel per robot.

`shared_frames` lists frames left unprefixed in merged mode. The moment your robots localize against a common map, set `shared_frames: [map]` and the subtrees join at the shared root.

Late joiners are handled: the airlock aggregates every static transform it has seen and re-latches the complete set, so an rviz started an hour after the robot still receives the full static tree.

**Do not mix the conventions in one consumer.** With both modes on, the direction publishes robot-local frames on the namespaced pair and prefixed frames on the merged pair. A consumer must pick one: either the raw pair plus its own prefixing, or merged `/tf` plus this direction's bridged topics. Mixing them makes TF lookups fail silently; `airlock doctor` prints a note whenever both modes are active.

Frame names inside other payloads (a URDF in `robot_description`, for example) are not rewritten; consumers of merged TF that also load the URDF should apply their own frame prefix there, which is standard rviz/state-publisher practice.
