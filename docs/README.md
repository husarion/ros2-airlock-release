# ros2_airlock user guide

The airlock joins two ROS 2 worlds that may differ in RMW implementation, RMW configuration, domain ID, and namespace, and forwards exactly what its config allows: allowlisted topics, services, and actions, each with hard rate and size caps. Your robot runs in an isolated world nothing can discover; your fleet tools see a curated, namespaced API.

Install instructions (apt and Docker) are in the [release repository README](https://github.com/husarion/ros2-airlock-release#readme). The airlock ships one image and one deb per supported ROS 2 release: `husarion/ros2-airlock:<version>` and `ros-jazzy-ros2-airlock` for Jazzy, `:<version>-humble` and `ros-humble-ros2-airlock` for Humble, `:<version>-lyrical` and `ros-lyrical-ros2-airlock` for Lyrical, the same engine version in each. The zero-config demo is one command away after install (source the release you installed):

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
11. [Answer everyday questions with the toolbox](use-the-toolbox.md): `trace`, `top`, `check --dry-run`, `open` and `close`, `history` and `init`.
12. [Move a config to schema 6](move-a-config-to-schema-6.md): what the new schema changes, its new keys, and the preview `airlock check` prints for your file.

Configuration reference: every config key is machine-validated by [`airlock.schema.json`](https://raw.githubusercontent.com/husarion/ros2_airlock/main/schema/airlock.schema.json). Put the `yaml-language-server` header from the snippets in your own files and your editor autocompletes and validates as you type. `ros2 run ros2_airlock airlock check <file>` validates offline and explains every error with the rule and the fix.

On Husarion robots (Panther, Lynx) the airlock is managed from Husarion Cockpit and this configuration is generated for you; these pages are for running it on your own robots.
