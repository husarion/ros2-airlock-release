# ros2_airlock user guide

The airlock joins two ROS 2 worlds that may differ in RMW implementation, RMW configuration, domain ID, and namespace, and forwards exactly what its config allows: allowlisted topics, services, and actions, each with hard rate and size caps. Your robot runs in an isolated world nothing can discover; your fleet tools see a curated, namespaced API.

Install instructions (apt and Docker) are in the [release repository README](https://github.com/husarion/ros2-airlock-release#readme). The airlock ships one image and one deb per supported ROS 2 release: `husarion/ros2-airlock:<version>-jazzy` and `ros-jazzy-ros2-airlock` for Jazzy, `:<version>-humble` and `ros-humble-ros2-airlock` for Humble, `:<version>-lyrical` and `ros-lyrical-ros2-airlock` for Lyrical, the same engine version in each. The zero-config demo is one command away after install (source the release you installed):

```bash
source /opt/ros/jazzy/setup.bash
ros2 launch ros2_airlock demo.launch.yaml
```

Each page below is one task and starts with a working snippet:

01. [Bridge your first topic](bridge-your-first-topic.md): one YAML file, two processes, a topic crosses the wall.
02. [Add a command topic](add-a-command-topic.md): let the outside world drive the robot, on your terms.
03. [Expose a service](expose-a-service.md): services and actions across the wall, with bounded failure.
04. [Namespace a robot](namespace-a-robot.md): prefixes, renames, and TF for one robot or a fleet.
05. [Harden — the airlock pattern](harden-the-airlock-pattern.md): the production profile: deny-by-default, forensics, deployment isolation.
06. [Debug a silent topic](debug-a-silent-topic.md): `airlock trace`, `airlock doctor` and the ladder of reasons a topic isn't flowing.
07. [Bridge two ROS 2 releases](bridge-two-ros2-releases.md): a robot on one release, a fleet on another, one airlock between them; what `same`, `converted` and `refused` mean.
08. [Write a type rule](write-a-type-rule.md): from a refused lane in the doctor's output to a rule file both halves reload.
09. [Check your own types before you deploy](check-your-types-before-you-deploy.md): `airlock drift` on your two images, and how to read its report.
10. [Let the airlock learn your allowlist](learn-your-allowlist.md): `ros2 airlock learn` proposes the config from what your worlds do; try it for five minutes, then keep it or let it expire.
11. [Answer everyday questions with the toolbox](use-the-toolbox.md): `trace`, `top`, `api`, `check --dry-run`, `open` and `close`, `history`, `init`, `migrate`, and the one-command launcher that reads `worlds.<w>.env`.
12. [Move a config to schema 9](move-a-config-to-schema-9.md): `ros2 airlock migrate` rewrites an older file, what it leaves to you, and the shared memory a file reserves.
13. [Threads and CPU pinning](threads-and-pinning.md): the name of every engine thread, which ones carry your traffic, and what a CPU pinner should do with them.
14. [Keep camera traffic away from tf](bounded-bulk-lanes.md): the control, normal and bulk classes, the 4 MiB/s cap on bulk lanes, how to raise or lift it, and where to watch delivery age and dropped samples.
15. [Status and health reference](status-and-health-reference.md): every value a status file carries that a dashboard branches on, the health verdicts in rung order with every reason, and what stays stable.
16. [The constants contract](constants-contract.md): `airlock_half --constants`, the file and labels every image carries, and the rule for changing it.
17. [The check plan contract](check-plan-contract.md): `ros2 airlock check <file> --plan --format json`, every entry's class, cap and real-time policy as one document a tool can read before a file is applied.
18. [Read and call the robot from the host](read-and-call-from-the-host.md): `ros2 airlock sample` and `call` ask the half already in the robot's world through a local socket, instead of starting a participant per read; who may ask, the fallback rule, and status and forensics directories of each half's own.
19. [Verify the images](verify-images.md): check the cosign signature of an image, its SBOM and the static `airlock_api` binary against Husarion's public key before you deploy them.

Configuration reference: every config key is machine-validated by [`airlock.schema.json`](https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json). Put the `yaml-language-server` header from the snippets in your own files and your editor autocompletes and validates as you type. `ros2 run ros2_airlock airlock check <file>` validates offline and explains every error with the rule and the fix. The normative specification and the research behind it are mapped in [`docs/INDEX.md`](../INDEX.md).

On Husarion robots (Panther, Lynx) the airlock is managed from Husarion Cockpit and this configuration is generated for you; these pages are for running it on your own robots.
