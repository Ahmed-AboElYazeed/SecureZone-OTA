## To Clone the project

```bash
git clone --recurse-submodules https://github.com/Ahmed-AboElYazeed/SecureZone-OTA.git
```

If you already cloned:

```bash
git submodule update --init --recursive
```

# OTA Update System — Final Version

## Laptop → QNX Gateway (RPi5) → Yocto ECU (RPi3B+) with U-Boot A/B Boot

------

## Overview

This document describes the **final production-grade OTA update pipeline** connecting three systems:

1. **Laptop (Ubuntu)** — OTA management application (Qt/QML + Python sender)
2. **QNX RPi5 (Gateway)** — receives, verifies, and forwards the update
3. **Yocto RPi3B+ (ECU)** — the update target, using U-Boot for safe A/B boot

The system implements a full **A/B rootfs partition scheme** with U-Boot environment variable-based slot switching, SHA256 integrity verification at two independent points (QNX and Yocto), and a boot-confirm watchdog that automatically rolls back to the previous slot if the new rootfs is unhealthy or OTA-incompatible.

> **Key design principle:** The QNX gateway acts as the security boundary. It receives the full image, verifies integrity, and only then connects to the Yocto ECU via SOME/IP. The Yocto ECU never communicates directly with the laptop.

------

## System Architecture

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                         Final OTA System Architecture                             │
│                                                                                   │
│  ┌─────────────────┐     WiFi TCP      ┌──────────────────────┐                  │
│  │     Laptop      │  ──────────────►  │    QNX RPi5          │                  │
│  │  (Ubuntu)       │    port 55000     │    (OTA Gateway)     │                  │
│  │                 │                   │                      │                  │
│  │  Qt/QML App     │  ◄──────────────  │  TCP port 55001      │                  │
│  │  (OTA Manager)  │   Status/Logs     │  (Status Reporter)   │                  │
│  │                 │                   │                      │                  │
│  │  Python Sender  │                   │  1. Receive image    │                  │
│  │  send_update.py │                   │  2. SHA256 verify    │                  │
│  └─────────────────┘                   │  3. Forward via      │                  │
│                                        │     SOME/IP          │                  │
│                                        └──────────┬───────────┘                  │
│                                                   │ Dedicated Ethernet           │
│                                                   │ 192.168.50.0/24              │
│                                                   │ SOME/IP / CommonAPI          │
│                                                   ▼                              │
│                                        ┌──────────────────────┐                  │
│                                        │   Yocto RPi3B+       │                  │
│                                        │   (OTA ECU Target)   │                  │
│                                        │                      │                  │
│                                        │  OtaUpdateService    │                  │
│                                        │  (SOME/IP daemon)    │                  │
│                                        │                      │                  │
│                                        │  A/B partition       │                  │
│                                        │  U-Boot slot switch  │                  │
│                                        │  boot-confirm WD     │                  │
│                                        └──────────────────────┘                  │
└───────────────────────────────────────────────────────────────────────────────────┘
```

------

## Hardware

| Device     | Board            | OS                               | Role                                               |
| ---------- | ---------------- | -------------------------------- | -------------------------------------------------- |
| Laptop     | x86_64 PC        | Ubuntu 22.04 LTS                 | OTA source + management UI                         |
| Gateway    | Raspberry Pi 5   | QNX 8.0                          | Security gateway, image staging, SOME/IP forwarder |
| ECU Target | Raspberry Pi 3B+ | Yocto Linux (Scarthgap, aarch64) | Update target                                      |

### Network Interfaces

| Link         | Interface                   | IP                                 | Protocol             |
| ------------ | --------------------------- | ---------------------------------- | -------------------- |
| Laptop ↔ QNX | WiFi (`bcm0`)               | Dynamic (mDNS: `qnxpi.local`)      | TCP port 55000/55001 |
| QNX ↔ Yocto  | Ethernet (`cgem0` / `eth0`) | `192.168.50.100` / `192.168.50.50` | SOME/IP over TCP/UDP |

------

## SD Card Partition Layout (Yocto RPi3B+)

```
┌──────────┬────────────┬────────────┬──────────────┐
│  boot    │  rootfs-a  │  rootfs-b  │    mydata    │
│ (FAT32)  │   (ext4)   │   (ext4)   │    (ext4)    │
│  256MB   │   700MB    │   700MB    │   512MB      │
└──────────┴────────────┴────────────┴──────────────┘
  p1            p2            p3           p4
  /boot          /          (inactive)   /mydata
```

| Partition | Label    | Mount             | Purpose                                    |
| --------- | -------- | ----------------- | ------------------------------------------ |
| p1        | boot     | `/boot`           | U-Boot, kernel Image, DTB, `boot.scr`      |
| p2        | rootfs-a | `/` (when slot=a) | Root filesystem slot A                     |
| p3        | rootfs-b | `/` (when slot=b) | Root filesystem slot B                     |
| p4        | mydata   | `/mydata`         | Persistent: state file, logs, pending flag |

Both `rootfs-a` and `rootfs-b` are pre-populated from `--source rootfs` in the WKS file at flash time. The inactive slot is overwritten during OTA.

### WKS File (`meta-myapp/wic/rpi-ab.wks`)

```bash
part /boot   --source bootimg-partition --ondisk mmcblk0 --fstype=vfat \
             --label boot --active --align 4096 --fixed-size 256M
part /       --source rootfs --ondisk mmcblk0 --fstype=ext4 \
             --label rootfs-a --align 4096 --fixed-size 700M
part         --source rootfs --ondisk mmcblk0 --fstype=ext4 \
             --label rootfs-b --align 4096 --fixed-size 700M
part /mydata --ondisk mmcblk0 --fstype=ext4 \
             --label mydata --align 4096 --fixed-size 512M
```

------

## U-Boot Boot Slot Mechanism

**This is the key architectural change from Phase 1.** Slot selection is no longer done by editing `cmdline.txt`. Instead, U-Boot reads persistent environment variables from `/boot/uboot.env` to determine which partition to boot.

### U-Boot Environment Variables

| Variable      | Values     | Description                               |
| ------------- | ---------- | ----------------------------------------- |
| `active_slot` | `a` or `b` | Which rootfs slot to boot                 |
| `rootpart`    | `2` or `3` | Derived from `active_slot` by boot script |

### Boot Script (`meta-myapp/recipes-bsp/rpi-u-boot-scr/files/boot.cmd.in`)

```bash
# Read active slot from U-Boot environment
if env exists active_slot && test "${active_slot}" = "b"; then
    setenv rootpart 3
else
    setenv rootpart 2
fi

setenv bootargs "coherent_pool=1M 8250.nr_uarts=0 snd_bcm2835.enable_compat_alsa=0 \
    snd_bcm2835.enable_hdmi=1 bcm2708_fb.fbwidth=0 bcm2708_fb.fbheight=0 \
    vc_mem.mem_base=0x3ec00000 vc_mem.mem_size=0x40000000 dwc_otg.lpm_enable=0 \
    console=serial0,115200 root=/dev/mmcblk0p${rootpart} rootfstype=ext4 \
    rootwait quiet"

load mmc 0:1 ${kernel_addr_r} @@KERNEL_IMAGETYPE@@
load mmc 0:1 ${fdt_addr_r}    bcm2710-rpi-3-b-plus.dtb
@@KERNEL_BOOTCMD@@ ${kernel_addr_r} - ${fdt_addr_r}
```

### U-Boot Environment Config (`meta-myapp/recipes-bsp/u-boot/files/fw_env.config`)

```
# Device           Offset  Size
/boot/uboot.env    0       0x4000
```

This tells `fw_printenv` / `fw_setenv` where to read and write U-Boot environment variables within the boot partition.

### Slot Switching from C++ (OTA Agent)

```cpp
// Read current active slot
FILE *fp = popen("fw_printenv -n active_slot", "r");
char buf[16] = {};
fgets(buf, sizeof(buf), fp);
pclose(fp);
std::string currentSlot(buf);

// Switch to the other slot
std::string newSlot = (currentSlot.find('b') != std::string::npos) ? "a" : "b";
std::string cmd = "fw_setenv active_slot " + newSlot;
system(cmd.c_str());
```

> **Important:** `/boot` must be mounted before `fw_printenv` or `fw_setenv` can access `uboot.env`. The OTA agent systemd unit has `Requires=boot.mount` and `After=boot.mount` for this reason.

------

## Complete OTA Update Protocol

### Phase 1 — Laptop → QNX (TCP, port 55000)

The laptop Python sender uses a simple custom framing protocol:

```
┌──────────────────────────────────────┐
│  4 bytes  │  N bytes   │  M bytes    │
│  hdr_len  │  JSON hdr  │  raw image  │
│ (big-end) │            │   stream    │
└──────────────────────────────────────┘
```

**JSON Header fields:**

```json
{
    "version"   : "2.0.0",
    "imageSize" : 595591168,
    "sha256"    : "950394fa...",
    "signature" : ""
}
```

### Phase 2 — QNX Verification

After receiving the full image to `/var/ota_staging/update.img`:

```
1. Compute SHA256 of staged file
2. Compare with announced sha256
3. PASS → proceed to forward
   FAIL → delete staged file, log error, do NOT connect to Yocto
```

### Phase 3 — QNX → Yocto (SOME/IP / CommonAPI)

QNX acts as a CommonAPI client, replaying the OTA protocol to Yocto:

```
QNX (Client)                           Yocto (OtaUpdateServiceImpl)
────────────                           ────────────────────────────

AnnounceUpdate(version,          ──►   resolveInactiveSlot()
  imageSize, sha256)                   → fw_printenv active_slot
                                       → if a: write to p3
                                       → if b: write to p2
                                       openPartition(O_WRONLY|O_SYNC)
                                       updateStatusFile("in_progress")
                                 ◄──   reply(true, "Ready")

loop:
  SendChunk(offset, data)        ──►   lseek64 + write to block device
                                 ◄──   reply(nextOffset)

FinalizeUpdate(sha256)           ──►   fsync + close partition
  [CallInfo timeout: 180s]             computeSHA256() ← reads full partition
                                       compare hashes
                                       FAIL → reply(false), no slot switch
                                       PASS → fw_setenv active_slot <new>
                                              updateStatusFile("complete")
                                              write /mydata/boot_pending
                                 ◄──   reply(true, "rebooting in 5s")
                                       sleep(5s) → reboot

                                 [Yocto reboots into new slot via U-Boot]
```

**SOME/IP Async Events** (Yocto → QNX, fire-and-forget):

| Trigger                 | `status`      | `percent` |
| ----------------------- | ------------- | --------- |
| AnnounceUpdate accepted | `in_progress` | 0         |
| Every 1% chunk written  | `in_progress` | 1–99      |
| FinalizeUpdate complete | `complete`    | 100       |
| Any failure             | `failed`      | 0         |

------

## Boot-Confirm Watchdog

Located at `/usr/bin/boot-confirm.sh`, runs as a systemd oneshot at every boot.

### Logic

```
Boot
  │
  ▼
/mydata/boot_pending exists?
  │ NO  → normal boot, exit 0
  │ YES → this is a post-OTA boot, run health checks:
  │
  ├─ Check: /usr/bin/ota_update_service binary exists
  ├─ Check: /etc/vsomeip/vsomeip-ota.json exists
  ├─ Check: /usr/lib/systemd/system/ota-update-agent.service exists
  ├─ Check: eth0 interface exists (ip link show eth0)
  └─ Check: fw_printenv is readable
       │
       ├─ ALL PASS →
       │     touch /mydata/boot_confirmed
       │     rm /mydata/boot_pending
       │     log "Boot confirmed"
       │     exit 0
       │
       └─ ANY FAIL →
             fw_setenv active_slot <previous slot>
             rm /mydata/boot_pending
             updateStatusFile("fallback", reason)
             reboot → boots back into old trusted slot
```

### Systemd Unit Dependencies

```
boot-confirm.service
  Requires=boot.mount mydata.mount
  After=boot.mount mydata.mount
  Before=ota-update-agent.service
  Type=oneshot
  RemainAfterExit=yes
```

This ensures:

- `/boot` is mounted before `fw_printenv` is called
- `/mydata` is mounted before reading/writing the pending flag
- The watchdog always runs before the OTA agent starts

------

## QNX Gateway Application

Built using the **CTI (Custom Target Image)** build system for QNX 8.0.

### Source Files

| File                     | Purpose                                                      |
| ------------------------ | ------------------------------------------------------------ |
| `TcpReceiver.cpp/hpp`    | TCP server on port 55000, receives JSON header + image stream |
| `ImageVerifier.cpp/hpp`  | Computes and verifies SHA256 of staged file using OpenSSL    |
| `YoctoForwarder.cpp/hpp` | CommonAPI proxy — replays OTA protocol to Yocto over SOME/IP |
| `StatusReporter.cpp/hpp` | TCP server on port 55001, pushes JSON status lines to Qt app |
| `main.cpp`               | Orchestrates the pipeline, queries Yocto version on startup  |

### Status JSON Messages (port 55001)

```json
{ "type": "state",           "state": "idle",             "activeSlot": "a", "versionA": "1.0.0", "versionB": "" }
{ "type": "state",           "state": "receiving_from_laptop" }
{ "type": "state",           "state": "verifying" }
{ "type": "state",           "state": "forwarding_to_ecu" }
{ "type": "state",           "state": "complete" }
{ "type": "state",           "state": "failed" }
{ "type": "laptop_progress", "received": 104857600, "total": 595591168, "percent": 17 }
{ "type": "yocto_progress",  "percent": 45, "msg": "Received 268MB / 595MB" }
{ "type": "version",         "activeSlot": "b", "versionA": "1.0.0", "versionB": "2.0.0" }
{ "type": "log",             "msg": "[GW] Image verified OK" }
```

### CTI Makefile Fragment (`src/ota-gateway.mk`)

```makefile
OTA_GW_DIR  = $(CURDIR)/ota-gateway
OTA_GW_SRCS = $(OTA_GW_DIR)/main.cpp          \
              $(OTA_GW_DIR)/TcpReceiver.cpp    \
              $(OTA_GW_DIR)/ImageVerifier.cpp  \
              $(OTA_GW_DIR)/YoctoForwarder.cpp \
              $(OTA_GW_DIR)/StatusReporter.cpp \
              $(OTA_GW_DIR)/src-gen/...

source/ota-gateway-built-$(QNX_ARCH): source/ota-gateway-ready $(OTA_GW_SRCS)
    # stub Linux-only eventfd symbol
    echo 'int eventfd(unsigned int i, int f){ return -1; }' > stub_eventfd.c
    qcc -Vgcc_nto$(CC_TARGET) -D_QNX_SOURCE -c stub_eventfd.c -o stub_eventfd.o

    q++ -Vgcc_nto$(CC_TARGET)_cxx -D_QNX_SOURCE -DSA_RESTART=0 -std=c++17 \
        -D__LITTLE_ENDIAN=1234 -D__BIG_ENDIAN=4321 -D__BYTE_ORDER=__LITTLE_ENDIAN \
        $(OTA_GW_SRCS) stub_eventfd.o \
        -L$(CURDIR)/stage/usr/lib \
        -lssl -lcrypto -lCommonAPI -lCommonAPI-SomeIP \
        -lvsomeip3 -lvsomeip3-sd -lvsomeip3-cfg -lvsomeip3-e2e \
        -lboost_system -lboost_thread -lboost_filesystem \
        -lboost_log -lboost_log_setup -lboost_atomic -lboost_chrono -lboost_regex \
        -lsocket \
        -o $(CURDIR)/stage/usr/bin/ota-gateway
    touch $@
```

### CTI Snippets

| Snippet file                      | Purpose                                                      |
| --------------------------------- | ------------------------------------------------------------ |
| `system_files.custom.ota-gateway` | Binary, vsomeip config, staging dir, all `.so` libs          |
| `post_start.custom.ota-gateway`   | `mkdir -p /var/ota_staging`, set multicast route, launch daemon |

### QNX vsomeip Config (`/system/etc/vsomeip/vsomeip-ota-qnx.json`)

```json
{
    "unicast"  : "192.168.50.100",
    "netmask"  : "255.255.255.0",
    "network"  : "cgem0",
    "applications" : [{ "name" : "OtaGateway", "id" : "0x1300" }],
    "services" : [{
        "service"  : "0x1101",
        "instance" : "0x0001",
        "methods"  : [
            { "id" : "0x0001" },
            { "id" : "0x0002" },
            { "id" : "0x0003", "timeout" : "180000" }
        ]
    }],
    "routing"  : "OtaGateway",
    "service-discovery" : {
        "enable" : "true", "multicast" : "224.0.0.1",
        "port" : "30490", "protocol" : "udp"
    }
}
```

> Method `0x0003` (`FinalizeUpdate`) has a **180-second timeout** because SHA256 verification of ~600MB on RPi3 takes 45–60 seconds. Without this, CommonAPI returns `TIMEOUT` before Yocto finishes.

------

## Yocto Target Application

### Libraries and Packages

| Package                  | Version | Purpose                              |
| ------------------------ | ------- | ------------------------------------ |
| `vsomeip3`               | 3.5.5   | SOME/IP transport                    |
| `capicxx-core-runtime`   | 3.2.4   | CommonAPI core                       |
| `capicxx-someip-runtime` | 3.2.4   | CommonAPI SOME/IP binding            |
| `boost`                  | 1.82.0  | Threading, logging                   |
| `openssl`                | 3.x     | SHA256 via EVP API                   |
| `u-boot-env`             | —       | Provides `fw_printenv` / `fw_setenv` |

### OTA Agent Key Logic Summary

```
AnnounceUpdate:
  fw_printenv -n active_slot → determine inactive slot
  open(/dev/mmcblk0pX, O_WRONLY|O_SYNC)
  updateStatusFile("in_progress")
  reply(true, "Ready")

SendChunk:
  lseek64(fd, offset, SEEK_SET)
  write(fd, data, size)
  reply(nextOffset)

FinalizeUpdate:
  fsync(fd) + sync()
  close(fd)
  computeSHA256 via EVP_sha256 → read exactly imageSize bytes
  compare with announced hash
    MISMATCH → reply(false), NO slot switch, NO reboot
    MATCH    →
      fw_setenv active_slot <new>
      updateStatusFile("complete", version)
      write /mydata/boot_pending
      reply(true, "rebooting in 5s")
      sleep(5s) → system("reboot")
```

### Yocto vsomeip Config (`/etc/vsomeip/vsomeip-ota.json`)

```json
{
    "unicast" : "192.168.50.50",
    "netmask" : "255.255.255.0",
    "network" : "eth0",
    "applications" : [{ "name" : "OtaUpdateService", "id" : "0x1100" }],
    "services" : [{
        "service"    : "0x1101",
        "instance"   : "0x0001",
        "reliable"   : { "port" : "30510", "enable-magic-cookies" : "false" },
        "unreliable" : "30511",
        "methods"    : [
            { "id" : "0x0001" },
            { "id" : "0x0002" },
            { "id" : "0x0003" }
        ],
        "events" : [{
            "service" : "0x1101", "instance" : "0x0001",
            "event" : "0x8001", "is_field" : "false",
            "reliability" : "reliable"
        }],
        "eventgroups" : [{
            "service" : "0x1101", "instance" : "0x0001",
            "eventgroup" : "0x0001",
            "multicast" : { "address" : "224.0.0.1", "port" : "30490" },
            "threshold" : "0"
        }]
    }],
    "routing" : "OtaUpdateService",
    "service-discovery" : {
        "enable" : "true", "multicast" : "224.0.0.1",
        "port" : "30490", "protocol" : "udp",
        "cyclic_offer_delay" : "2000"
    }
}
```

### Yocto Recipes Structure

```
meta-myapp/
├── wic/
│   └── rpi-ab.wks                              ← A/B partition layout
├── recipes-bsp/
│   ├── rpi-u-boot-scr/
│   │   └── files/boot.cmd.in                   ← U-Boot script (active_slot)
│   └── u-boot/
│       └── files/fw_env.config                 ← points to /boot/uboot.env
├── recipes-core/
│   └── base-files/
│       └── base-files_%.bbappend               ← adds /boot to fstab
├── recipes-ota/
│   ├── boot-confirm/
│   │   ├── boot-confirm.bb
│   │   ├── files/boot-confirm.sh               ← boot watchdog
│   │   └── files/boot-confirm.service
│   ├── ota-update-agent/
│   │   ├── ota-update-agent.bb
│   │   ├── files/ota_update/ (source tree)
│   │   └── files/ota-update-agent.service
│   └── update-status-init/
│       ├── update-status-init.bb
│       └── files/update-status.json            ← seed state file
└── recipes-core/
    └── images/
        └── core-aboelyazeed-image.bb
```

------

## Qt/QML Management Application (Laptop)

A metallic-themed Qt6 QML desktop application that connects to the QNX gateway status port (55001) and provides real-time visibility and control over the update process.

### Features

| Feature                   | Implementation                                               |
| ------------------------- | ------------------------------------------------------------ |
| Gateway connection toggle | `GatewayClient` connects/disconnects to port 55001           |
| ECU state display         | State machine: idle → receiving → verifying → forwarding → complete/failed |
| Active slot + versions    | Populated from QNX status JSON on connect and after each update |
| Dual progress bars        | Laptop→QNX (blue) and QNX→ECU (green) independently tracked  |
| Log console               | Colour-coded, monospace, auto-scroll, filterable by source tag |
| Image file browser        | `QFileDialog` for `.ext4` / `.img` selection                 |
| Send/Cancel button        | Launches `send_update.py` as a `QProcess`, cancel kills it   |

### Backend Classes

| Class           | Purpose                                                      |
| --------------- | ------------------------------------------------------------ |
| `GatewayClient` | QObject wrapping `QTcpSocket` to port 55001, parses JSON lines, exposes Q_PROPERTYs |
| `UpdateSender`  | QObject wrapping `QProcess` running `send_update.py`         |

### Build

```bash
cd ~/ITI_Files/linux/ota-manager-qt
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
./OtaManager
```

------

## SOME/IP Service Definition

Defined in Franca IDL, code-generated with CommonAPI generators.

### Service Interface (`fidl/OtaUpdate.fidl`)

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

### Deployment IDs (`fidl/OtaUpdate.fdepl`)

| Element                 | ID                |
| ----------------------- | ----------------- |
| Service ID              | `0x1101`          |
| Instance ID             | `0x0001`          |
| AnnounceUpdate          | `0x0001`          |
| SendChunk               | `0x0002`          |
| FinalizeUpdate          | `0x0003`          |
| UpdateStatus event      | `0x8001`          |
| UpdateStatus eventgroup | `0x0001`          |
| TCP port (Yocto)        | `30510`           |
| UDP port (unreliable)   | `30511`           |
| SD multicast            | `224.0.0.1:30490` |

------

## Update State File (`/mydata/update-status.json`)

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

| Field         | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| `active_slot` | `a` or `b` — updated by OTA agent after successful slot switch |
| `version_a/b` | Version string written during `FinalizeUpdate`               |
| `status`      | `idle`, `in_progress`, `complete`, `failed`, `fallback`      |

> The `// files/update-status.json` comment line that appeared in older versions must be removed — it breaks any JSON parser that reads this file.

------

## Persistent Flags on `/mydata`

| File                 | Written by                   | Read by                        | Purpose                              |
| -------------------- | ---------------------------- | ------------------------------ | ------------------------------------ |
| `boot_pending`       | OTA agent after slot switch  | `boot-confirm.sh` at next boot | Triggers health check on new rootfs  |
| `boot_confirmed`     | `boot-confirm.sh` on success | Monitoring / debugging         | Records that this boot was validated |
| `update-status.json` | OTA agent                    | QNX gateway (via SSH), Qt app  | Current version and state            |
| `vsomeip-ota.log`    | vsomeip runtime              | Developer debugging            | SOME/IP transport log                |
| `boot-confirm.log`   | `boot-confirm.sh`            | Developer debugging            | Watchdog decision log                |

------

## Running a Full Update

### Prerequisites

```bash
# Laptop — add multicast route if needed (for direct testing)
sudo ip route add 224.0.0.0/4 dev enp3s0 2>/dev/null || true

# QNX — verify gateway is running
ssh root@qnxpi.local "ps -A | grep ota-gateway"

# Yocto — verify OTA agent is running and on eth0
ssh root@192.168.50.50 "systemctl status ota-update-agent"
ssh root@192.168.50.50 "ip addr show eth0"

# QNX — verify multicast route on cgem0
ssh root@qnxpi.local "netstat -rn | grep 224"
```

### Option A — Qt App (recommended)

1. Launch `./OtaManager`
2. Enter `qnxpi.local` in the gateway field, click **Connect**
3. Verify ECU status panel shows current slot and version
4. Click **Browse**, select the `.ext4` rootfs image
5. Click **Send Update**
6. Watch both progress bars and the log console
7. After reboot, Qt app auto-refreshes slot and version info

### Option B — Command line

```bash
# Terminal 1 — watch Yocto
ssh root@192.168.50.50 "journalctl -u ota-update-agent -f"

# Terminal 2 — watch QNX status stream
nc qnxpi.local 55001

# Terminal 3 — send update
python3 ~/ITI_Files/linux/ota_update/laptop-app/python/send_update.py \
    /path/to/rootfs.ext4 \
    qnxpi.local
```

------

## Deploying Changes Without Full Reflash

### Update Yocto OTA Agent

```bash
# Sync source
rsync -av --delete \
  ~/ITI_Files/linux/ota_update/ \
  ~/yocto/meta-myapp/recipes-ota/ota-update-agent/files/ota_update/ \
  --exclude build-laptop/ --exclude ".git/"

# Build
cd ~/yocto/build-rpi
bitbake -c cleansstate ota-update-agent && bitbake ota-update-agent

# Deploy
WORK=~/yocto/shared_build/tmp/work/cortexa53-poky-linux/ota-update-agent/1.0/image
scp $WORK/usr/bin/ota_update_service root@192.168.50.50:/usr/bin/
ssh root@192.168.50.50 "systemctl restart ota-update-agent"
```

### Update QNX Gateway

```bash
# Build
cd ~/ITI_Files/QNX/QNX_Snippets_repo
make TARGET=rpi5

# Deploy binary only
scp src/stage/usr/bin/ota-gateway root@qnxpi.local:/system/usr/bin/ota-gateway

# Restart
ssh root@qnxpi.local "slay ota-gateway; sleep 1; \
    VSOMEIP_CONFIGURATION=/system/etc/vsomeip/vsomeip-ota-qnx.json \
    VSOMEIP_APPLICATION_NAME=OtaGateway \
    /system/usr/bin/ota-gateway &"
```

### Update vsomeip Config Only (no rebuild)

```bash
# Yocto
scp ~/ITI_Files/linux/ota_update/rpi/vsomeip-ota.json \
    root@192.168.50.50:/etc/vsomeip/vsomeip-ota.json
ssh root@192.168.50.50 "systemctl restart ota-update-agent"

# QNX
scp ~/ITI_Files/QNX/QNX_Snippets_repo/src/ota-gateway/vsomeip-ota-qnx.json \
    root@qnxpi.local:/system/etc/vsomeip/vsomeip-ota-qnx.json
ssh root@qnxpi.local "slay ota-gateway && sleep 1 && /system/usr/bin/ota-gateway &"
```

------

## Monitoring and Debugging

### Verify slot after reboot

```bash
# Check which partition is mounted as root
ssh root@192.168.50.50 "lsblk && cat /mydata/update-status.json"

# Read U-Boot environment directly
ssh root@192.168.50.50 "fw_printenv active_slot"
```

### Expected Yocto journal for a successful update

```
[OTA] Currently booted from slot A → will write to slot B
[OTA] AnnounceUpdate: version=2.0.0 size=595591168 hash=950394fa...
[OTA] SendChunk ... (repeated)
[OTA] FinalizeUpdate: starting...
[OTA] Partition flushed and closed
[OTA] Computing SHA256 of written partition (568 MB)...
[OTA] Expected : 950394fa...
[OTA] Computed : 950394fa...
[OTA] SHA256 OK
[OTA] Switching boot slot...
[OTA] Boot slot switched successfully (fw_setenv active_slot b)
[OTA] Pending boot flag written
[OTA] Reply sent to QNX
[OTA] Rebooting now.
```

### Expected boot-confirm log (`/mydata/boot-confirm.log`)

```
[BOOT-CONFIRM] Boot confirmation check starting...
[BOOT-CONFIRM] Pending update detected — running health checks...
[BOOT-CONFIRM] OTA compatibility check passed
[BOOT-CONFIRM] Network interface check passed
[BOOT-CONFIRM] fw_printenv check passed
[BOOT-CONFIRM] Boot confirmed successfully
```

### QNX status stream test

```bash
# Connect from laptop and watch raw JSON
nc qnxpi.local 55001
# Output (one JSON line per event):
# {"type":"state","state":"idle","activeSlot":"b","versionA":"1.0.0","versionB":"2.0.0"}
# {"type":"state","state":"receiving_from_laptop"}
# {"type":"laptop_progress","received":52428800,"total":595591168,"percent":8}
# ...
```

------

## Complete Project Source Tree

```
ota_update/                              ← Laptop + Yocto SOME/IP source
├── CMakeLists.txt
├── fidl/
│   ├── OtaUpdate.fidl
│   └── OtaUpdate.fdepl
├── src-gen/v1/com/myapp/ota/            ← CommonAPI generated (never edit)
├── rpi/
│   ├── CMakeLists.txt
│   ├── OtaUpdateServiceImpl.hpp/cpp     ← U-Boot slot switching, SHA256, write
│   ├── main_rpi.cpp
│   └── vsomeip-ota.json
└── laptop/
    ├── CMakeLists.txt
    ├── main_laptop.cpp                  ← Phase 1 direct client (reference only)
    └── vsomeip-ota-laptop.json

laptop-app/
└── python/
    └── send_update.py                   ← TCP framing sender used by Qt app

ota-manager-qt/                          ← Qt/QML management application
├── CMakeLists.txt
├── main.cpp
├── backend/
│   ├── GatewayClient.hpp/cpp
│   └── UpdateSender.hpp/cpp
└── qml/
    ├── main.qml
    ├── components/
    │   ├── MetalButton.qml
    │   ├── MetalPanel.qml
    │   ├── StatusIndicator.qml
    │   ├── DualProgressPanel.qml
    │   └── LogConsole.qml
    └── theme/
        └── Theme.qml

meta-myapp/                              ← Yocto custom layer
├── conf/distro/aboelyazeed-distro.conf
├── wic/rpi-ab.wks
├── recipes-bsp/
│   ├── rpi-u-boot-scr/files/boot.cmd.in
│   └── u-boot/files/fw_env.config
├── recipes-core/
│   ├── base-files/base-files_%.bbappend
│   └── images/core-aboelyazeed-image.bb
├── recipes-ota/
│   ├── boot-confirm/
│   │   ├── boot-confirm.bb
│   │   └── files/ (boot-confirm.sh, .service)
│   ├── ota-update-agent/
│   │   ├── ota-update-agent.bb
│   │   └── files/ota_update/ (full source tree)
│   └── update-status-init/
│       ├── update-status-init.bb
│       └── files/ (update-status.json, .service)
├── recipes-someip/vsomeip3/
├── recipes-capicxx-core-runtime/
├── recipes-capicxx-someip-runtime/
└── recipes-support/boost/

QNX_Snippets_repo/                       ← QNX CTI build system
├── src/
│   ├── Makefile.mk
│   ├── ota-gateway.mk
│   ├── vsomeip.mk
│   ├── commonapi.mk
│   ├── commonapi_someip.mk
│   └── ota-gateway/
│       ├── TcpReceiver.hpp/cpp
│       ├── ImageVerifier.hpp/cpp
│       ├── YoctoForwarder.hpp/cpp
│       ├── StatusReporter.hpp/cpp
│       ├── main.cpp
│       ├── vsomeip-ota-qnx.json
│       └── src-gen/v1/com/myapp/ota/   ← same generated files as ota_update
├── snippets/
│   ├── system_files.custom.ota-gateway
│   └── post_start.custom.ota-gateway
└── stage/usr/                           ← built libraries (boost, vsomeip, CommonAPI)
```

------

## Known Issues and Mitigations

| Issue                                                        | Root Cause                                                   | Mitigation                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| QNX logs `FinalizeUpdate failed` even when update succeeds   | SHA256 of 600MB takes 45–60s; default CommonAPI timeout is shorter | `CallInfo(180000)` on `FinalizeUpdate` call + `"timeout":"180000"` in vsomeip config |
| `multicast SD message` warnings on Yocto after laptop disconnects | vsomeip continues SD after client leaves                     | Cosmetic only — does not affect function                     |
| `boot_pending` flag not cleared if Yocto crashes mid-update  | OTA agent killed before writing the flag                     | Flag is only written after `FinalizeUpdate` completes, so partial writes leave no flag |
| `update-status.json` has `//` comment line                   | Legacy seed file format                                      | `updateStatusFile()` strips comment lines before parsing     |
| QNX WiFi IP is dynamic                                       | DHCP assignment changes                                      | Resolved via mDNS hostname `qnxpi.local` — no static IP needed |

------

## Authors

Ahmed Aboelyazeed — ITI Embedded Linux Track