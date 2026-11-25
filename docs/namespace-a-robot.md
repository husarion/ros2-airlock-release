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

## Changing the namespace

Since 1.30.1 a new namespace is a reload, not a restart. Write the file with the new `namespace:` and send SIGHUP to both halves; `airlock_half --plan` reads `sighup` for both worlds. The pair moves to the new file together, every name the direction republishes reappears under the new namespace (a latched topic such as the e-stop is delivered again to late joiners under the new name), and the old names leave the fleet world. Neither half restarts, so the robot world is not touched.

The file's `namespace:` is the only namespace a half reads. `ROS_NAMESPACE` in a half's environment does nothing (ROS 2 does not read it either), so an image need not set it and a change of it is no reason to restart a half. If you restart one half onto the new file instead of signalling it, the other half notices the new epoch when the restarted one connects and re-reads its own file once, as a SIGHUP would, so a shared file still converges without a second restart. Two halves whose files really differ still end in an exit: the half whose file on disk is not what it runs (a refused file, an edit nobody signalled) exits after 30 s so its restart picks the file up; a half whose file is what it runs waits longer (45 s for the connector, 60 s for the `ipc.listener` half), so only one half restarts and the robot's half goes last.

This includes entries marked `protected: true`. A reload normally refuses to move a protected entry's name, and a namespace change counts as a move only when something else changes with it. Two shapes pass:

- An entry of the direction whose namespace changed keeps its source name, its `rename` and its `absolute` setting. Its new name is its old name under the new namespace.
- An inbound entry (fleet to robot) whose source lives under the robot's namespace, written as `/panther_1234/autonomous/cmd_vel` with `rename: /autonomous/cmd_vel`, moves its source to `/<new namespace>/autonomous/cmd_vel` and keeps its `rename`. Its name in the robot world does not change.

The reload is still refused, and the halves keep the running file, when the same file also renames a protected entry, changes its type, removes its `protected` mark, drops it, or puts another entry on its new name. Protected services and the parts of a protected action follow the same rules.

## Renaming an action

An exact action entry takes `rename` the way a topic or a service does (1.30.0). The action appears as the namespace plus the new name, and its five parts follow it: the three services `<rename>/_action/send_goal`, `get_result` and `cancel_goal`, and the two topics `<rename>/_action/feedback` and `status`.

```yaml
      actions:
        - action: /navigate_to_pose
          type: nav2_msgs/action/NavigateToPose
          rename: /nav/goto
```

The fleet sends goals to `/panther_1234/nav/goto`, and `/panther_1234/navigate_to_pose` does not exist there. A renamed action whose name equals another action's, or whose part equals a service or topic entry's destination, is refused at load, and the message names the action. A glob cannot `rename`, because one name for many matches means nothing; it takes `rename_prefix` (below). A host client of the fleet half addresses the action by its new name.

## A name outside the namespace: `absolute`

`absolute: true` on a topic, service or action entry, exact or glob, keeps the direction's namespace off that entry's destination. The destination is then the `rename`, the prefix-mapped name, or the source name:

```yaml
      namespace: /panther_1234
      topics:
        - topic: /diagnostics_agg
          type: diagnostic_msgs/msg/DiagnosticArray
          absolute: true                # appears as /diagnostics_agg
        - topic: /status
          type: std_msgs/msg/String
          rename: /fleet/robot_status
          absolute: true                # appears as /fleet/robot_status
```

A leading slash in `rename` does not mean this: `rename: /odom` is still placed under the namespace. Every check that already exists runs on the resulting name: two entries of a direction with one destination are refused, and a service exposed in both directions, or one whose source is the other direction's exposed name, is refused (a topic loop is broken by the publisher identity check and needs no name rule). An absolute `/tf` or `/tf_static` is always refused, whether the tf block is on or not, because frames one robot writes there unprefixed break every other robot's tree; use `tf.merged`. `airlock check` prints a note (`rename.absolute_shared`) on every absolute entry: the name is the same on every robot rendered from one template, so two robots on one domain share it. Into a robot world with `namespace: ""` the key changes nothing. Switching it on reload moves the lane, so the lane is re-created, and a `protected` entry refuses the change.

## Renaming a subtree: `rename_prefix`

A glob takes `rename_prefix: {strip, add}`: a match `<strip><rest>` is exposed as the namespace (unless `absolute`) and then `<add><rest>`. Either part may be left out, not both.

```yaml
      namespace: /panther_1234
      topics:
        - topic: /sensors/**
          rename_prefix: {strip: /sensors, add: /payload}   # /sensors/scan -> /panther_1234/payload/scan
```

`strip` must be one or more whole segments at the start of the glob, before its first wildcard, so every match carries it; `/sens*/x` cannot strip `/sens`. `add` is a name prefix. The main use is the inbound direction: a fleet-side glob `/panther_1234/*` with `rename_prefix: {strip: /panther_1234}` reaches the robot's nodes under their bare names, which is what they call. An action glob maps the action's name, and the five parts follow.

What the engine refuses or skips:

- `rename` on a glob is refused at load, as before.
- Two globs of one direction whose destination prefixes can produce one name from different sources are refused at load (`rename.ambiguous`), because the receiving half could not tell which entry a name came from. Globs without a prefix map overlap as they always could.
- An exact entry wins on a shared source as well as on a shared destination. A match whose source name an exact entry of the direction also names crosses only by that entry, under the exact entry's destination, and the glob counts it as `exact` in its status row (topic globs from 1.30.1, as service and action globs always did). A match whose mapped name an exact entry of the direction holds, or another source's lane already carries, is skipped and counted as `dest_taken`; the exact entry keeps the name.
- A match whose mapped name is not a valid ROS name or is longer than 255 bytes is skipped as `dest_invalid`, never truncated.

`airlock check --plan` prints each entry's destination name or pattern when it uses one of these keys, and `airlock doctor` names `exact`, `dest_taken` and `dest_invalid` skips with their glob. The receiving half's lane rows carry `source`, the name the lane came from. Every half that reads a config with one of the three keys must be 1.30.0 or later; an older half refuses the file as an unknown key.

## TF: two conventions, both supported

TF is where namespacing usually breaks, because frame names travel inside message payloads, not in topic names. The `tf` block bridges `/tf` and `/tf_static` in either or both of two conventions:

**`namespaced: true`** republishes the pair as `/panther_1234/tf` and `/panther_1234/tf_static` with the payload byte-identical: frames stay `base_link`, `odom`, robot-local. This is what per-robot consumers want. An Open-RMF fleet adapter, or any Nav2-style stack pointed at one robot, reads this pair and applies its own prefixing convention.

**`merged: true`** republishes into the fleet-global `/tf` and `/tf_static` with every `frame_id` and `child_frame_id` prefixed: `panther_1234/base_link`. Because a frame mentioned in TF must match the frame in sensor headers, merged mode also rewrites `header.frame_id` on every bridged stamped topic in the direction. This is the convention that lets one rviz session display N robots at once, each a distinct subtree of one tree: distinct topics, distinct TF subtrees, a RobotModel per robot.

When only a few lanes need the merged-frame convention, put `frame_prefix: true` on those topic entries instead of enabling `merged` (schema 3). It runs the identical header rewrite — `header.frame_id` plus `child_frame_id` where the type has one, `shared_frames` honored, empty frames untouched — on that lane alone, while every other topic of the direction keeps byte-identical passthrough. The canonical case is a fleet-side EKF that fuses the robot's odometry and IMU: two `frame_prefix` lanes give it frames matching the merged tree without rewriting twenty other stamped topics.

`shared_frames` lists frames left unprefixed in merged mode. The moment your robots localize against a common map, set `shared_frames: [map]` and the subtrees join at the shared root.

Late joiners are handled: the airlock aggregates every static transform it has seen and re-latches the complete set, so an rviz started an hour after the robot still receives the full static tree.

`max_bytes` caps both tf lanes, like an entry's `max_bytes` (default 65536, about 500 transforms). A `/tf` message above it is dropped where it is taken, counted in the lane's `dropped_size`, and never crosses. A `/tf_static` aggregate above it is not shipped: the lane's row reads `tf_static_over_cap: true`, the half logs it, and the previous aggregate stays latched in the far world. The status rows' `max_frame_bytes` say what your tree really sends; raise `tf: {max_bytes: ...}` when it is near the cap. At 64 KiB or less the tf lanes can run at real-time priority (see [threads and pinning](threads-and-pinning.md)).

### Forward only some frames: `frames` and `frame_max_hz`

`frames` narrows what the tf block forwards. List the frames the far world needs, for example `frames: [base_link]`, and the airlock forwards the transform of each listed frame and of every frame above it, up to the root, so the forwarded tree is always connected. Everything else stays in the robot world. It applies to `/tf` and `/tf_static`, in `namespaced` and in `merged` mode. Leave the key out and every frame is forwarded, as before.

```yaml
tf:
  merged: true
  shared_frames: [map]
  frames: [base_link]
  frame_max_hz: 10
```

The airlock learns which frames sit above a listed one from the transforms it sees, so a frame that gains a parent later (a localizer that starts publishing `map -> odom`) is forwarded from that moment, and a static transform that arrived before it became an ancestor is latched as soon as it is one. Write frame names as the robot knows them, without the namespace or a prefix. The `/tf` and `/tf_static` rows of the robot half's status list the frames that cross (`tf_filter.forwarded`) and count what was left out (`filtered`, `filtered_msgs`); a listed frame that no transform names 30 s after the lanes first carried is listed as `unseen`, and doctor names it (`tf.frame_unseen`), which usually means a typo. At most 64 names, no globs.

`frame_max_hz` caps each child frame's dynamic transforms on each tf lane, so `base_link` from a 50 Hz EKF crosses at 10 Hz while a 5 Hz frame keeps its 5 Hz. It applies with or without `frames`; `/tf_static` is not limited; 0 means no cap. Dropped transforms are counted as `rate_dropped`.

A filter saves bandwidth; it is not a security boundary. Topics whose headers name a frame you filtered out still cross, and a viewer on the far side cannot place them: forward the frames your bridged sensor topics use, or do not bridge those topics (doctor prints `tf.filter_stamped` for a merged direction that has both). A robot model in rviz needs every link, so a filtered robot shows as its listed frames only. With `frames` set the namespaced pair carries only the kept transforms, with their frame names unchanged.

### Choose the prefix and rename frames: `frame_prefix` and `frame_rename`

`frame_prefix` sets the text written before every rewritten frame name, exactly as you give it: `frame_prefix: "ursa/"` or `"ursa_"`. Without it the prefix is the namespace followed by `/`, as before. `frame_prefix: ""` writes no prefix at all; `airlock check` then notes `tf.empty_prefix`, because two robots on one domain would share frame names. With `frame_prefix` set, merged mode and a topic's `frame_prefix: true` work with `namespace: ""` too; both tf modes with no namespace are refused, since they would publish the same `/tf`. The topic entry's `frame_prefix: true` stays a boolean; the string belongs in the `tf` block.

`frame_rename` renames single frames: `frame_rename: {base_link: base}` writes `base` (then the prefix) wherever the direction rewrites frames, that is the merged `/tf` and `/tf_static`, the headers of the direction's stamped topics in merged mode, and every `frame_prefix: true` lane. A frame in `shared_frames` is renamed and not prefixed. The `namespaced` pair is never renamed: a consumer that wants renamed frames reads the merged pair (with `frame_prefix: ""` if it wants no prefix). Two frames renamed to one name, and a new name that is also a key, are refused. The rewritten rows carry `frame_prefix` and `frames_renamed`; a key no frame met 30 s after the lanes carried is `tf.rename_unseen`, and a new name that a frame of the robot already uses is `tf.rename_collision` in doctor.

The order is fixed: on the robot half the filter, then the cap, both on the robot's own frame names; on the far half the rename, then the prefix. So `frames`, `shared_frames` and the keys of `frame_rename` always name frames as the robot knows them; `check` warns about a `frames` entry that names a new name (`tf.frames_renamed_name`) or already carries the prefix (`tf.frames_prefixed`).

**Do not mix the conventions in one consumer.** With both modes on, the direction publishes robot-local frames on the namespaced pair and prefixed frames on the merged pair. A consumer must pick one: either the raw pair plus its own prefixing, or merged `/tf` plus this direction's bridged topics. Mixing them makes TF lookups fail silently; `airlock doctor` prints a note whenever both modes are active.

Frame names inside other payloads (a URDF in `robot_description`, for example) are not rewritten; consumers of merged TF that also load the URDF should apply their own frame prefix there, which is standard rviz/state-publisher practice.
