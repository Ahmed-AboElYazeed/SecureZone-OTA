## To Clone the project

```bash
git clone --recurse-submodules https://github.com/Ahmed-AboElYazeed/SecureZone-OTA.git
```

If you already cloned:

```bash
git submodule update --init --recursive
```

# OTA Update System — Version 1

## Direct Laptop → Yocto RPi 3B+ via SOME/IP / CommonAPI

------

## Overview

Phase 1 establishes the baseline OTA (Over-The-Air) update pipeline between a **Linux laptop** and a **Yocto-based Raspberry Pi 3B+** using **SOME/IP** as the communication protocol and **CommonAPI C++** as the middleware layer. No intermediate gateway is involved — the laptop acts as both the update source and the SOME/IP client.

The system performs a full **A/B rootfs partition update**: the laptop streams a new rootfs image in chunks directly to the inactive partition on the RPi, which then verifies integrity, switches the boot slot by rewriting `cmdline.txt`, and reboots into the new rootfs.

------

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Phase 1 Architecture                     │
│                                                                 │
│   ┌──────────────┐          WiFi / Ethernet          ┌──────────────────────┐  │
│   │    Laptop    │  ──────────────────────────────►  │   Raspberry Pi 3B+   │  │
│   │  (Ubuntu)    │       SOME/IP over TCP/UDP         │  (Yocto Linux)       │  │
│   │              │                                    │                      │  │
│   │  ota_client  │  AnnounceUpdate ──────────────►   │  OtaUpdateService    │  │
│   │  (C++ binary)│  SendChunk(s)   ──────────────►   │  (systemd daemon)    │  │
│   │              │  FinalizeUpdate ──────────────►   │                      │  │
│   │              │  ◄──────────── StatusEvents        │                      │  │
│   └──────────────┘                                    └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

------

## Hardware

| Component | Details                                            |
| --------- | -------------------------------------------------- |
| Laptop    | Ubuntu 22.04 LTS, x86_64                           |
| Target    | Raspberry Pi 3B+                                   |
| Target OS | Yocto Linux (Scarthgap, aarch64)                   |
| Network   | Both devices on same subnet `192.168.1.x` via WiFi |
| RPi IP    | `192.168.1.100` (wlan0)                            |
| Laptop IP | `192.168.1.3` (enp3s0)                             |

------

## SD Card Partition Layout

```
┌──────────┬────────────┬────────────┬──────────────┐
│  boot    │  rootfs-a  │  rootfs-b  │    mydata    │
│ (FAT32)  │   (ext4)   │   (ext4)   │    (ext4)    │
│  256MB   │   3584MB   │   3584MB   │   512MB      │
└──────────┴────────────┴────────────┴──────────────┘
  /dev/mmcblk0p1   p2            p3           p4
```

| Partition | Label    | Mount     | Purpose                              |
| --------- | -------- | --------- | ------------------------------------ |
| p1        | boot     | `/boot`   | Kernel, DTB, `cmdline.txt` (FAT32)   |
| p2        | rootfs-a | `/`       | Active rootfs slot A                 |
| p3        | rootfs-b | —         | Standby slot, written during OTA     |
| p4        | mydata   | `/mydata` | Persistent state, logs, version info |

The WKS file (`meta-myapp/wic/rpi-ab.wks`) defines this layout and is used by the Yocto `wic` image tool at build time.

------

## Boot Slot Switching

The active boot slot is controlled purely through `/boot/cmdline.txt`. No U-Boot environment variables are used.

**Slot A active:**

```
dwc_otg.lpm_enable=0 console=serial0,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait
```

**Slot B active:**

```
dwc_otg.lpm_enable=0 console=serial0,115200 root=/dev/mmcblk0p3 rootfstype=ext4 rootwait
```

The `OtaUpdateServiceImpl` remounts `/boot` as read-write, rewrites `cmdline.txt`, calls `sync()`, remounts read-only, then reboots.

------

## Software Stack

### On the RPi (Yocto Target)

| Component                 | Version | Location                                           |
| ------------------------- | ------- | -------------------------------------------------- |
| vsomeip                   | 3.5.5   | `/usr/lib/libvsomeip3.so`                          |
| CommonAPI Core Runtime    | 3.2.4   | `/usr/lib/libCommonAPI.so`                         |
| CommonAPI SOME/IP Runtime | 3.2.4   | `/usr/lib/libCommonAPI-SomeIP.so`                  |
| Boost                     | 1.82.0  | `/usr/lib/libboost_*.so`                           |
| OpenSSL                   | 3.x     | `/usr/lib/libssl.so`                               |
| OTA Update Agent          | custom  | `/usr/bin/ota_update_service`                      |
| vsomeip config            | —       | `/etc/vsomeip/vsomeip-ota.json`                    |
| systemd unit              | —       | `/usr/lib/systemd/system/ota-update-agent.service` |
| Update state file         | —       | `/mydata/update-status.json`                       |

### On the Laptop (Client)

| Component                 | Version | Location                                |
| ------------------------- | ------- | --------------------------------------- |
| vsomeip                   | 3.5.5   | `/usr/local/lib/libvsomeip3.so`         |
| CommonAPI Core Runtime    | 3.2.4   | `/usr/local/lib/libCommonAPI.so`        |
| CommonAPI SOME/IP Runtime | 3.2.4   | `/usr/local/lib/libCommonAPI-SomeIP.so` |
| Boost                     | 1.82.0  | `/usr/local/lib/libboost_*.so`          |
| OTA Client Binary         | custom  | `build-laptop/laptop/ota_client`        |
| vsomeip config            | —       | `laptop/vsomeip-ota-laptop.json`        |

------

## SOME/IP Service Definition

The service interface is defined using **Franca IDL** and code-generated with the CommonAPI generators.

### `fidl/OtaUpdate.fidl`

```
package com.myapp.ota

interface OtaUpdate {
    version { major 1 minor 0 }

    method AnnounceUpdate {
        in  { String newVersion, UInt64 imageSize, String sha256Hash }
        out { Boolean accepted, String message }
    }

    method SendChunk {
        in  { UInt64 offset, ByteBuffer data }
        out { Int64 nextOffset }
    }

    method FinalizeUpdate {
        in  { String sha256Hash }
        out { Boolean success, String message }
    }

    broadcast UpdateStatus {
        out { String status, UInt32 percent, String message }
    }
}
```

### `fidl/OtaUpdate.fdepl` — SOME/IP deployment

| Parameter                | Value                           |
| ------------------------ | ------------------------------- |
| Service ID               | `0x1101`                        |
| Instance ID              | `0x0001`                        |
| AnnounceUpdate method ID | `0x0001`                        |
| SendChunk method ID      | `0x0002`                        |
| FinalizeUpdate method ID | `0x0003`                        |
| UpdateStatus event ID    | `0x8001`                        |
| Transport                | TCP (reliable), port `30510`    |
| Service Discovery        | UDP multicast `224.0.0.1:30490` |

------

## OTA Update Protocol Flow

```
Laptop (Client)                          RPi (Server — OtaUpdateServiceImpl)
───────────────                          ───────────────────────────────────

[startup]                                [startup]
                                         resolveInactiveSlot() → reads cmdline.txt
                                         opens SOME/IP service, starts SD broadcasts

buildProxy → isAvailable()          ←──  SD multicast every 2s

AnnounceUpdate(version,             ──►  openPartition(/dev/mmcblk0p3)
  imageSize, sha256)                      updateStatusFile("in_progress")
                                    ◄──  reply(true, "Ready")

loop until EOF:
  SendChunk(offset, data)           ──►  lseek64 + write to /dev/mmcblk0p3
                                    ◄──  reply(nextOffset)

FinalizeUpdate(sha256)              ──►  fsync + closePartition
                                         computeSHA256() ← reads 3.5GB from p3
                                         compare hashes
                                         switchBootSlot() → rewrite cmdline.txt
                                         updateStatusFile("complete")
                                         write /mydata/boot_pending
                                    ◄──  reply(true, "rebooting in 5s")
                                         sleep(5s) → reboot

                                    [RPi reboots into new slot]
```

**Async events** (RPi → Laptop, fire-and-forget via SOME/IP broadcast):

| Trigger                 | Status        | Percent | Message            |
| ----------------------- | ------------- | ------- | ------------------ |
| AnnounceUpdate accepted | `in_progress` | 0       | "Ready for chunks" |
| Every 1% written        | `in_progress` | 1–99    | "X / Y bytes"      |
| FinalizeUpdate complete | `complete`    | 100     | "Rebooting in 5s"  |
| Any failure             | `failed`      | 0       | Error description  |

------

## Update State File

Located at `/mydata/update-status.json`. Persists across reboots on the `mydata` partition.

```json
{
    "active_slot"   : "a",
    "pending_slot"  : "",
    "slot_a_dev"    : "/dev/mmcblk0p2",
    "slot_b_dev"    : "/dev/mmcblk0p3",
    "boot_part"     : "/dev/mmcblk0p1",
    "boot_mount"    : "/boot",
    "cmdline_path"  : "/boot/cmdline.txt",
    "version_a"     : "1.0.0",
    "version_b"     : "",
    "status"        : "idle",
    "last_update"   : ""
}
```

| Field                       | Description                                 |
| --------------------------- | ------------------------------------------- |
| `active_slot`               | Currently booted slot (`a` or `b`)          |
| `slot_a_dev` / `slot_b_dev` | Block device paths for each slot            |
| `version_a` / `version_b`   | Version string of rootfs in each slot       |
| `status`                    | `idle`, `in_progress`, `complete`, `failed` |

------

## Yocto Build

### Layers Used

```
poky
meta-raspberrypi
meta-openembedded (meta-oe, meta-networking, meta-python)
meta-myapp                ← custom layer
meta-qt6
```

### Custom Recipes in `meta-myapp`

| Recipe                                          | Purpose                                  |
| ----------------------------------------------- | ---------------------------------------- |
| `recipes-someip/vsomeip3/vsomeip3_git.bb`       | Cross-compiles vsomeip 3.5.5 for aarch64 |
| `recipes-capicxx-core-runtime/`                 | Cross-compiles CommonAPI Core Runtime    |
| `recipes-capicxx-someip-runtime/`               | Cross-compiles CommonAPI SOME/IP binding |
| `recipes-support/boost/boost_1.82.0.bb`         | Boost 1.82.0                             |
| `recipes-ota-update/ota-update-agent/`          | Builds and installs the OTA daemon       |
| `recipes-ota-update/update-status-init/`        | Initialises state file on first boot     |
| `recipes-core/images/core-aboelyazeed-image.bb` | Custom image recipe                      |

### Build Commands

```bash
cd ~/yocto/build-rpi
source ../poky/oe-init-build-env

# Full image build
bitbake core-aboelyazeed-image

# OTA agent only (faster iteration)
bitbake -c cleansstate ota-update-agent
bitbake ota-update-agent

# Flash to SD card
sudo bmaptool copy \
  tmp/deploy/images/raspberrypi3-64/core-aboelyazeed-image-raspberrypi3-64.wic.bz2 \
  /dev/sdX
```

### WKS File (`meta-myapp/wic/rpi-ab.wks`)

```bash
part /boot   --source bootimg-partition --ondisk mmcblk0 --fstype=vfat \
             --label boot --active --align 4096 --fixed-size 256M
part /       --source rootfs --ondisk mmcblk0 --fstype=ext4 \
             --label rootfs-a --align 4096 --fixed-size 3584M
part         --ondisk mmcblk0 --fstype=ext4 \
             --label rootfs-b --align 4096 --fixed-size 3584M
part /mydata --ondisk mmcblk0 --fstype=ext4 \
             --label mydata --align 4096 --fixed-size 512M
```

------

## Laptop Client Build

```bash
cd ~/ITI_Files/linux/ota_update
mkdir build-laptop && cd build-laptop

cmake .. \
  -DBUILD_TARGET=laptop \
  -DCAPI_PREFIX=/usr/local

make -j$(nproc)
```

### Running the Client

```bash
# Add multicast route (required for SOME/IP service discovery)
sudo ip route add 224.0.0.0/4 dev enp3s0

# Run the update
cd build-laptop/laptop

VSOMEIP_CONFIGURATION=../vsomeip-ota-laptop.json \
VSOMEIP_APPLICATION_NAME=OtaUpdateClient \
  ./ota_client /path/to/new-rootfs.ext4 2.0.0
```

------

## vsomeip Configuration

### RPi (`/etc/vsomeip/vsomeip-ota.json`)

```json
{
    "unicast" : "192.168.1.100",
    "netmask" : "255.255.255.0",
    "network" : "wlan0",
    "applications" : [
        { "name" : "OtaUpdateService", "id" : "0x1100" }
    ],
    "services" : [
        {
            "service"  : "0x1101",
            "instance" : "0x0001",
            "reliable" : { "port" : "30510", "enable-magic-cookies" : "false" }
        }
    ],
    "routing" : "OtaUpdateService",
    "service-discovery" : {
        "enable"    : "true",
        "multicast" : "224.0.0.1",
        "port"      : "30490",
        "protocol"  : "udp"
    }
}
```

### Laptop (`laptop/vsomeip-ota-laptop.json`)

```json
{
    "unicast" : "192.168.1.3",
    "netmask" : "255.255.255.0",
    "applications" : [
        { "name" : "OtaUpdateClient", "id" : "0x1200" }
    ],
    "routing" : "OtaUpdateClient",
    "service-discovery" : {
        "enable"    : "true",
        "multicast" : "224.0.0.1",
        "port"      : "30490",
        "protocol"  : "udp"
    }
}
```

------

## Deploying Changes During Development

Instead of a full reflash, push binaries directly to the running RPi:

```bash
# Sync source into recipe files
rsync -av --delete \
  ~/ITI_Files/linux/ota_update/ \
  ~/yocto/meta-myapp/recipes-ota-update/ota-update-agent/files/ota_update/ \
  --exclude build-laptop/ --exclude build-rpi/ --exclude ".git/"

# Build agent only
cd ~/yocto/build-rpi
bitbake -c cleansstate ota-update-agent && bitbake ota-update-agent

# Push binary
SYSROOT=~/yocto/shared_build/tmp/work/cortexa53-poky-linux/ota-update-agent/1.0/image

scp $SYSROOT/usr/bin/ota_update_service       root@192.168.1.100:/usr/bin/
scp $SYSROOT/etc/vsomeip/vsomeip-ota.json     root@192.168.1.100:/etc/vsomeip/
scp $SYSROOT/usr/lib/systemd/system/ota-update-agent.service \
                                               root@192.168.1.100:/usr/lib/systemd/system/

ssh root@192.168.1.100 \
  "systemctl daemon-reload && systemctl restart ota-update-agent"
```

------

## Monitoring

### Watch RPi OTA service live

```bash
ssh root@192.168.1.100 "journalctl -u ota-update-agent -f"
```

### Expected log during a successful update

```
[OTA] Inactive slot (write target): /dev/mmcblk0p3
[OTA] Service registered. Waiting for laptop...
[OTA] AnnounceUpdate: version=2.0.0 size=3758096384 hash=abc123...
[OTA] Currently booted from slot A → will write to slot B
[OTA] SendChunk at offset 0 ... (repeated)
[OTA] FinalizeUpdate: starting...
[OTA] Computing SHA256 of written partition (3584 MB)...
[OTA] Expected : abc123...
[OTA] Computed : abc123...
[OTA] SHA256 OK
[OTA] Switching boot slot...
[OTA] cmdline.txt: slot A → B
[OTA] Boot slot switched successfully
[OTA] Pending boot flag written
[OTA] Reply sent to QNX
[OTA] Rebooting now.
```

### Verify active slot after reboot

```bash
ssh root@192.168.1.100 "cat /boot/cmdline.txt && mount | grep 'on / '"
# Should show: root=/dev/mmcblk0p3 and /dev/mmcblk0p3 on /
```

------

## Project Source Tree

```
ota_update/
├── CMakeLists.txt                   ← root cmake (dispatches by BUILD_TARGET)
├── fidl/
│   ├── OtaUpdate.fidl               ← Franca IDL service definition
│   └── OtaUpdate.fdepl              ← SOME/IP deployment spec
├── src-gen/                         ← CommonAPI generator output (never edit)
│   └── v1/com/myapp/ota/
│       ├── OtaUpdate.hpp
│       ├── OtaUpdateProxy.hpp
│       ├── OtaUpdateStubDefault.hpp
│       ├── OtaUpdateSomeIPProxy.cpp/hpp
│       ├── OtaUpdateSomeIPStubAdapter.cpp/hpp
│       └── OtaUpdateSomeIPDeployment.cpp/hpp
├── rpi/
│   ├── CMakeLists.txt
│   ├── OtaUpdateServiceImpl.hpp     ← Stub override — update logic
│   ├── OtaUpdateServiceImpl.cpp
│   ├── main_rpi.cpp                 ← CommonAPI service registration
│   └── vsomeip-ota.json             ← RPi vsomeip runtime config
└── laptop/
    ├── CMakeLists.txt
    ├── main_laptop.cpp              ← CommonAPI proxy — sends update
    └── vsomeip-ota-laptop.json      ← Laptop vsomeip runtime config

meta-myapp/
├── conf/
│   └── distro/aboelyazeed-distro.conf
├── wic/
│   └── rpi-ab.wks                   ← A/B partition layout
└── recipes-ota-update/
    ├── ota-update-agent/
    │   ├── ota-update-agent.bb
    │   ├── files/ota_update/        ← full source tree (synced from above)
    │   └── files/ota-update-agent.service
    └── update-status-init/
        ├── update-status-init.bb
        └── files/
            ├── update-status.json   ← seed state file
            └── update-status-init.service
```

------

## Known Limitations in Version 1

| Limitation                                                   | Resolution in Later Phases                  |
| ------------------------------------------------------------ | ------------------------------------------- |
| No authentication — any client can send an update            | Phase 3: Ed25519 signature verification     |
| No transport encryption                                      | Phase 3: TLS 1.3 on the laptop link         |
| Laptop connects directly to RPi — no gateway                 | Phase 2: QNX RPi5 acts as OTA gateway       |
| CommonAPI `FinalizeUpdate` call times out while RPi hashes 3.5GB | Fixed: `CallInfo(180000)` 3-minute timeout  |
| No watchdog — bad rootfs causes permanent boot failure       | Planned: U-Boot BCR script for A/B fallback |

------

## Authors

Ahmed Aboelyazeed — ITI Embedded Linux Track