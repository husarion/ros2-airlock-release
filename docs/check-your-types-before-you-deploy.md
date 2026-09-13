# Check your own types before you deploy

Before a robot on one ROS 2 release meets a fleet on another, find out what will cross untouched, what will be converted, what needs a rule and what cannot be bridged at all, with nothing running on a robot. On a machine with Docker and the airlock installed:

```bash
ros2 run ros2_airlock airlock drift husarion/ros2-airlock:1.19.0 husarion/ros2-airlock:1.19.0-humble -c airlock.yaml --markdown drift.md
```

It pulls every message, service and action definition out of both images, converts sample messages of every type that differs in both directions, checks each result on the destination image with that release's own generated code, and writes one report. The first line is the verdict:

```text
Verdict: PASS. 513 types share a definition, 50 differ, 82 exist only in jazzy, 0 only in humble. On the humble image 315 converted frames were accepted by the generated deserializer (3215 field values compared); on the jazzy image 315 (3527). Rules keyed on the pair: 10 applicable, 0 never fired.
```

The exit code is 0 on PASS, so a deploy pipeline can gate on it. The rest of the report is six sections, each answering one question.

## Reading the report

**Drifted types.** One row per type whose definition differs, with what happens in each direction:

| Type | jazzy -> humble | humble -> jazzy | Verified |
| -- | -- | -- | -- |
| `sensor_msgs/msg/Range` | drops variance | defaults variance | 7 + 7 frames |
| `lifecycle_msgs/msg/TransitionEvent` | copies (rule lifecycle.yaml:rules[0]) | copies (rule lifecycle.yaml:rules[0] (derived inverse)) | 7 + 4 frames |
| `nav2_msgs/action/BackUp_Goal` | defaults disable_collision_checks — automatic only; a service or protected lane needs a rule | drops disable_collision_checks — automatic only; … | 7 + 7 frames |

- `drops X`: the field exists only on the source side and is left behind. `defaults X`: the field exists only on the destination side and is filled with its default. `widens X`: an integer grew. `copies`: every field maps, under the named rule (a rename, or a unit conversion).
- `(rule …)`: a rule shipped in the image or found in your `type_map.d/` covers it.
- `automatic only; a service or protected lane needs a rule`: on a topic this converts by itself; used in a service, an action, or a topic marked `protected`, the same type is refused until a rule acknowledges the field. The next section lists exactly those.
- `Verified`: how many sample messages were converted and accepted by the other release's code. A type that failed there shows `FAILED` and the verdict is FAIL.

**Types that need a rule on a service or protected lane.** Every type from the table above whose conversion is automatic on topics but needs an acknowledgement elsewhere, each with the ready-to-paste rule. Copy the ones your config uses in a service or protected entry into `type_map.d/`, as [Write a type rule](write-a-type-rule.md) shows.

**Types no rule can bridge.** A field changed its type or a fixed-size array changed its length. The airlock refuses such a lane, by name, wherever it appears. If your config needs one of these, rebuild one side's package so the two definitions agree.

**Semantic drift.** Types whose layout is identical but whose constants or defaults changed between the releases, which the hash cannot see. These cross as `same`, so the report points them out for you to judge: `NavSatStatus`, for instance, defaults its `status` to a new `STATUS_UNKNOWN` on Jazzy where Humble meant `FIX`. A type here fails the verdict until you list it in an allowlist file (`--allowlist my_allowlist.yaml`, a YAML with a `semantic_drift:` list of type names) to record that you looked.

**Your config, both ways.** With `-c airlock.yaml` the report ends with every lane of your config planned twice, once with the robot on each image, and the roll-up: how many are the same, converted, refused, or name a type missing on one image, and the worst-case memory the converting half needs. Entries without a `type:` (glob entries, or ones left to discovery) are not planned unless you tell the tool their types: `--types types.yaml`, a map from topic name to type.

**Failures.** Empty on a PASS.

## Your own types, one overlay per release

The airlock's generic publishers and clients load a type's support library when they create an entity, so a type the image does not contain is a topic or service that never appears in the other world: skipped with a warning naming the remedy, never an error. The remedy is an image that contains it. `docker/Dockerfile.ugv` in the airlock repository is the pattern: `FROM` one of the airlock images, `apt install` or build your interface packages, and assert they load.

A type is a different thing on each release, so build one overlay per release you deploy and run the drift report on the pair of overlays:

```bash
docker build -f Dockerfile.ugv --build-arg BASE=husarion/ros2-airlock:1.19.0 -t my-airlock:jazzy .
docker build -f Dockerfile.ugv --build-arg BASE=husarion/ros2-airlock:1.19.0-humble -t my-airlock:humble .
ros2 run ros2_airlock airlock drift my-airlock:jazzy my-airlock:humble -c airlock.yaml --markdown drift.md
```

The report now includes your packages. A type present in one overlay and absent from the other is listed as existing only on one image, and the doctor on a live pair will say `not installed in the fleet image` rather than treat it as drift.

Rerun the report whenever either image changes: a new airlock version, a new version of one of your packages, or a third release joining the fleet. On a same-release pair it is also the check for a package that moved version between the robot and the fleet image, which is the one kind of drift that does not need two releases to happen.

## Next

[Write a type rule](write-a-type-rule.md) for the lanes the report says need one. [Bridge two ROS 2 releases](bridge-two-ros2-releases.md) runs the pair.
