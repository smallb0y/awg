# YAAWG: Yet another AmneziaWG variation for OpenWrt

## FAQ & TL;DR

**Q: Where can I get the binaries?**  
A: Prebuilt packages for the most popular architectures are published on the [releases page](../../releases): a versioned release for the latest stable OpenWRT version and a daily refreshed `snapshot` pre-release for `SNAPSHOT`. If your architecture is not covered, [build it yourself](#how-to-use-the-new-workflow).

**Q: What is the latest supported version of the protocol?**  
A: YAAWG fully supports AmneziaWG v3.1, adding random packet trailers and the option to disable cookie replies on top of the v3.0 header protection, content padding and customizable timings, and everything v2.0 offers (S3-S4, I1-I5 and ranged H1-H4 parameters). See the [protocol parameters](#protocol-parameters) section.

**Q: Should I use the kernel module or the Go implementation?**  
A: Use the kernel module by default. If it doesn't work for you, switch to the Go implementation. More info [here](#kmod-amneziawg-vs-amneziawg-go).

**Q: Why should I still compile the packages myself?**  
A: There are several reasons:  
1. OpenWRT supports many architectures and targets, and only the most popular ones are prebuilt.  
2. It takes just a few clicks and about 20 minutes to compile binaries with your parameters.  
3. Kernel modules built for `SNAPSHOT` are only installable while the firmware `vermagic` matches.
4. You can review all the sources and make sure there are no unexpected issues or vulnerabilities before building or deploying.

**Q: How are versions named?**  
A: Versioning follows the pattern `x.y.z` where `x` and `y` represent the current AmneziaWG version, and `z` corresponds to the YAAWG version.

**Q: I get errors on steps `Download and prepare SDK`, `Checkout OpenWRT repository`, `Update feeds`. Any advice?**  
A: It looks like the OpenWRT repository went for a coffee break and is unavailable at the moment. Try restarting the workflow in 15-40 minutes.

## Project description

This project aims to update the sources of the initial AmneziaWG repository and align them as closely as possible with four upstream projects ([luci-proto-wireguard](https://github.com/openwrt/luci/tree/master/protocols/luci-proto-wireguard), [amneziawg-tools](https://github.com/amnezia-vpn/amneziawg-tools/), [amneziawg-linux-kernel-module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module), [amneziawg-go](https://github.com/amnezia-vpn/amneziawg-go)).

Why? Because it seems the original repository has been abandoned: over half a year has passed since the last commit, no bugs were fixed, and no new protocol versions were added, while the four upstream projects continue receiving regular updates (at least from the community).

The main differences and objectives are:
1. `luci-proto-amneziawg` has been aligned with [luci-proto-wireguard](https://github.com/openwrt/luci/tree/master/protocols/luci-proto-wireguard):
   - Based on the `luci-proto-wireguard` codebase.
   - The AmneziaWG settings tab now includes placeholders where available (default values, if chosen, mimic the Wireguard protocol).
   - Fixed a bug with QR code generation. The generated QR code now contains AmneziaWG-specific information.
   - Added checkboxes to enable/disable peers.
   - Added an icon for the interface.
   - Added support for ranged H1-H4 parameters (delimiter: `-`, e.g., `123456-123500`), including a check that the four ranges do not overlap each other.
   - Added support for v2.0 protocol parameters: S3-S4, I1-I5.
   - Added support for v3.0 protocol parameters, ranged `Persistent Keep Alive` and a generator for the header protection key.
   - Added support for v3.1 protocol parameters: the `Random Trailers` and `Disable Cookies` interface checkboxes.
   - The AmneziaWG parameters are now described by a single table, so the settings tab, the configuration import and the configuration export can never drift apart.
   - The exported peer QR code is now bounded so that even a long v3.1 configuration always scales down to fit the page instead of getting cut off.

2. `amneziawg-tools` has been aligned with the upstream repository [amneziawg-tools](https://github.com/amnezia-vpn/amneziawg-tools/):
   - The package is now compiled based on the upstream repository.
   - Fixed a bug with the non-existent `proto_amneziawg_check_installed` method.
   - Changed temporary folders and files to match the protocol name.
   - Refactored scripts to make them look more "amneziish".
   - Fixed a bug with an incorrect path when using `amneziawg-go`.
   - Added support for ranged H1-H4 parameters (with `-` delimiter, e.g., `123456-123500`).
   - Added support for v2.0 protocol parameters: S3-S4, I1-I5.
   - Added support for v3.0 protocol parameters and ranged `PersistentKeepalive`.
   - Added support for v3.1 protocol parameters: `RandomTrailers` and `DisableCookies` on the interface.

3. `kmod-amneziawg` is now compiled entirely based on the upstream [amneziawg-linux-kernel-module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module) repository.
   - Added support for v2.0 protocol parameters: S3-S4, I1-I5.
   - Added support for v3.0 protocol parameters.
   - Added support for v3.1 protocol parameters.

4. `amneziawg-go` acts as an alternative to `kmod-amneziawg`. Please refer to [this section](#kmod-amneziawg-vs-amneziawg-go) for more information. The Go implementation is fully based on the upstream project [amneziawg-go](https://github.com/amnezia-vpn/amneziawg-go).
   - Added support for v3.0 protocol parameters.
   - Added support for v3.1 protocol parameters.

5. Packaging and build process:
   - Prebuilt packages for the most popular architectures are published automatically, see [below](#download-prebuilt-amneziawg-packages).


## Protocol parameters

All AmneziaWG parameters are optional. A parameter that is left empty is treated as `0`, and an interface with every parameter left empty behaves exactly like plain WireGuard. Interface parameters live on the `AmneziaWG Settings` tab of the interface (UCI section `network.<interface>`), while `Persistent Keep Alive` is configured per peer.

Some parameters are **server-side**: they describe the wire format and therefore must be identical on both ends of the tunnel. The others are **client-side**: they only influence the behaviour of the local instance and may differ between peers.

Since v3.0 several parameters accept a **range** instead of a single number. A range is written as `low-high` (e.g. `25-30`), where `low` must not be greater than `high`, and makes AmneziaWG pick a fresh random value from these bounds for every use.

### Parameters of AmneziaWG v1.5 and v2.0

| UCI option | Configuration file key | Side | Description |
| --- | --- | --- | --- |
| `awg_jc` | `Jc` | client | Amount of junk packets sent before every handshake. The recommended range is 4-12. |
| `awg_jmin` | `Jmin` | client | Minimum size of a junk packet, in bytes. Must not exceed `Jmax`. |
| `awg_jmax` | `Jmax` | client | Maximum size of a junk packet, in bytes. Keep it below the MTU of your uplink, otherwise the junk packets get fragmented and look suspicious. |
| `awg_s1` | `S1` | server | Junk header prepended to the handshake initiation packet, in bytes. |
| `awg_s2` | `S2` | server | Junk header prepended to the handshake response packet, in bytes. |
| `awg_s3` | `S3` | server | Junk header prepended to the cookie reply packet, in bytes. |
| `awg_s4` | `S4` | server | Junk header prepended to transport packets, in bytes. |
| `awg_h1` | `H1` | server | Packet type header of the handshake initiation packet. Default Wireguard value is `1`. |
| `awg_h2` | `H2` | server | Packet type header of the handshake response packet. Default Wireguard value is `2`. |
| `awg_h3` | `H3` | server | Packet type header of the cookie reply packet. Default Wireguard value is `3`. |
| `awg_h4` | `H4` | server | Packet type header of the transport packet. Default Wireguard value is `4`. |
| `awg_i1` … `awg_i5` | `I1` … `I5` | client | Signature packets sent before every handshake, in the order `I1`, `I2`, …, `I5`. An empty parameter simply skips its packet. |

`H1`-`H4` are 32-bit values and accept ranges. The four ranges must not overlap each other, otherwise the interface cannot tell one packet type from another and refuses to start.

The signature packets `I1`-`I5` are described by a sequence of tags that is expanded anew for every packet:

| Tag | Description |
| --- | --- |
| `<b 0x[hex]>` | Writes the given bytes verbatim. The hex sequence is always of even length. |
| `<r [size]>` | Writes `[size]` random bytes. |
| `<rd [size]>` | Writes `[size]` random digits (`0-9`). |
| `<rc [size]>` | Writes `[size]` random characters (`a-zA-Z`). |
| `<c>` | Writes an incrementing counter. |
| `<t>` | Writes the current time as a 4 byte UNIX timestamp. |

A typical value therefore looks like `<b 0xc21f2749><r 16><t>`. As with junk packets, keep the resulting size below the MTU.

### Parameters of AmneziaWG v3.0

| UCI option | Configuration file key | Side | Description |
| --- | --- | --- | --- |
| `awg_header_protection_key` | `HeaderProtectionKey` | server | Base64-encoded 32 byte key that encrypts the low entropy header fields WireGuard uses for authentication. Generate it with the button next to the field or with `awg genkey`. |
| `awg_content_padding_addition` | `ContentPaddingAddition` | server | Extra padding appended to transport packets, in bytes. Accepts a range. |
| `awg_rekey_after_time` | `RekeyAfterTime` | client | Seconds after which a new handshake is started. Accepts a range. Default is `120`. |
| `awg_rekey_timeout` | `RekeyTimeout` | client | Seconds after which an unanswered handshake is repeated. Accepts a range. Default is `5`. |
| `awg_reject_after_time` | `RejectAfterTime` | client | Seconds after which a session key is discarded and incoming data is rejected. Accepts a range. Default is `180`. |
| `awg_keepalive_timeout` | `KeepaliveTimeout` | client | Seconds of silence after which a keep alive message is sent. Accepts a range. Default is `10`. |
| `awg_max_handshake_attempts` | `MaxHandshakeAttempts` | client | Number of handshake retries before the peer is given up on. Accepts a range. Default is `18`. |
| `persistent_keepalive` (per peer) | `PersistentKeepalive` | client | Seconds between keep alive messages. Since v3.0 it also accepts a range, e.g. `25-30`. Default is `0`, which disables it. |

How to use them:

1. **Header protection** is the strongest addition of v3.0. It removes the last static fields that can be fingerprinted, at the cost of requiring `S1`-`S4` to be at least `12` (the nonce is taken from the crypto padding). Set the same `HeaderProtectionKey` on the server and on every client, and make sure `S1`-`S4` are set accordingly; otherwise the interface refuses to come up.
2. **Content padding** hides the exact size of the encrypted payload. It is meant to be set on both sides, although it also works one-sided.
3. **Timings** change how often a client rekeys, retries and sends keep alives. Because they only alter local behaviour, they are the safest parameters to experiment with: use ranges to avoid a regular, easily recognizable traffic pattern. Note that a `RejectAfterTime` shorter than `RekeyAfterTime` makes the tunnel stall.
4. **Ranged `Persistent Keep Alive`** serves the same purpose for the keep alive interval of a single peer.

### Parameters of AmneziaWG v3.1

v3.1 adds three **boolean** parameters. In a text configuration file a boolean is written as `on`/`off` (the canonical form emitted by `awg showconf`) or as `1`/`0`; the words `true`/`false` are **not** accepted. In LuCI they are simple checkboxes, and YAAWG writes the enabled state as `on` and omits the parameter entirely when it is left off.

| UCI option | Configuration file key | Side | Description |
| --- | --- | --- | --- |
| `awg_random_trailers` | `RandomTrailers` | server | Appends a random-length trailer of extra bytes to packets so their on-wire size varies and cannot be fingerprinted. Must be enabled on both ends of the tunnel. |
| `awg_disable_cookies` | `DisableCookies` | server | Stops sending the WireGuard cookie reply messages, removing a distinctive packet type used under load. Meant to be enabled on both ends. |

How to use them:

1. **Random trailers** hide the exact packet length. Because a receiver only accepts oversized packets when it, too, has the option enabled, it changes the wire format and must be set on both ends.
2. **Disable cookies** removes the cookie reply, one of the last fixed WireGuard message types. Enable it on both ends so that neither side expects the cookie handshake.

> The v3.1 protocol also defines a per-peer `AdvancedSecurity` flag. It is not exposed by YAAWG yet, because the current kernel module and Go implementation do not act on it; it will be added once they gain real support for it.

### Backward compatibility

AmneziaWG v3.1 is a superset of the previous versions, and both `kmod-amneziawg` and `amneziawg-go` keep supporting everything that came before:

1. With all parameters left empty, an AmneziaWG v3.1 interface is indistinguishable from plain WireGuard and interoperates with WireGuard peers.
2. With only the v1.5 and v2.0 parameters filled in, it behaves exactly like an AmneziaWG v1.5 or v2.0 interface and interoperates with peers running those versions. Existing configurations therefore keep working after the upgrade and require no changes.
3. The client-side timing parameters (the v3.0 timings and the ranged `Persistent Keep Alive`) may be used against an older peer, since they never change the wire format.
4. The server-side parameters that change the wire format (`HeaderProtectionKey`, `ContentPaddingAddition` from v3.0 and `RandomTrailers`, `DisableCookies` from v3.1) must be set identically on both ends. A peer that does not enable them will silently drop the packets it cannot parse.
5. Always upgrade `amneziawg-tools` together with `kmod-amneziawg` or `amneziawg-go`. The tools only pass a parameter down to the implementation when it is actually configured, so an older kernel module keeps working as long as the newer parameters are empty, but starts to fail with `Unable to modify interface` once they are set.

## `kmod-amneziawg` vs `amneziawg-go`

When the AmneziaWG authors introduced the v1.5 protocol, it was supported only in the Go implementation. Thus the user namespace (Go) implementation was added to the repo in order to support the newer protocol version. Later, v2.0, v3.0 and v3.1 protocol support was added to both the user namespace (Go) and kernel module implementations. To maintain backward compatibility, this repository will continue to support both packages.

Differences:
1. `kmod-amneziawg`: requires a less powerful device to run, consumes less space and provides a faster throughput. Recommended option.
2. `amneziawg-go`: requires a more powerful device and uses more space but provides a user namespace implementation of the protocol. Use it if kernel module doesn't work for you.

If both implementations are installed, `kmod-amneziawg` will be used by default.

## Results

Everything seems to work fine. No major problems have been detected or reported so far.

## How to build and use

### Build OpenWRT firmware with AmneziaWG packages included

This repository is intended primarily for compiling packages during the firmware build process. Follow these steps:

1. Clone the OpenWrt repo by running `git clone https://github.com/openwrt/openwrt.git`. You may choose any tag/commit hash by adding `-b {tag/commit hash}`.

2. Add the line `src-git awgopenwrt https://github.com/this-username-has-been-taken/amneziawg-openwrt.git` to the `feeds.conf.default` file.

3. Update package feeds by running: `{path to openwrt dir}/scripts/feeds update -a`

4. Shall you build the firmware with the `amneziawg-go` package, please make sure the included Go package version is `1.25.0` or higher. Most OpenWRT versions except `SNAPSHOT` have older Go versions. To upgrade:
   - Clone the latest OpenWRT Packages repository: `git clone https://github.com/openwrt/packages.git`.
   - Replace `{path to openwrt dir}/feeds/packages/lang/golang` with the one from the cloned repository at `{path to the cloned repository}/packages/lang/golang`.

5. Install packages by running: `{path to openwrt dir}/scripts/feeds install -a`

6. Configure firmware (choose target, settings, AmneziaWG packages: `amneziawg-go` or `kmod-amneziawg` + `amneziawg-tools` + `luci-proto-amneziawg` and others) using: `make -C {path to openwrt dir} menuconfig` and save.

7. Make the defconfig: `make -C {path to openwrt dir} defconfig`

8. Build the firmware: `make -C openwrt -j$(nproc) V=sc`

9. After building, firmware will be located at: `{path to openwrt dir}/bin/targets/{your target}/{your subtarget}` and compiled packages at: `{path to openwrt dir}/bin/targets/{your target}/{your subtarget}/packages` (kernel module) and `{path to openwrt dir}/bin/packages/{your architecture}/awgopenwrt` (other packages).

### Download Prebuilt AmneziaWG Packages

The most popular architectures are built automatically and published on the [releases page](../../releases), so for most routers no compilation is needed at all.

1. Obtain your router parameters:
   - **OpenWRT version:** `SNAPSHOT` or a stable release (e.g., `24.10.2`), found under `Status -> Overview` on the `Firmware Version` line.
   - **Target and Subtarget:** found under `Status -> Overview` on the `Target Platform` line (before and after the slash).

2. Pick the archive matching your parameters:
   - For a stable release, take it from the latest versioned release. It is built against the newest stable OpenWRT version.
   - For `SNAPSHOT`, take it from the `snapshot` pre-release, which is rebuilt every day. Since `SNAPSHOT` firmware changes daily as well, the kernel module only installs while the `vermagic` recorded in `build-info.txt` matches your firmware (see [below](#vermagic-control-for-snapshot-versions)).

3. Extract the archive and [install the packages](#how-to-install-amneziawg).

Two workflows keep those artifacts up to date, and both of them can also be started by hand from a fork:

1. `Release - Build AmneziaWG for the latest OpenWrt release` runs whenever a `vX.Y.Z` tag is pushed. It resolves the newest stable OpenWRT version on its own and collects the packages into a draft release, so that the changelog can be written before it goes public. Started by hand against an already published release, it only adds the new archives to it, which is how an unchanged YAAWG version is rebuilt for a freshly released OpenWRT version. The same workflow also runs once a day on a schedule: it looks up the newest stable OpenWRT version and the latest published YAAWG release, and if that release does not yet contain archives for that OpenWRT version it builds them and attaches them automatically. When everything is already up to date the scheduled run does nothing, so new OpenWRT releases are covered without any manual tracking.
2. `Snapshot - Build AmneziaWG for OpenWrt SNAPSHOT` runs daily and republishes the rolling `snapshot` pre-release.

Every archive is named `amneziawg-{target}-{subtarget}-{architecture}-openwrt-{OpenWRT version}.tar.gz`, so builds of the same YAAWG version for several OpenWRT versions can live side by side in one release.

The list of built architectures lives in [.github/targets.json](.github/targets.json); add an entry there to cover another target.

### Compile AmneziaWG Packages Without Building the Firmware

You can compile packages independently without building the full firmware. There are two workflows available: the **new workflow** and the **legacy workflow**. It is recommended to use the new workflow. The legacy workflow will continue to be supported to maintain backward compatibility.

#### New Workflow vs. Legacy Workflow

Key differences between the workflows:

1. The new workflow runs much faster (~20 minutes vs. 2.5 hours).
2. The new workflow uses the SDK instead of building the toolchain from scratch.
3. The new workflow consists of a single step instead of two.
4. The new workflow also compiles the localization package.

Both workflows record the `vermagic` value on the `Vermagic:` line of `build-info.txt` within the artifacts (see below).

#### How to Use the New Workflow

The new workflow is a single step process: run it, and when complete, all compiled packages will be available in the run's artifacts section (at the bottom of the GitHub Actions page).

Steps to follow:

1. Obtain your router parameters:
   - **OpenWRT version:** `SNAPSHOT` or a stable release (e.g., `24.10.2`), found under `Status -> Overview` on the `Firmware Version` line.
   - **CPU/package architecture:** run `apk info kernel` or `opkg info kernel` to check the `Architecture` value (e.g., `aarch64_cortex-a53`), or consult the [OpenWRT router database](https://openwrt.org/toh/start).
   - **Target and Subtarget:** found under `Status -> Overview` on the `Target Platform` line (before and after the slash).

2. Fork this repository.

3. Enable GitHub Actions, if not already enabled.

4. Select the workflow `New - Build AmneziaWG from SDK`, enter your router parameters in the `Run workflow` form, and start the run.
   - Optionally, specify a different YAAWG version using the release tag or commit hash field.
   - Choose whether to compile the kernel module, Go implementation, or both.

5. Wait approximately 20 minutes until the build completes.

6. If you get an error in step `Download and prepare SDK`, it is more likely that the OpenWRT repository is unavailable. Please restart the process in 15-40 minutes.

7. Download the artifacts, extract them, and install the packages.

#### How to Use the Legacy Workflow

The legacy process involves two workflows (steps): building the OpenWRT toolchain cache (about 2.5 hours) and compiling AmneziaWG packages (under 20 minutes). The toolchain build needs to be completed once, after which the package compilation step can be repeated as needed.

Steps:

1. Obtain your router parameters following the same instructions as in the new workflow.

2. Fork this repository.

3. Enable GitHub Actions, if not already enabled.

4. Select the workflow `Legacy - step 1. Build OpenWrt toolchain cache`, enter your router parameters, and start the run.
   - Optionally set a different YAAWG version using the release tag or commit hash field.
   - Do **not** disable `Update Go` unless you understand the consequences; `amneziawg-go` requires Go version 1.25.0 or higher.

5. Wait approximately 2 to 2.5 hours for the cache build to complete.

6. If you get an error in steps `Checkout OpenWRT repository`, `Update feeds`, it is more likely that the OpenWRT repository is unavailable. Please restart step 1 in 15-40 minutes.

7. Select the workflow `Legacy - step 2. Build AmneziaWG from cache`, input the parameters, and run it.
   - Choose whether to compile the kernel module, Go implementation, or both.
   
8. Again, errors in step `Checkout OpenWRT repository` show that the OpenWRT repository is unavailable. Please restart step 2 in 15-40 minutes.

9. Wait around 10–20 minutes for the packages to compile.

10. Download the artifacts, extract, and install.

### How to Install AmneziaWG

1. Choose your installation method:

   - **Via Web Interface (LuCI):**
     - Navigate to `System -> Software`.
     - Click `Upload Package...`.
     - Upload `kmod-amneziawg` or `amneziawg-go` `.ipk` or `.apk` files.
     - Confirm the installation.
     - Repeat for `amneziawg-tools` and `luci-proto-amneziawg`.

   - **Via Console:**
     - Transfer the package files to your router.
     - Run `apk install {path to kmod-amneziawg or amneziawg-go .apk}` or `opkg install {path to kmod-amneziawg or amneziawg-go .ipk}`.
     - Install `amneziawg-tools` and `luci-proto-amneziawg` similarly.

2. Reboot the router or restart the network service with: `/etc/init.d/network restart`

3. Congratulations! AmneziaWG is installed. Go to `Network -> Interfaces`, click `Add new interface...`, then select `AmneziaWG` as the protocol.

> **Note:** You may need to clear your browser cache to see the new protocol available in OpenWRT.

#### Vermagic control for `SNAPSHOT` versions
> **Note:** Every workflow records the `vermagic` on the `Vermagic:` line of `build-info.txt` within the artifacts — the prebuilt `release` and `snapshot` archives as well as both compile-it-yourself workflows (`New - Build AmneziaWG from SDK` and the legacy one).

Vermagic is a hash calculated for the OpenWRT kernel. When installing kernel-related packages, OpenWRT checks if the package's `vermagic` matches the kernel's. If not, installation won't succeed. Since `SNAPSHOT` versions update daily, `vermagic` values may differ. Check your firmware's `vermagic` by running `apk info kernel` or `opkg info kernel` and noting the hash after the kernel version in `Version`. For example, `6.6.52~f58afd3748410d3b1baa06a466d6682-r1` means `vermagic` is `f58afd3748410d3b1baa06a466d6682`. Every workflow records the compiled package's `vermagic` value on the `Vermagic:` line of `build-info.txt` within the artifacts. If these do not match, the kernel module cannot be installed.
