# Xiaomi SDM845 Kernel Builder with BakaSU or OriginSU

[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue?logo=github-actions&logoColor=white)](https://github.com/1992Ade/sdm845-origin-builder/actions/workflows/build.yml)
[![Latest release](https://img.shields.io/github/v/release/1992Ade/sdm845-origin-builder?label=latest&logo=github)](https://github.com/1992Ade/sdm845-origin-builder/releases/latest)

> [!IMPORTANT]
> **This builder defaults to BakaSU. OriginSU `v4.3.0` is available as an alternative.**
> OriginSU has been renamed and rebuilt as **[KazeSU](https://github.com/KazeSU/KazeSU)**, which targets modern (GKI 5.10+) kernels and **does not build on this Linux 4.9 kernel** (its SELinux code needs the 5.10+ layout). OriginSU `v4.3.0` remains available for this kernel (non-GKI hooks, SuSFS, built-in module mounting); it is frozen, but the builder is still maintained by hand for kernel updates and optimizations.
> **4.9 is also the newest kernel LineageOS ships for SDM845.** Mainline Linux runs on some of these phones (POCO F1, Mi MIX 2S), but only for Linux distributions or bare AOSP test builds, not for LineageOS-based ROMs.
>
> KazeSU builds for SDM845 ([sdm845-KazeSU-builder](https://github.com/1992Ade/sdm845-KazeSU-builder)) are **on hold** until KazeSU supports older kernels again. The matching OriginSU `v4.3.0` manager APK is available from the KazeSU repository: [tag `v4.3.0`, release "legacy-v4.3.0"](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0). BakaSU builds are published as GitHub Releases; OriginSU builds are available only as workflow-run artifacts.

> **Your warranty is now void.** Flashing a custom kernel can brick your device. You are responsible for what you flash. Always keep a backup of your stock `boot` partition.

## 📱 About This Repository

This repository automates the compilation of a custom **Linux 4.9** kernel for **Xiaomi Snapdragon 845 (SDM845)** devices — Mi 8, POCO F1, Mi MIX 2S and their siblings. It is based on the LineageOS-derived kernel tree by **[Coconutat](https://github.com/Coconutat/android_kernel_xiaomi_sdm845_lineageos_Exp)**, which already ships **SuSFS** and its inline hooks (originally used with BakaSU).

The workflow builds the kernel with **BakaSU by default**, or **[OriginSU v4.3.0](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)** when selected. Both build modes include **SuSFS** (v2.4.0 by default), **Baseband Guard**, **NoMount**, **Droidspaces**, **TCP BBR** and the [optimizations](#-optimizations) below, and produce a recovery-flashable `.zip` for the selected device. The current matrix builds **dipper (Xiaomi Mi 8) only**. BakaSU zips are published in a GitHub Release and attached to the Actions run; OriginSU builds are artifacts only.

> [!WARNING]
> **LineageOS-based ROMs only.** This is a LineageOS kernel. Kernels for LineageOS (and ROMs built on it, such as /e/OS) and kernels for other ROM families such as crDroid are **not interchangeable** on SDM845: one does not boot on the other. Check that your ROM uses the LineageOS `sdm845` kernel before flashing.


**Manual builds only:** run the workflow from the Actions tab. BakaSU is the default and creates or replaces a release for the selected LineageOS version; selecting OriginSU produces run artifacts only.

**Always the latest LineageOS:** each run picks the most recent LineageOS version available in the Coconutat tree (today `lineage-22.2-BakaSU`, LineageOS 22.2 / Android 15). When a newer LineageOS branch with SuSFS appears (e.g. LineageOS 23 / Android 16), the next manual run switches to it on its own.

---

## 🧩 Included Features

Both root-solution build modes enable the following kernel features and optimizations:

| Feature | Upstream | What it does | Kernel config |
|---|---|---|---|
| **SuSFS v2.4.0** | [simonpunk](https://gitlab.com/simonpunk/susfs4ksu), 4.9 port by [JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd) | Hides root/mounts from detection. The Coconutat tree ships v2.3.0; the build updates it to v2.4.0 (see [SuSFS v2.4.0](#-susfs-v240)) | `CONFIG_KSU_SUSFS_*` |
| **Baseband Guard** | [vc-teahouse](https://github.com/vc-teahouse/Baseband-guard) | LSM that blocks writes to the modem/baseband partitions | `CONFIG_BBG=y` |
| **NoMount** | [maxsteeel](https://github.com/maxsteeel/nomount) | Builds the NoMount driver into the kernel (VFS path redirection, nothing in `/proc/mounts`). OriginSU v4.3.0 can control it through *OriginMount*; BakaSU manager integration has not been verified | `CONFIG_NOMOUNT=y` |
| **Droidspaces** | [ravindu644](https://github.com/ravindu644/Droidspaces-OSS) | Namespace/cgroup isolation to run isolated Android/Linux spaces | `CONFIG_NAMESPACES`, `CONFIG_*_NS`, cgroups, netfilter… |
| **TCP BBR + fq** | [Google](https://github.com/google/bbr) (mainline since 4.9) | BBR congestion control **set as default** (the stock Coconutat kernel uses CUBIC), with the **`fq` qdisc as default** so BBR can pace its traffic | `CONFIG_DEFAULT_BBR=y`, `CONFIG_NET_SCH_FQ=y`, `CONFIG_DEFAULT_FQ=y` |
| **Optimizations** | this repo | ZRAM in lz4 by default, power-efficient workqueues, TTL/HL targets, `vm.swappiness` and per-cgroup swappiness up to 200, Cortex-A55 tuning (see [Optimizations](#-optimizations)) | `CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y`… |

> **Droidspaces note:** namespaces, overlayfs, networking and the pids, freezer, devices and net cgroups are enabled, and **per-container memory/swap limits work** (`CONFIG_MEMCG` is on in this kernel). The **CPU limit (`--cpus`) is not available**: it needs `CONFIG_CFS_BANDWIDTH`, which this tree only allows when Qualcomm's **WALT** scheduler is off — and WALT drives CPU frequency and task placement on SDM845, so turning it off would cost performance and battery. The build asks for it, Kconfig drops it, and Droidspaces simply greys out `--cpus`. Droidspaces only ships its two "fix" patches for newer trees (the cgroup one targets `kernel/cgroup/cgroup.c`, which exists only since 4.15). On this **4.9** tree the same function and symbols live in `kernel/cgroup.c`, so the build first tries each patch as-is and, if it doesn't apply, retries it with the path rewritten to the 4.9 location — verified to apply cleanly on the Coconutat 4.9 tree. If a future kernel ever changes enough that a patch no longer fits, it is skipped with a `::warning` and the build still completes with the configs enabled. If you use Droidspaces with a systemd-based Linux space, mask the `NetworkManager` and `wpa_supplicant` services inside that space (as noted upstream).

> **TCP BBR note:** on 4.9, BBR has no TCP-internal pacing (it only arrived in 4.13): the kernel's own `tcp_bbr.c` states that BBR **must** run on the `fq` qdisc, otherwise it does not work properly. This 4.9 tree had no config option to make `fq` the default qdisc, so the build **backports the small upstream change** that adds it (`CONFIG_NET_SCH_DEFAULT` + a boot-time call to the existing `qdisc_set_default()`, identical to mainline) and selects `fq`. No `service.d` script is needed any more. The build fails if the final `.config` does not have `CONFIG_DEFAULT_TCP_CONG="bbr"` and `CONFIG_DEFAULT_NET_SCH="fq"`.
>
> Check on the phone (root shell):
> ```sh
> cat /proc/sys/net/ipv4/tcp_congestion_control   # bbr
> cat /proc/sys/net/core/default_qdisc            # fq
> tc qdisc show dev wlan0                          # fq on Wi-Fi (if tc is available)
> ```

### ⚡ Optimizations

The Coconutat kernel already has WireGuard, PSI and memory cgroups. This builder adds what is missing (low risk; the build fails if a config is not in the final `.config`):

| Change | Effect | How to use / check |
|---|---|---|
| `CONFIG_WQ_POWER_EFFICIENT_DEFAULT=y` | Background kernel work runs on CPUs that are already awake instead of waking an idle core | `cat /sys/module/workqueue/parameters/power_efficient` → `Y` |
| **ZRAM in lz4 by default** (was `lzo`) + `CONFIG_CRYPTO_ZSTD=y` | The ROM does not choose a ZRAM algorithm, so the kernel default applies; it was hard-coded to the old `lzo`. **lz4** is much faster, above all to bring apps back from ZRAM, for a slightly lower compression ratio that does not matter with 6–8 GB of RAM. zstd (better compression, more CPU) stays available | `cat /sys/block/zram0/comp_algorithm` → `[lz4]` |
| `CONFIG_IP_NF_TARGET_TTL=y`, `CONFIG_IP6_NF_TARGET_HL=y` | iptables `TTL` / `HL` targets, e.g. to hide tethering from the carrier | `iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64` |
| `vm.swappiness` max 100 → 200 | Lets you make the kernel prefer swapping to ZRAM over dropping app caches. The ROM's value (100) is unchanged | `echo 160 > /proc/sys/vm/swappiness` |
| Per-cgroup swappiness max 100 → 200 | The 4.9 kernel refused memory-cgroup swappiness values above 100 (newer kernels allow 200). Now a module or a manual tweak can set e.g. 150 on a group in `/dev/memcg`. The ROM itself sets none: LineageOS' `init.qcom.rc` tries to give a `bg` group swappiness 140, but that group is never created on LineageOS 22 (memcg lives in `/dev/memcg`), so nothing changes by default | `mkdir /dev/memcg/t; echo 150 > /dev/memcg/t/memory.swappiness; cat /dev/memcg/t/memory.swappiness; rmdir /dev/memcg/t` → `150` |
| `-mcpu=cortex-a55` | Code tuned for the cores of the Snapdragon 845 (Cortex-A75 + Cortex-A55, ARMv8.2) | — |

Not included on purpose: LTO (experimental on this 4.9 tree, and it can rename internal symbols that root/hiding code looks up at boot), MGLRU (too large to backport to 4.9) and a newer LZ4 (its new decompression loop was measured slower on in-order Cortex-A53 cores and with Clang).

---

## ✅ Supported Devices & ROMs

One zip is built per device. **Flash only the zip whose codename matches your phone** (each zip refuses to install on any other device).

| Device | Codename | Test status |
|---|---|---|
| Xiaomi Mi 8 | `dipper` | Current build target; previous builds tested by the maintainer |

> ### 🧪 Testing status
> The workflow currently builds **only `dipper`**. Previous builds of this device were tested, but the current BakaSU configuration has not been validated on hardware. Back up your stock `boot` and test before daily use.

The kernel comes from the LineageOS kernel tree, so it works with ROMs built on the LineageOS `sdm845` 4.9 kernel. Each build records its LineageOS version in the zip name (e.g. `LOS22.2`) — **use a build that matches the LineageOS / /e/OS version on your phone**. Currently:

| ROM | Status | Links |
|---|---|---|
| **LineageOS 22.2** (Android 15) | ✅ Built from the LineageOS source | [LineageOS wiki](https://wiki.lineageos.org/devices/dipper/) |
| **/e/OS** (Android 15, community) | ✅ /e/OS is built on LineageOS 22 | [/e/OS docs](https://doc.e.foundation/devices/dipper/) |

Other ROMs may work only if they are built on the LineageOS `sdm845` 4.9 kernel and the same vendor firmware, and they are not tested. ROMs built on another kernel tree, such as **crDroid**, are **not** compatible: LineageOS and crDroid kernels do not boot on each other's ROMs on SDM845. Stock MIUI is **not** supported.

> Make sure your device is on the vendor firmware LineageOS asks for (see the *"Update to a newer vendor firmware version"* section of the LineageOS wiki).

---

## 📦 Which zip should I download?

Each build produces a zip for the selected device. The filename identifies the root solution:

```
sdm845-<codename>-LOS<lineage>-BakaSU-susfs-<date>.zip
sdm845-<codename>-LOS<lineage>-OriginSU-<version>-susfs-<date>.zip
```

For example `sdm845-dipper-LOS22.2-BakaSU-susfs-20261115.zip` for the Mi 8. BakaSU zips and configs are available in both the GitHub Release and workflow run's **Artifacts** section. OriginSU zips and configs are available only from the workflow run's artifacts; its manager APK is not included. Only the **SuSFS** variant is built: the Coconutat tree has its KernelSU hooks written for SuSFS.

### Getting a BakaSU build

Run **Build Xiaomi SDM845 Kernels with BakaSU or OriginSU** from the Actions tab with `root` set to `bakasu` (the default). Download the `sdm845-dipper-bakasu` artifact from the run, or the latest `bakasu-los<lineage>` GitHub Release, and extract the flashable zip and build config. Back up `boot`, then flash the zip in recovery or with `adb sideload`. Use the manager/version recommended by BakaSU; no manager is attached to the release. NoMount is built into the kernel, but BakaSU manager control of its driver has not been verified here.

---

## 🔧 Installing an OriginSU artifact

0. **Coming from Magisk, APatch, BakaSU or another KernelSU?** Read [this warning](#️-coming-from-another-root-solution) first.
1. Back up your stock `boot` partition (from TWRP/OrangeFox, or keep the `boot.img` from your LineageOS / /e/OS zip).
2. Download the `sdm845-dipper-originsu` artifact from the workflow run and extract the zip matching your device codename.
3. Reboot to recovery and flash it, or use `adb sideload <file>.zip`.
4. Reboot to system.
5. Install the **OriginSU manager** APK matching the selected OriginSU tag. For the default `v4.3.0` build, get `OriginSU_v4.3.0_…-arm64-v8a-release.apk` from the [KazeSU legacy release](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0). Do **not** install the KazeSU v1.x manager with this kernel.
6. **Pick a module mount mode** in the manager's **OriginMount** screen — otherwise modules that change `/system` are not mounted (see [Modules](#-modules)).

> **Updating the ROM?** A LineageOS / /e/OS OTA update replaces the `boot` partition, so the custom kernel is removed. Flash the matching zip again after each ROM update.

### 🧱 Modules

For OriginSU builds, module mounting is controlled in the manager's **OriginMount** screen (the choice applies from the next boot):

| Mode | What it does |
|---|---|
| `auto` (default) | Uses a metamodule if one is installed, otherwise Magic Mount if you enabled it, otherwise **no mounting** |
| `nomount` | Uses the **NoMount** driver built into this kernel: module files are injected without real mounts, so nothing shows in `/proc/mounts` |
| `builtin` | OriginSU's built-in Magic Mount |
| `metamodule` | Delegates to an installed metamodule (e.g. meta-overlayfs) |
| `off` | No module mounting (scripts, `sepolicy.rule` and `system.prop` still work) |

NoMount is already in the kernel and OriginSU talks to it directly: **do not install the separate NoMount module zip** on top. If your modules are installed but their files don't show up, you are probably in `auto` mode with nothing to mount them: pick a mode.

### 🔍 Checking the kernel after flashing

Root shell (`adb shell` then `su`, or a terminal app):

```sh
uname -r                                              # 4.9.x
cat /proc/sys/net/ipv4/tcp_congestion_control         # bbr
cat /proc/sys/net/core/default_qdisc                  # fq
cat /sys/block/zram0/comp_algorithm                   # [lz4] lzo ... zstd
cat /proc/net/ip_tables_targets | grep TTL            # TTL
cat /proc/net/ip6_tables_targets | grep HL            # HL
cat /sys/module/workqueue/parameters/power_efficient  # Y
dmesg | grep -iE "nomount|baseband|susfs" | head      # NoMount, Baseband Guard, SuSFS lines
ls /proc/self/ns                                      # user net pid uts ipc ... (Droidspaces)
```

For an OriginSU build, verify the manager reports **v4.3.0**, SuSFS **v2.4.0**, and NoMount as available in **OriginMount**. For BakaSU, use its own manager and verify its supported features; NoMount manager integration has not been tested. Then check Wi-Fi, mobile data and calls, camera, Bluetooth, fingerprint, audio, charging and overnight battery drain. If something is wrong, save the logs right after reproducing it: `dmesg > /sdcard/dmesg.txt` and `logcat -b all -d > /sdcard/logcat.txt`.

### ↩️ Restore the stock kernel

```sh
su -c rm -rf /data/adb/*        # remove root data first
adb reboot bootloader
fastboot flash boot stock_boot.img
fastboot reboot
```

---

## ⚙️ How the Build Works

The workflow lives in [`.github/workflows/build.yml`](.github/workflows/build.yml):

```
check  ──►  build (dipper)  ──►  release (BakaSU only)
```

1. **check**
   - For OriginSU builds, validates the selected tag (`v4.3.0` by default; only `v4.x` tags are accepted). BakaSU builds do not depend on an OriginSU release.
   - Picks the kernel branch: among the `lineage-<version>[-suffix]` branches of the Coconutat tree, the highest LineageOS version first, and for the same version the most recently updated branch. A branch is kept only if it contains SuSFS, its inline hooks and the shared SDM845 configs.
2. **build** (currently `dipper` only)
   - Clones the selected Coconutat SDM845 kernel branch.
   - Downloads **AOSP Clang r547379**, the toolchain recommended by the kernel's author.
   - Installs the selected root solution: BakaSU by default, or OriginSU at the exact selected tag with its 4.9 compatibility fixes.
   - Applies the selected SuSFS version and enables the shared features (Baseband Guard, NoMount, Droidspaces, TCP BBR + `fq` as default) and [optimizations](#-optimizations) for either root solution. The NoMount API compatibility check applies to OriginSU.
   - Packages like Coconutat: osm0sis AnyKernel3 + the tree's own `tools/AnyKernel3_sh/anykernel.sh`, with both `Image.gz` and `Image.gz-dtb`. AnyKernel3 flashes `Image.gz` and **keeps the DTB of your ROM's `boot.img`**. Each zip installs on its own device only.
3. **release** (BakaSU only) — publishes/replaces `bakasu-los<lineage>` with the BakaSU zip and config. OriginSU builds remain downloadable from the Actions run artifacts.

### Schedule

| Trigger | Behaviour |
|---|---|
| Manual (`Run workflow`) | Builds BakaSU by default; you can select OriginSU, pick a kernel branch, choose the SuSFS version, toggle KPM and choose whether to publish the BakaSU release |

There is no scheduled build.

A manual BakaSU run with `publish` enabled for a LineageOS version that is already released **replaces** that release (useful to pick up kernel updates). OriginSU runs do not publish or replace a release.

---

## 🚀 Using This Repository (forks)

1. Fork the repository.
2. Go to **Settings → Actions → General** and make sure Actions are enabled. **Read and write** workflow permissions are needed to publish BakaSU releases.
3. Run **Build Xiaomi SDM845 Kernels with BakaSU or OriginSU** manually. BakaSU builds publish a release by default; OriginSU builds produce run artifacts only.

### Manual run options

| Input | Default | Description |
|---|---|---|
| `originsu_ref` | `v4.3.0` | OriginSU tag to build (`v4.x` only) |
| `kernel_branch` | *(latest LineageOS with SuSFS)* | Force a branch of the Coconutat kernel tree (e.g. `lineage-22.2-ReSukiSU`) |
| `root` | `bakasu` | Select `bakasu` (default) or `originsu`; both receive the shared SuSFS update and kernel features. OriginSU is artifact-only |
| `susfs` | `v2.4.0` | `v2.4.0` updates SuSFS to JackA1ltman's 4.9 port (pinned commit); `tree` keeps the tree's v2.3.0 |
| `kpm` | `false` | Enable `CONFIG_KPM` (experimental on 4.9 kernels) |
| `publish` | `true` | Publish or replace the BakaSU GitHub Release (ignored for OriginSU builds) |

---

## 🫥 SuSFS v2.4.0

The Coconutat tree ships SuSFS **v2.3.0** (JackA1ltman's 4.9 port). The build moves it to **v2.4.0**, the same port at a pinned commit: between the two, only `fs/susfs.c`, the two SuSFS headers and the SuSFS hooks in `fs/namei.c`, `fs/readdir.c` and `fs/Makefile` change; every other hook is identical. The build stops if the tree is not exactly the expected v2.3.0, and `susfs` = `tree` keeps v2.3.0.

What changes for you, with the **susfs4ksu** module (sidex15):

| | v2.3.0 | v2.4.0 |
|---|---|---|
| Who cannot see the *sus paths* | Apps root is unmounted for | **Every app** (uid 10000–19999), except the ones you allow |
| Allow an app to see them | — | **sus_path uid filter** (new command `CMD_SUSFS_SET_SUS_PATH_UID_FILTER`): a list of package names |
| After an app is reinstalled | — | Its new uid is picked up automatically (when `packages.list` is rewritten) |

OriginSU v4.3.0 predates v2.4.0, so its build wires that one new command into OriginSU's SuSFS dispatcher and refreshes the allowed apps' uids on package changes. For BakaSU, the workflow leaves the setup-provided integration unchanged; manager-side support for every v2.4.0 feature should be confirmed with BakaSU. Root processes (`su`) always see everything.

---

## 🛠️ Troubleshooting

| Problem | What to do |
|---|---|
| Build fails with "the tree's SuSFS is not the expected base port" | Coconutat updated SuSFS in the tree: set `SUSFS_BASE_REF` in the workflow to the matching JackA1ltman commit, or run with `susfs` = `tree` |
| SuSFS v2.4.0 problem (an app no longer sees a file it needs, detection) | Run again with `susfs` = `tree` (v2.3.0) and compare, then report it |
| Build fails in *Install the root solution* with "hook … missing" | The kernel tree changed its SuSFS hooks — check the Coconutat repo, or pin an older `kernel_branch` |
| `::warning` about a Droidspaces patch being skipped | Expected on 4.9 — the upstream fix doesn't fit this tree; the build continues with the Droidspaces configs enabled (see the Droidspaces note above) |
| Droidspaces greys out `--cpus` | Expected: the CPU limit needs `CONFIG_CFS_BANDWIDTH`, which Qualcomm's WALT scheduler excludes on this kernel. Memory/swap limits work |
| Build fails with `… missing from the final config` | A config this builder relies on was dropped by Kconfig — check the *Optimizations* lines of the *Extra features* step |
| OriginSU modules installed but their files don't show up | Pick a mode in **OriginMount** (`auto` mounts nothing without a metamodule or Magic Mount) — see [Modules](#-modules) |
| Bootloop after flashing on crDroid or another non-LineageOS ROM | Expected: this LineageOS kernel does not boot on crDroid-family ROMs. Restore your `boot` backup |
| Build fails in *Extra features* (Baseband Guard / NoMount setup) | Check `build-<device>.log` in the run artifacts; the setup script of one of the modules failed to download or patch the tree |
| The device build fails | Check `build-dipper.log` in the run artifacts |
| Build fails with KPM errors | Re-run manually with `kpm` disabled |
| *check* fails with "No LineageOS branch with SuSFS found" | The Coconutat tree changed its branch layout — run manually with `kernel_branch` set |
| Zip refuses to install ("unsupported device") | You flashed the wrong device's zip — pick the one whose codename matches your phone, and make sure your ROM matches the LineageOS version of the release |
| Device stuck on boot logo, or reboots straight to fastboot | Restore the stock `boot.img`. The last kernel log is kept in RAM: boot into TWRP right away and get `/sys/fs/pstore/console-ramoops-0` (e.g. `adb pull /sys/fs/pstore/`), then open an issue with it |
| Boots, but no Wi-Fi / mobile network / camera | Almost always a module conflict — see [Coming from another root solution?](#️-coming-from-another-root-solution) below |
| Root gone after a ROM update | Normal: OTA updates replace `boot`. Flash the zip again |
| Build fails with "fq is not the default qdisc" | The `NET_SCH_DEFAULT` backport no longer applies to the kernel tree — check the *TCP BBR + fq* lines of the *Extra features* step |

### ⚠️ Coming from another root solution?

If you previously used **Magisk, APatch, BakaSU, KernelSU, SukiSU or ReSukiSU**, review and remove incompatible modules before switching root solutions. For an OriginSU build, old modules in `/data/adb/modules` may be loaded by OriginSU.

OriginSU ships its **own Zygisk (OriginZygisk)**. On OriginSU builds, old modules — especially another Zygisk implementation (ReZygisk, NeoZygisk, Zygisk Next) or Zygisk-based modules such as LSPosed — can conflict with it. The typical symptom is a phone that **boots fine but has no Wi-Fi, no mobile network and/or no camera**. For BakaSU builds, follow BakaSU's migration and module compatibility guidance.

For OriginSU, uninstall incompatible modules and reinstall them one by one. For BakaSU, follow BakaSU's migration guidance instead.

The following cleanup commands are for an OriginSU migration and remove the selected modules:

From a PC (use the interactive shell — it also works in Windows PowerShell):

```sh
adb shell
su
for m in /data/adb/modules/*/; do touch "$m/remove"; done
rm -rf /data/adb/modules_update/*
reboot
```

Each module's `uninstall.sh` runs on the next boot. If some folders remain afterwards:

```sh
adb shell
su
rm -rf /data/adb/modules/*
reboot
```

After cleaning up for OriginSU:

- Reinstall your modules **one at a time**, checking Wi-Fi, network and camera after each reboot.
- Use **only one Zygisk**: keep OriginZygisk and don't install another Zygisk module.
- Reinstall Zygisk-based modules (e.g. LSPosed) last.
- Optional cleanup of leftovers from other root solutions, e.g. APatch: `rm -rf /data/adb/ap /data/adb/apd`.

---

## 🙏 Credits & Acknowledgments

This project relies on the incredible work of the following open-source projects and developers:

* **[Coconutat](https://github.com/Coconutat/android_kernel_xiaomi_sdm845_lineageos_Exp)** — For the Xiaomi SDM845 LineageOS kernel tree with SuSFS already integrated (`lineage-*` branches).
* **[LineageOS](https://github.com/LineageOS/android_kernel_xiaomi_sdm845)** — For the original Xiaomi SDM845 kernel source and device support.
* **[/e/OS](https://e.foundation/)** — For the de-Googled Android 15 builds.
* **[BakaSU](https://github.com/Baka-SU/BakaSU)** — For the default kernel-based root solution.
* **[OriginSU](https://github.com/KazeSU/KazeSU/releases/tag/v4.3.0)** (now [KazeSU](https://github.com/KazeSU/KazeSU)) — For the optional kernel-based root solution.
* **[riarumoda](https://github.com/riarumoda/perf_neon-builder)** — For perf_neon-builder, whose feature integration (Baseband Guard, NoMount, Droidspaces) this build reuses.
* **[osm0sis](https://github.com/osm0sis/AnyKernel3)** — For AnyKernel3.
* **[simonpunk](https://gitlab.com/simonpunk/susfs4ksu)** — For SuSFS.
* **[vc-teahouse](https://github.com/vc-teahouse/Baseband-guard)** — For Baseband Guard.
* **[maxsteeel](https://github.com/maxsteeel/nomount)** — For NoMount.
* **[ravindu644](https://github.com/ravindu644/Droidspaces-OSS)** — For Droidspaces.
* **[JackA1ltman](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd)** — For the non-GKI SuSFS work the kernel tree builds on.
* **[AOSP](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/)** — For the Clang toolchain.
* **[tiann/KernelSU](https://github.com/tiann/KernelSU)** — The project BakaSU and OriginSU are based on.

## 📄 License

Kernel sources are licensed under **GPL-2.0**. See each upstream project for its own license.
