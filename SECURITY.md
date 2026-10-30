# Security policy

This policy covers ros2_airlock: the `husarion/ros2-airlock` Docker images, the `ros-<distro>-ros2-airlock` Debian packages and the static `airlock_api` binaries published on [husarion/ros2-airlock-release](https://github.com/husarion/ros2-airlock-release).

## Reporting a vulnerability

Email **security@husarion.com**. It is the only report channel: please do not open a public issue, and do not report through GitHub's vulnerability reporting, which is not enabled for these repositories.

A useful report names the version (the image tag or digest, or `airlock_half --version`), the ROS 2 release and RMW on each side of the airlock, the configuration involved (with anything private removed), and the steps that show the problem. Say how you would like to be credited, or that you would rather not be.

## What happens after a report

1. We acknowledge the report within five working days and tell you who handles it.
2. We assess it: whether it is a vulnerability in the airlock or in an upstream component, which versions it affects, and how severe it is. We tell you the outcome and keep you informed while the work goes on.
3. The fix ships through our normal release process, with its regression test and the full release gate. A fix in an upstream component reaches you as an updated pin (a new base image, Fast DDS build or vendored library) in an airlock release.
4. We publish an advisory with the fixed version on the release repository, crediting you unless you asked us not to.
5. We offer you a disclosure window of 90 days from the report, or until the fixed release is out if that comes first. We can agree on a different window with you when a fix needs longer or an upstream project sets its own date.

## Which versions get fixes

The newest release line gets security fixes: a fix arrives as a new patch or minor release of the newest version, and you update to it. Older lines get no fixes until Husarion publishes a support period, which has not been decided yet.

## Upstream components we watch

A vulnerability in a component the airlock ships is handled like one in the airlock itself. We follow the security advisories and release notes of:

- Fast DDS and Husarion's patched Fast DDS build (`husarion/fastdds-patched`) that the robot images carry
- Cyclone DDS
- Zenoh and `rmw_zenoh`
- rclcpp and the rest of the ROS 2 core in the images
- iceoryx2 and the Rust crates vendored with it
- yaml-cpp
- the `ros:<distro>-ros-base` base images and the Ubuntu packages they contain

Every release publishes a CycloneDX SBOM for each image and each `airlock_api` binary (see [Verify the images](https://github.com/husarion/ros2-airlock-release/blob/main/docs/verify-images.md)), so you can match a published advisory against exactly what you run.

## Husarion's reporting obligations under the EU Cyber Resilience Act

This section describes Husarion's own obligations as the manufacturer of the product toward ENISA and the CSIRT designated as coordinator. It is not a promise to the person who reports a vulnerability, and it is separate from the disclosure window above. Husarion reports through ENISA's single reporting platform on two tracks:

- **An actively exploited vulnerability** in the product: an early warning within 24 hours of becoming aware of it, a vulnerability notification within 72 hours, and a final report within 14 days after a corrective or mitigating measure is available.
- **A severe incident affecting the security of the product**: an early warning within 24 hours of becoming aware of it, an incident notification within 72 hours, and a final report within one month of the incident notification.

Husarion also informs the affected users about the vulnerability or incident and the measures they can take.
