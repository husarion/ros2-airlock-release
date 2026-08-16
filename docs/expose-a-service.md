# Expose a service

A service server in one world becomes callable from the other by listing it under the direction whose `from` world hosts the server. Here the robot's e-stop becomes callable from the fleet world:

```yaml
    - from: robot
      to: fleet
      namespace: /my_robot
      topics:
        - topic: /chatter
          type: std_msgs/msg/String
          max_bytes: 65536
          max_hz: 50
      services:
        - service: /hardware/e_stop_trigger
          type: std_srvs/srv/Trigger
          timeout_s: 2.0
          max_request_bytes: 1024
```

```bash
# fleet world: call the robot's service on its namespaced name
ROS_DOMAIN_ID=18 ros2 service call /my_robot/hardware/e_stop_trigger std_srvs/srv/Trigger
```

The call crosses the wall, executes on the robot's real server, and the response comes back. Unlisted services do not exist in the fleet world: `ros2 service list` there shows only what you exposed.

## The rules services follow

**Direction means server location.** `services` under `from: robot` exposes robot-world servers to fleet-world callers. The reverse direction's `services` list would expose fleet-world servers to the robot, which the production profile leaves empty.

**No call ever hangs.** `timeout_s` is the ceiling on every call. Upstream server down, peer half dead, entry removed by a reload, plain timeout: every failure path returns an error reply to the caller within bounded time. A dead robot-side server can never wedge a fleet-side caller's future.

**A reply the airlock had to invent says so, and you must read it.** ROS services carry no error channel, so a call the bridge could not complete comes back as a default-initialized response — for `std_srvs/srv/Trigger` that is `success=false`, which on its own is exactly what a server refusing looks like. So every invented reply puts the reason in the response's `message` field, prefixed `airlock:`, and says that the request WAS forwarded and may have been executed:

```text
response:
std_srvs.srv.Trigger_Response(success=False, message='airlock: TIMEOUT after 2.0 s waiting for the server's reply — the request was forwarded and MAY HAVE BEEN EXECUTED; this is not a refusal, verify the robot's state before retrying')
```

Treat that as "unknown", never as "no" — this bit us on a real robot, where an e-stop trigger the caller was told had failed had in fact latched. Automation calling a safety service should check the `message` prefix (or the robot's actual state) before deciding a command did not take effect. The halves count these per service in `status_<world>.json` and `/metrics` (`airlock_service_timeouts_total`), and `airlock doctor` fails while any are recorded — a service that times out regularly needs a longer `timeout_s` or a faster server, not a caller that learns to ignore it.

**Requests are bounded.** `max_request_bytes` rejects oversized requests before they cross. Services are for commands and parameters, not bulk data; the shared-memory fast path is for topics.

**The type's support library must be installed in both halves' containers.** Topics cross as raw bytes with no type code involved, but services convert between wire and in-memory representation at each world edge, which loads the type's support library at runtime. Standard ROS types ship with the image. For your own interface packages, extend the image with your interface debs (`FROM husarion/ros2-airlock` plus an `apt install` of your packages). A service whose typesupport is missing is skipped with a diagnostic, never fatal, and `airlock doctor` points at exactly this.

## Actions

Actions work the same way, under an `actions` list. A Nav2 goal sent in the fleet world executes on the robot, with live feedback, status, and cancel:

```yaml
      actions:
        - action: /navigate_to_pose
          type: nav2_msgs/action/NavigateToPose
          timeout_s: 60.0
```

The airlock bridges the goal, result, and cancel services plus the feedback and status topics as one unit, and correlates goals across the wall by their UUID. Fleet-side tooling sees a genuine action server on `/my_robot/navigate_to_pose`; Open-RMF fleet adapters work through it unmodified. Note that `timeout_s` bounds every bridged call of the action, and the result request completes only when the goal itself finishes, so set it longer than your longest expected goal. The Nav2 test configurations use 60 s.
