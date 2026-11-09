# Verify the images

From 1.28.0 on, Husarion signs every image it publishes, the SBOM attached to each image, and every static `airlock_api` binary, with one cosign key. You can check that what you pull is what Husarion released, and read exactly which packages and Rust crates it contains, before you deploy it. Releases before 1.28.0 carry no signatures.

```bash
v=1.29.0   # the release you deploy
curl -fsSLO https://github.com/husarion/ros2-airlock-release/releases/download/v$v/cosign.pub
cosign verify --key cosign.pub --insecure-ignore-tlog=true husarion/ros2-airlock:$v-jazzy
```

`cosign verify` prints the signed payload and exits 0 when the signature on the image matches `cosign.pub`; anything else exits non-zero, and you should not deploy that image.

**Why `--insecure-ignore-tlog=true` is on every command:** Husarion keeps these signatures out of the public Sigstore transparency log on purpose, so cosign has no log entry to check and refuses the signature without the flag; the check against `cosign.pub` still runs in full.

## What you need

- `cosign`, from the [Sigstore releases](https://github.com/sigstore/cosign/releases). Husarion signs with cosign 2.6.5; the commands on this page were checked with 2.6.5 and 3.1.3. With cosign 3, `verify-blob` prints a deprecation notice for `--signature` and verifies the same way.
- `cosign.pub`, the public key. It is a release asset of every signed release, sits at the root of the [release repository](https://github.com/husarion/ros2-airlock-release) and is printed in full below. Keep the copy you trust and compare it with each new release's: a release signed by another key is a reason to stop and ask Husarion at security@husarion.com.
- For the SBOM steps, `docker buildx` (or `crane`) to look up the platform digests, and `jq`.

## The public key

This is Husarion's release key, `cosign.pub`, as text. Compare it byte for byte with the copy you download:

```text
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEKig75dB9eVg+/tQClFdLkl15Lt6w
WcV0+Npu+KQrW20NJ52SZ+9lHzf/jilFbVWp+5BenO8ykKWr4zRWSQhNHg==
-----END PUBLIC KEY-----
```

From 1.28.1 on, every image also carries a copy at `/usr/share/doc/ros2_airlock/cosign.pub`. That copy is there for comparison only. Never verify an image with the key it carries itself, because whoever changed the image could have changed the key too. Verify with a copy you obtained independently: this page, the release page or the root of the release repository.

## Verify an image

Each tag `:<version>-<distro>` names a multi-arch index. The signature covers the index and both platform images (amd64 and arm64), so verifying the tag, the index digest or one platform's digest all work:

```bash
cosign verify --key cosign.pub --insecure-ignore-tlog=true husarion/ros2-airlock:$v-humble
```

In production, pin the digest rather than the tag. `cosign verify` names it in its output (`docker-manifest-digest`); verify that digest and deploy by it:

```bash
cosign verify --key cosign.pub --insecure-ignore-tlog=true husarion/ros2-airlock@sha256:<index digest>
```

## Verify and read an image's SBOM

Each platform image carries a signed CycloneDX attestation: the Debian packages, the Python packages and the Rust crates of the vendored iceoryx2 that the image contains. The attestation belongs to the platform image, so look up its digest first:

```bash
docker buildx imagetools inspect husarion/ros2-airlock:$v-jazzy   # lists one digest per platform
cosign verify-attestation --key cosign.pub --insecure-ignore-tlog=true --type cyclonedx \
  husarion/ros2-airlock@sha256:<arm64 digest> > attestation.json
jq -rs '.[-1].payload' attestation.json | base64 -d | jq .predicate > sbom.cdx.json
```

The same SBOM is a file on the release page, `sbom-<version>-<distro>-<arch>.cdx.json`, for tools that read SBOMs from disk. The attested copy is the one whose signature you checked.

## Verify the static `airlock_api` binary

Each architecture's binary has a `.sha256` and a `.sig` beside it on the release page, and so does its SBOM (`sbom-airlock_api-<version>-linux-<arch>.cdx.json`, made from the packages the static link pulled in, because the binary carries no package database). From 1.28.1 on the binary is built against musl on Alpine with yaml-cpp built in, so its SBOM lists the Alpine packages (musl and the GCC runtime) and the vendored yaml-cpp:

```bash
a=arm64
base=https://github.com/husarion/ros2-airlock-release/releases/download/v$v
for f in airlock_api-$v-linux-$a airlock_api-$v-linux-$a.sha256 airlock_api-$v-linux-$a.sig; do curl -fsSLO "$base/$f"; done
sha256sum -c airlock_api-$v-linux-$a.sha256
cosign verify-blob --key cosign.pub --insecure-ignore-tlog=true \
  --signature airlock_api-$v-linux-$a.sig airlock_api-$v-linux-$a
```

`Verified OK` means the binary is the one Husarion signed. The SBOM verifies the same way with its own `.sig`.

## Licences and the written source offer

From 1.28.1 on, every image carries `/usr/share/doc/ros2_airlock/` with:

- `ATTRIBUTION.md`: every open-source component in the image with its version, licence and copyright holders, and the changes Husarion made to iceoryx2 and to Fast DDS;
- `licenses/`: the full text of each licence a component needs that its own Debian copyright file does not carry;
- `licence-inventory.csv`: one row per component with its licence, whether it is copyleft, and how the airlock links it;
- `SOURCE-OFFER`: Husarion's written offer to provide the complete source of every copyleft component of that image on request to security@husarion.com, for three years from the release date or for as long as the release is supported, whichever is longer (the image label `com.husarion.airlock.source-offer` says the same);
- `cosign.pub`, for comparison (see above).

Read them without starting the image: `docker create`, then `docker cp <container>:/usr/share/doc/ros2_airlock .`.

The release page also carries each image's inventory as `licence-inventory-<version>-<distro>-<arch>.csv`, and each `airlock_api` binary's own attribution and inventory as `attribution-airlock_api-<version>-linux-<arch>.md` and `licence-inventory-airlock_api-<version>-linux-<arch>.csv`. Each has a `.sig` and verifies like the binary:

```bash
cosign verify-blob --key cosign.pub --insecure-ignore-tlog=true \
  --signature licence-inventory-$v-jazzy-arm64.csv.sig licence-inventory-$v-jazzy-arm64.csv
```

Each deb carries the attributions of what it installs in `/usr/share/doc/ros-<distro>-ros2-airlock/copyright`; the packages it depends on carry their own.

## The Debian packages

The `ros-<distro>-ros2-airlock` packages come from a GPG-signed apt repository, and apt checks that signature on every install. The release repository's README has the key and the source line.

## Reporting a problem

If a signature does not verify, or you find a vulnerability, write to security@husarion.com. The security policy says what happens next; its copy of record is [SECURITY.md in the release repository](https://github.com/husarion/ros2-airlock-release/blob/main/SECURITY.md).
