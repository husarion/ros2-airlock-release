# ros2_airlock releases

This repository distributes release builds of ros2_airlock, Husarion's two-world ROS 2 bridge. The airlock runs your robot's ROS 2 graph in an isolated world with no network exposure and republishes a curated, rate- and size-bounded API into the shared network, so the robot stays visible to your fleet tools but cannot be flooded by them. Source code is not published here; this repository carries the binaries, the apt repository metadata, and this guide.

Each GitHub release provides:

- `ros-jazzy-ros2-airlock` Debian packages for amd64 and arm64 (ROS 2 Jazzy)
- apt repository index files, so you can install and upgrade with plain `apt`
- the changelog for that version

The same versions are also published as the `husarion/ros2-airlock` Docker image on Docker Hub.

## Install with apt

Add the signing key and the repository once:

```bash
curl -fsSL https://raw.githubusercontent.com/husarion/ros2-airlock-release/main/husarion-airlock.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/husarion-airlock.gpg

echo "deb [signed-by=/usr/share/keyrings/husarion-airlock.gpg] https://github.com/husarion/ros2-airlock-release/releases/latest/download/ ./" \
  | sudo tee /etc/apt/sources.list.d/husarion-airlock.list
```

Then install:

```bash
sudo apt update
sudo apt install ros-jazzy-ros2-airlock
```

Upgrades arrive through the normal channel: `sudo apt update && sudo apt upgrade` always tracks the latest release. To pin or roll back, download the specific `.deb` from that version's release page and install it with `apt install ./<file>.deb`.

The signing key fingerprint is:

```
5F29 0E43 49D5 0A12 24A6 53F3 7F52 74C2 DD89 850C
```

## Install with Docker

```bash
docker pull husarion/ros2-airlock:latest
```

On Husarion robots (Panther, Lynx) the airlock ships as part of the robot software stack and is managed from Husarion Cockpit; you do not need to install anything from this page there.

## First run

After the apt install, a complete two-world demo runs on one machine with zero configuration written:

```bash
source /opt/ros/jazzy/setup.bash
ros2 launch ros2_airlock demo.launch.yaml
```

A talker publishes in an isolated "robot" world, the airlock bridges it, and a listener in the "fleet" world prints `/demo_bot/chatter`.

To bridge your own system, write one YAML file that names the two worlds and lists what may cross, then start one `airlock_half` process in each world:

```yaml
schema: 1
airlock:
  ipc: { dir: /run/airlock, listener: robot }
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
```

```bash
# terminal 1, robot world (its own RMW/domain via environment):
ROS_DOMAIN_ID=17 airlock_half --world robot -c airlock.yaml
# terminal 2, fleet world:
ROS_DOMAIN_ID=18 airlock_half --world fleet -c airlock.yaml
```

Each direction is a deny-by-default allowlist: anything not listed does not exist in the destination world. Topics, services, actions, and TF are supported, and every entry carries hard rate and size caps.

Two commands cover most operational questions. `airlock check <config>` validates a configuration offline and explains any error with the rule and the fix. `airlock doctor` inspects a running deployment and answers why a topic is not flowing.

## Support

Issues with a release build, or questions about running the airlock on your robots: support@husarion.com. Release notes for every version are on the [releases page](https://github.com/husarion/ros2-airlock-release/releases).

## License

The packages distributed here are proprietary software, © Husarion. See [LICENSE](LICENSE).
