# The check plan contract

`ros2 airlock check <file> --plan --format json` prints what a configuration file will do as one JSON document: whether the file is valid, and for every entry its class, the reason for it, its bandwidth cap and its real-time policy. A configuration tool (Cockpit's pre-apply view, a CI job) reads it instead of parsing the text form. It is a versioned contract since 1.26.0, check plan schema 1, with the shape in `schema/check_plan.schema.json`.

```bash
ros2 airlock check airlock.yaml --plan --format json      # the check and every entry
ros2 airlock check airlock.yaml --format json             # the check alone, no entries
ros2 airlock check airlock.yaml --world fleet --format json   # the check for the fleet half
```

`--world <world>` names the half the check is for. It matters on a Humble image only: there `api.actions` is refused (an error) in the named world's `api:` block, and every other world's block, or every block when no `--world` is given, is reported as the note `api.actions_humble`, because a Humble half refuses its own world's block when it starts while a Jazzy or Lyrical half of the other world serves its block. A `--world` that is no world of the file is an error.

The engine prints the document itself (`airlock_half --check --classes --format json -c <file>`), with the classifier the running halves use, so the values are the ones the status rows carry for the same file: `class`, `class_reason` and `cap_bytes_per_s` on the `tx` and `rx` rows of both halves, and `rt` on the destination half's `rx` rows (see the [status and health reference](status-and-health-reference.md)). A view made before applying a file and the view of the running pair agree. The one value only a running half can show is a live promotion (`class_reason: observed`), and the one `rt` value only a running half can earn is `refused`.

## Exit codes

The document is the whole of stdout; the exit code is `airlock check`'s: `0` valid, `1` a semantic error, `2` a schema or parse error, `3` valid with warnings. A file that does not load still gets a document, with `verdict: refused` and the loader's message in `errors`, and no `bulk`, `rt`, `standing` or `entries`.

## An example

The file of [bounded bulk lanes](bounded-bulk-lanes.md), planned; the notes are shortened and three of its eight entries are shown:

```json
{
  "schema": 1,
  "engine_version": "1.27.1",
  "config_schema": 9,
  "file": "airlock.yaml",
  "verdict": "ok",
  "fragments": [],
  "errors": [],
  "warnings": [],
  "notes": [
    "standing set: 18.5 MiB of ipc.shm_budget 512.0 MiB reserved at load for 8 exact entries at queue_depth 2; largest: \u2026",
    "standing cost per service and action entry: service '/hardware/e_stop_trigger' 0.1 MiB"
  ],
  "bulk": {
    "max_bytes_per_s": 4194304,
    "threshold_bytes": 262144,
    "threads": 1,
    "type_files": []
  },
  "rt": {
    "requested": false,
    "pump_fifo_priority": 0,
    "pump_batch": 8,
    "max_frame_bytes": 65536
  },
  "standing": {
    "lanes": 8,
    "shm_bytes": 19351908,
    "shm_budget": 536870912
  },
  "entries": [
    {
      "direction": {
        "from": "robot",
        "to": "fleet"
      },
      "kind": "topic",
      "name": "/camera/depth/points",
      "source": "/camera/depth/points",
      "destination": "/lynx_0001/camera/depth/points",
      "type": "sensor_msgs/msg/PointCloud2",
      "part_of": null,
      "glob": false,
      "per_lane": false,
      "class": "bulk",
      "class_reason": "max_bytes",
      "cap_bytes_per_s": 16777216,
      "cap_source": "entry",
      "promote_at_bytes": null,
      "rt": null,
      "rt_reason": "not_requested",
      "activation": "lazy",
      "standing": false,
      "max_bytes": 2097152,
      "max_hz": null,
      "protected": false,
      "lanes": 1
    },
    {
      "direction": {
        "from": "robot",
        "to": "fleet"
      },
      "kind": "tf",
      "name": "/tf",
      "source": "/tf",
      "destination": "/lynx_0001/tf",
      "type": "tf2_msgs/msg/TFMessage",
      "part_of": null,
      "glob": false,
      "per_lane": false,
      "class": "control",
      "class_reason": "tf",
      "cap_bytes_per_s": 0,
      "cap_source": "none",
      "promote_at_bytes": null,
      "rt": null,
      "rt_reason": "not_requested",
      "activation": "lazy",
      "standing": false,
      "max_bytes": 65536,
      "max_hz": null,
      "protected": false,
      "lanes": 1
    },
    {
      "direction": {
        "from": "robot",
        "to": "fleet"
      },
      "kind": "service",
      "name": "/hardware/e_stop_trigger",
      "source": "/hardware/e_stop_trigger",
      "destination": "/lynx_0001/hardware/e_stop_trigger",
      "type": "std_srvs/srv/Trigger",
      "part_of": null,
      "glob": false,
      "per_lane": false,
      "class": "control",
      "class_reason": "service",
      "cap_bytes_per_s": 0,
      "cap_source": "none",
      "promote_at_bytes": null,
      "rt": null,
      "rt_reason": null,
      "activation": null,
      "standing": false,
      "max_bytes": null,
      "max_hz": null,
      "protected": false,
      "lanes": 1,
      "timeout_s": 2,
      "max_request_bytes": 1024,
      "max_response_bytes": 1024,
      "max_inflight": 8,
      "pinned": false
    }
  ]
}
```

## The rule

- A key is never removed or re-typed within a schema number.
- Adding a key bumps nothing: a reader ignores keys it does not know.
- A change of a key's meaning bumps `schema`, announced in the CHANGELOG.
- A value list below may grow in a later release, announced in the CHANGELOG first; a reader that meets a value it does not know shows it as it is and treats the entry as unknown.

## The keys

| Key | Type | Meaning |
| -- | -- | -- |
| `schema` | integer | This contract's version, 1 (`airlock_half --constants` prints it as `check_plan_schema`). |
| `engine_version` | string | The engine that judged the file. |
| `config_schema` | integer | The one config schema this engine reads; a file at another one is refused, and `errors` names `ros2 airlock migrate`. |
| `file` | string | The file as named on the command line. |
| `verdict` | string | `ok`, `warnings` or `refused`, the exit code's three classes. |
| `fragments` | array | The include directory's fragments by name, in reading order. |
| `errors` | array | What refuses the file, one finding each. |
| `warnings` | array | The loader's and the check's warnings. |
| `notes` | array | Informational findings: the standing set, each glob's cost, standing bulk lanes. They never change the exit code. |
| `bulk` | object | The bulk settings as the file resolves them. |
| `bulk.max_bytes_per_s` | integer | `airlock.bulk_max_bytes_per_s`, the cap of a bulk lane without its own. |
| `bulk.threshold_bytes` | integer | `airlock.bulk_threshold_bytes`. |
| `bulk.threads` | integer | `airlock.bulk_threads`. |
| `bulk.type_files` | array | The bulk type list files read, in order. |
| `rt` | object | `airlock.rt` as the file resolves it. |
| `rt.requested` | boolean | Whether the file sets `pump_fifo_priority`. |
| `rt.pump_fifo_priority` | integer | The SCHED_FIFO priority of the control pumps, 0 when not requested. |
| `rt.pump_batch` | integer | Frames a real-time pump takes per wake. |
| `rt.max_frame_bytes` | integer | The largest `max_bytes` a real-time pump serves. |
| `standing` | object | What the file reserves at load. |
| `standing.lanes` | integer | The exact entries' lanes. |
| `standing.shm_bytes` | integer | The shared memory they reserve. |
| `standing.shm_budget` | integer | `ipc.shm_budget`. |
| `entries` | array | With `--plan` only: one record per line of the text form, in its order. |
| `entries[].direction` | object | The direction the entry is in. |
| `entries[].direction.from` | string | Its source world. |
| `entries[].direction.to` | string | Its destination world. |
| `entries[].kind` | string | `topic`, `tf` (a lane of the direction's tf block), `service` or `action`. |
| `entries[].name` | string | As written; a glob stays a pattern. |
| `entries[].source` | string | The name in the source world. |
| `entries[].destination` | string | The name in the destination world, namespace and rename applied. |
| `entries[].type` | string | The interface type; empty for a glob, whose types are discovered. |
| `entries[].part_of` | string or null | An action's five parts (two topics, three services) are records of their own that name the action here; null for an entry the file wrote. Count the records without it to count what the file wrote. |
| `entries[].glob` | boolean | The entry is a pattern. |
| `entries[].per_lane` | boolean | A topic glob: each of its lanes is classed when it is created, so its `class`, `class_reason` and `cap_bytes_per_s` are null. |
| `entries[].class` | string or null | The class the lane runs at. |
| `entries[].class_reason` | string or null | Why it has that class. |
| `entries[].cap_bytes_per_s` | integer or null | The effective byte cap per second, 0 for none (a service or action always 0). |
| `entries[].cap_if_bulk_bytes_per_s` | integer | A topic glob only: the cap a lane of it runs at if it is created bulk, the entry's own `max_bytes_per_s` or else `airlock.bulk_max_bytes_per_s`. |
| `entries[].cap_source` | string | Where the cap comes from. |
| `entries[].promote_at_bytes` | integer or null | A normal lane the rules classed (not `class: normal` written out): the sample size past which a running half moves it to bulk. |
| `entries[].rt` | string or null | The destination pump's real-time policy for a topic lane. |
| `entries[].rt_reason` | string or null | The verdict behind `rt`; null for a topic glob, a service or an action. |
| `entries[].activation` | string or null | A topic's `activation` (`lazy` or `always`); null for a service or action. |
| `entries[].standing` | boolean | A cost from the start: a topic with `activation: always`, or a service or action whose channels open at advertise. |
| `entries[].max_bytes` | integer or null | A topic's `max_bytes`. |
| `entries[].max_hz` | number or null | A topic's `max_hz` when the entry sets one. |
| `entries[].protected` | boolean | The entry is protected. |
| `entries[].lanes` | integer | Lanes toward `airlock.max_lanes`: 1, or 5 for an action. |
| `entries[].timeout_s` | number | Services and actions: the call timeout. |
| `entries[].max_request_bytes` | integer | Services and actions: the request cap (an action's goal and result requests take 65536). |
| `entries[].max_response_bytes` | integer | Services and actions: the response cap. |
| `entries[].max_inflight` | integer | Services and actions: requests in flight. |
| `entries[].pinned` | boolean | Services and actions: the channels open at advertise (a protected entry, or a timeout under 0.5 s); a glob's never do. |
| `watchable` | array | With `--plan` only (1.28.0): every name a host `watch` may follow, one per `api.retain` name of a world's `api:` block, in the block's order. |
| `watchable[].world` | string | The world whose half serves the watch. |
| `watchable[].topic` | string | The name, as `api.retain` lists it. |
| `watchable[].origin` | string | `wall` (a lane the wall carries from that world or delivers into it) or `local` (an `api.local.topics` name the wall does not carry). |
| `watchable[].type` | string or null | The type as the file states it; null for an entry that states none (the half learns it when the lane exists). |
| `watchable[].max_hz` | integer | `api.watch.max_hz`: updates a second per watched name at most. |
| `watchable[].keepalive_s` | number | `api.watch.keepalive_s`: a keepalive after this long without a line. |
| `watchable[].max_watchers` | integer | `api.max_watchers`: the watch streams the half serves at once, one of them reserved for uid 0. |
| `action_targets` | array | With `--plan` only (1.28.0): every action a host goal may drive, one per `api.actions` entry of a world's `api:` block, in the block's order. |
| `action_targets[].world` | string | The world whose half serves the goal. |
| `action_targets[].action` | string | The action's name in that world. |
| `action_targets[].origin` | string | `wall` (an exact action entry of a direction from that world) or `local` (an action the wall does not carry). |
| `action_targets[].type` | string or null | `pkg/action/Name`: the local entry's, or the wall entry's. |
| `action_targets[].uids` | array | The uids that may drive it: 0 first, then the entry's `allow_uids`. |
| `action_targets[].fixed_goal` | object or null | The entry's fixed `goal`; null when it has none. |
| `action_targets[].goal_constraints` | object | The entry's `goal_constraints`, field path to value; empty when it has none. |
| `action_targets[].max_duration_s` | number | When a host goal is cancelled and its stream ends `timeout`. |
| `action_targets[].accept_timeout_s` | number | The server's goal response within this. |
| `action_targets[].timeout_s` | number | A cancel's response, and the result after a terminal status, within this. |
| `action_targets[].lost_after_s` | number | A goal with no server, or absent from its status, this long is lost. |
| `action_targets[].feedback_max_hz` | integer | `api.actions_feedback_max_hz` of the block. |
| `action_targets[].max_goals` | integer | `api.max_goals` of the block. |

## The value lists

The verdict:

```yaml
# check_plan: verdict
- ok
- warnings
- refused
```

An entry's kind:

```yaml
# check_plan: entries[].kind
- topic
- tf
- service
- action
```

Its class (null for a topic glob):

```yaml
# check_plan: entries[].class
- control
- normal
- bulk
- null
```

Why it has its class: the status's reasons except `observed`, which only a running lane earns (null for a topic glob):

```yaml
# check_plan: entries[].class_reason
- explicit
- tf
- protected
- service
- type
- max_bytes
- default
- null
```

Where the cap comes from: the entry's own `max_bytes_per_s`, the bulk default (a bulk lane, or a lane of a topic glob without its own cap), or none:

```yaml
# check_plan: entries[].cap_source
- entry
- default
- none
```

The real-time policy, as the status's `rx[].rt` spells it: `fifo:<n>` (the pump runs at SCHED_FIFO n), `ineligible:max_bytes` (a control lane above `rt.max_frame_bytes`), or null (no `airlock.rt`, not a control lane, or no topic lane):

```yaml
# check_plan: entries[].rt
- fifo:<n>
- ineligible:max_bytes
- null
```

The verdict behind it:

```yaml
# check_plan: entries[].rt_reason
- not_requested
- eligible
- not_control
- ineligible:max_bytes
- null
```

Where a watchable name comes from:

```yaml
# check_plan: watchable[].origin
- wall
- local
```

```yaml
# check_plan: action_targets[].origin
- wall
- local
```
