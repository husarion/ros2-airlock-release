# The constants contract

`airlock_half --constants` prints the numbers and versions a tool needs to work with an engine without guessing: the shared-memory budget formulas, the schemas the engine reads and writes, its protocol, its thread-name scheme, the bulk and real-time defaults, and where its console log goes. Since 1.25.0 this output is a versioned contract, currently `schema: 3`, with the shape in `schema/constants.schema.json`.

## Reading it without running the engine

Every image carries the same bytes as a file and names the contract in two labels, so a build recipe or a dashboard learns it from the image manifest and reads the values without a ROS environment:

| Label | Value |
| -- | -- |
| `com.husarion.airlock.constants.schema` | the contract version, `3` |
| `com.husarion.airlock.constants.path` | `/opt/ros2_airlock/share/ros2_airlock/constants.json` |

```bash
docker inspect -f '{{index .Config.Labels "com.husarion.airlock.constants.schema"}}' husarion/ros2-airlock:1.25.0-jazzy
id=$(docker create husarion/ros2-airlock:1.25.0-jazzy)
docker cp "$id:/opt/ros2_airlock/share/ros2_airlock/constants.json" - | tar -xO
docker rm "$id"
```

The image build refuses a file that differs from the binary's output or that says another schema than the label.

## The rule

- A key is never removed or re-typed within a schema number.
- Adding a key bumps nothing: a reader ignores keys it does not know.
- A change of a key's meaning bumps `schema`, announced in the CHANGELOG.

## The keys

| Key | Type | Meaning |
| -- | -- | -- |
| `schema` | integer | This contract's version, 3. |
| `engine_version` | string | The engine's version, for example `1.25.0`. |
| `status_schema` | integer | The `schema` of every `status_<world>.json` the engine writes (see the [status and health reference](status-and-health-reference.md)). |
| `config_schema` | integer | The one config `schema:` the engine reads; an older file goes through `airlock migrate`. |
| `proto` | object | The control-plane protocol. |
| `proto.major` | integer | Its major version. |
| `proto.minor` | integer | The minor this engine speaks. |
| `proto.floor` | integer | The lowest peer minor it accepts; a peer below it is refused (`peer.refused: proto_minor`). |
| `log_streams` | object | Where a running half's console lines go. |
| `log_streams.default` | string | `split`: the severities in `log_streams.stdout` on stdout, those in `log_streams.stderr` on stderr. |
| `log_streams.modes` | array | `split` and `stderr`; `airlock_half --log-streams stderr` writes every line to stderr. |
| `log_streams.stdout` | array | `debug`, `info`. |
| `log_streams.stderr` | array | `warn`, `error`, `fatal`. |
| `log_streams.level_flag` | string | `--ros-args --log-level`: the flag that sets the level, after the half's own arguments. |
| `k_corr` | integer | Bytes of correlation id in a service frame. |
| `k_hdr` | integer | Bytes of frame header in every channel slot. |
| `slots_extra` | integer | The slots iceoryx2 adds to `queue_depth` for the engine's settings. |
| `slots_per_channel` | string | The slot formula, as text. |
| `channel_overhead_bytes` | integer | The fixed bytes per channel beyond its slots (measured). |
| `channel_overhead_fds` | integer | The file descriptors per channel. |
| `channel_overhead_measured` | string | Where and how that overhead was measured. |
| `topic_channel` | string | A topic lane's shared-memory formula. |
| `service_lane` | string | A service lane's shared-memory formula. |
| `service_slots` | string | A service lane's slot formula. |
| `max_inflight_default` | integer | A service entry's `max_inflight` when it sets none. |
| `lanes` | object | Lanes per exact entry toward `airlock.max_lanes`. |
| `lanes.topic` | integer | 1. |
| `lanes.service` | integer | 1. |
| `lanes.action` | integer | 5 (three services and two topics). |
| `thread_names` | integer | The thread-name scheme's version, 1 (see [threads and pinning](threads-and-pinning.md)). |
| `check_plan_schema` | integer | The `schema` of the document `airlock check <file> --format json` prints, 1 since 1.26.0 (see [the check plan contract](check-plan-contract.md)). |
| `bulk` | object | The bulk-lane defaults (see [bounded bulk lanes](bounded-bulk-lanes.md)). |
| `bulk.max_bytes_per_s` | integer | `airlock.bulk_max_bytes_per_s` when the file sets none. |
| `bulk.threshold_bytes` | integer | `airlock.bulk_threshold_bytes` when the file sets none. |
| `bulk.threads` | integer | `airlock.bulk_threads` when the file sets none. |
| `bulk.entry_max_bytes_per_s_min` | integer | The smallest non-zero `max_bytes_per_s` an entry may set. |
| `rt` | object | The real-time pump bounds. |
| `rt.max_priority` | integer | The highest `airlock.rt.pump_fifo_priority` the engine accepts. |
| `rt.default_batch` | integer | `airlock.rt.pump_batch` when the file sets none. |
| `rt.max_frame_bytes_default` | integer | `airlock.rt.max_frame_bytes` when the file sets none. |
| `api` | object | The host request surface's numbers (1.26.0): the request version and the defaults of a world's `api:` block. |
| `api.request_version` | integer | The `v` every request and reply carries, 1. |
| `api.max_connections_default` | integer | `api.max_connections` when the block sets none; one of them is kept for root. |
| `api.max_inflight_default` | integer | `api.max_inflight` (calls in flight across the surface) when the block sets none. |
| `api.max_request_bytes_default` | integer | `api.max_request_bytes`, the longest request line, when the block sets none. |
| `api.max_sample_bytes_default` | integer | `api.max_sample_bytes`, the largest rendered sample or graph reply, when the block sets none. |
| `api.max_raw_sample_bytes_default` | integer | `api.max_raw_sample_bytes`, the largest sample the reader takes, when the block sets none. |
| `api.rate_defaults` | object | The token buckets when the block sets none: `samples`, `calls` and `graph`, each `{per_s, burst}`; since 1.27.0 every admitted uid has its own set at these rates. |
| `api.ledger` | object | The call ledger per callable service: `ids` remembered, `outcomes` kept and `keep_s`, how long an outcome is kept. |
| `api.sample_schema` | integer | The version of a sample reply's document. |
| `api.call_schema` | integer | The version of a call reply's document (`schema/api_call.schema.json`). |
| `api.retain_max` | integer | The most lanes `api.retain` may name (1.26.1): each keeps one standing reader in the half's world. |
| `api.uid_idle_drop_s` | integer | How long a uid may hold no connection before its token buckets are dropped (1.27.0: every admitted uid has its own buckets, re-created full at its next request). |
