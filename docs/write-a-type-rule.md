# Write a type rule

A lane is refused because the two halves hold different definitions of a type and the airlock will not guess how to map them. You tell it, in a small YAML file both halves reload. Start from what the doctor prints for such a lane, here a service of your own whose request gained a field on the robot's newer release:

```text
FAIL [fleet] service /my_robot/dock: my_msgs/srv/Dock_Request is refused (rules_required): peer RIHS01_4f21c0d… vs this image RIHS01_9a7e33b…. the definitions of my_msgs/srv/Dock_Request differ and the entry converts only under an explicit rule; paste the rule below into the config directory's type_map.d/ and reload (airlock check --rules validates it). The plan would: drops approach_speed. [unprotected]
      paste into <config dir>/type_map.d/<name>.yaml and reload:
        schema: 1
        rules:
          - type: my_msgs/srv/Dock_Request
            from: { release: jazzy }
            to: { release: humble }
            map:
              - { field: approach_speed, allow: drop }
```

The doctor has done the analysis: the Jazzy request carries `approach_speed`, the Humble one does not, and a request crossing from the fleet to the robot would lose it. Topics that only gained or lost a field convert without asking. A service does not, because a request that silently loses a parameter is worse than a service that is visibly missing, so the airlock asks you to acknowledge each such field once. Save the block as `type_map.d/dock.yaml` next to `airlock.yaml`, check it, and reload both halves:

```bash
ros2 run ros2_airlock airlock check --rules -c airlock.yaml
kill -HUP <pid of each airlock_half>          # or restart the containers
```

The doctor now lists `/my_robot/dock` under the converted lanes, and calls from the fleet reach the robot with `approach_speed` dropped and the response converted back on the way out. Nothing else changed: the rule is a plain file, the config is untouched, and the halves kept running through the reload.

Every lane refused for this reason can be written out at once, with `airlock rules init -c airlock.yaml`: it reads both halves' status files and writes `type_map.d/generated.yaml` with one block per refused lane, for you to review before reloading.

## What a rule can say

A rule is keyed by the type and the two releases it maps between, and its `map` is a list of clauses from a closed set:

```yaml
schema: 1
rules:
  - type: controller_manager_msgs/srv/SwitchController_Request
    from: { release: humble }
    to: { release: jazzy }
    map:
      - { from: start_controllers, to: activate_controllers }     # a rename
      - { from: stop_controllers, to: deactivate_controllers }
      - { field: start_asap, allow: drop }                        # exists only on the source side
      - { field: activate_asap, allow: default }                  # exists only on the destination side
  - type: lifecycle_msgs/msg/TransitionEvent
    from: { release: jazzy }
    to: { release: lyrical }
    map:
      - { from: timestamp, to: stamp, convert: nanoseconds_to_time }   # a rename with a conversion
```

- **`from` / `to`**: a rename. Dotted paths reach into nested messages (`header.stamp`); a rule never indexes into an array.
- **`allow: drop | default | widen`** on a `field`: the acknowledgement a service or protected topic needs for a field that exists on one side only, or that widened (an `int32` that became `int64`).
- **`convert`**: `nanoseconds_to_time`, `time_to_nanoseconds`, `scale: <factor>`, or `const: <value>` for a destination field with no source. A value that does not fit the destination drops that one message and counts it in the lane's status; nothing is ever clamped.

That is the whole language. There are no expressions, no scripts and no references between rules, so a rule file can be read at a glance and validated completely before it runs. A rule whose clauses are all reversible (renames, the two time converters, `scale`) gets its inverse generated for the other direction, so one rule covers a message going either way.

`from` and `to` name releases, and a release name is resolved to the actual definition the other half holds when the two halves meet. If the peer turns out to hold a different definition than the one the rule was written against (a newer package version on the same release, say), the rule is reported inert with the reason and is never applied to the wrong layout. You can pin a rule to an exact definition instead by giving `hash:` in place of `release:`; the hashes are in the doctor's line.

## Checking before reloading

```bash
ros2 run ros2_airlock airlock check --rules -c airlock.yaml --peer-catalog peer_catalog.json
```

`check --rules` validates your `type_map.d/` against the config and this image's type catalog: unknown keys, a clause naming a field the type does not have, a `convert` whose types do not fit, and two files defining the same rule are all errors, with the file and line. Give it the other image's catalog (`docker cp <container>:/opt/ros2_airlock/share/ros2_airlock/type_catalog.json peer_catalog.json`) and it also resolves the peer's release names and checks each rule against both definitions; without it, a rule it cannot verify against the peer is a warning, exit code 3.

Two things a reload can refuse, both leaving the previous rule set running and the reason in the doctor's output:

- A file with any error is rejected as a whole. Your rules layer is all or nothing; the rules shipped in the image stay in force either way.
- A rule that would break a running `protected` lane is rejected with the lane named. To replace a shipped rule that serves a protected lane, set `override: true` on yours; a replacement of any other shipped rule is applied and reported as shadowing it.

Both halves must see the same files, mounted at the same path. If they do not, the doctor says so (`the two halves loaded DIFFERENT type-rule files`) and the half that plans conversions uses its own copy.

## When a rule is not the answer

- **`unmappable`**: a field changed its type or a fixed array changed size. No rule bridges that; rebuild one side's package so the two definitions agree, or leave the lane out.
- **`convert_denied`**: the entry says `convert: deny`. That was a decision; either change it to `explicit` and write the rule, or keep the lane refused.
- **`too_large`**: the converted message cannot fit the entry's `max_bytes`. Raise it.
- **A type missing on one image**: not a rule problem at all. The doctor prints which image lacks the package; add it to that image (the overlay pattern in [Check your own types before you deploy](check-your-types-before-you-deploy.md)).

## Next

[Check your own types before you deploy](check-your-types-before-you-deploy.md) runs this analysis over two whole images at once, so the rules are written before the robot ever sees the pair.
