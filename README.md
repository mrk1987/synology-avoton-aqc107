# synology-avoton-aqc107

AQC107 (atlantic) kernel module compiled for Synology Avoton NAS running DSM 7.x (kernel 3.10.108).

## The problem

Cheap 10GbE cards based on the Marvell/Aquantia AQC107 chip (ASUS XG-C100C, TP-Link TX401, and others) don't work out of the box on Synology Avoton NAS units running DSM 7.x. Two issues:

1. DSM's bundled `atlantic.ko` only recognises Synology's own sub-vendor IDs — generic cards are ignored even if the card appears in `lspci`
2. Existing precompiled drivers (syno-stuff/aqc107, hyllm/dsm72drivers) don't cover this combination — the syno-stuff driver targets DSM 6.2 (kernel 3.10.105) and the hyllm repo has no Avoton build

Avoton NAS units (DS1517+, DS1817+, DS2415+, DS2419+, etc.) run kernel **3.10.108** even on DSM 7.x, which means you need a driver compiled against that exact kernel.

## Compatible hardware

Tested on:
- **NAS:** Synology DS1817+ (Avoton, Intel Atom C2538), DSM 7.3.2-86009
- **NIC:** ASUS XG-C100C V1/V2 (AQC107, PCI ID `1d6a:94c0`)

Should work on any Avoton-based Synology NAS on DSM 7.x, and likely with other AQC107 cards (TP-Link TX401, generic cards).

## Download

Grab `atlantic.ko` from this repo and follow the install instructions below.

> **Disclaimer:** Loading unsigned kernel modules taints the kernel. This works on my setup but I take no responsibility for data loss or system damage. Back up your NAS before proceeding.

---

## Install

### 1. Copy the driver to your NAS

Upload `atlantic.ko` to your NAS via DSM File Station, then SSH in as root:

```bash
mkdir -p /volume1/aqcnetconf
cp /volume1/YourShare/atlantic.ko /volume1/aqcnetconf/atlantic.ko
```

### 2. Test it loads

```bash
rmmod atlantic_v2 atlantic 2>/dev/null
insmod /volume1/aqcnetconf/atlantic.ko
dmesg | tail -15
```

You should see MSI/MSI-X IRQ lines and `Detect ATL2FW`. If you see `version magic` errors, the driver doesn't match your kernel — see the build section below.

### 3. Create the boot script

Create `/usr/local/etc/rc.d/aqc107.sh`:

```bash
cat > /usr/local/etc/rc.d/aqc107.sh << 'EOF'
#!/bin/sh
confdir=/volume1/aqcnetconf
case "$1" in
start)
    rmmod atlantic_v2 atlantic 2>/dev/null
    insmod $confdir/atlantic.ko
    /etc/rc.network restart
    ;;
esac
exit 0
EOF
chmod 755 /usr/local/etc/rc.d/aqc107.sh
```

### 4. Configure the interface

DSM will show the card as a new LAN interface once the driver loads. Go to **Control Panel → Network → Network Interface** and configure the new interface with a static IP. For a direct link to another machine, a separate subnet works well (e.g. `10.0.0.1/24`).

For jumbo frames, set MTU to **9000** in the interface settings.

### 5. Reboot and verify

After rebooting, the boot script fires a few seconds after DSM starts. Check the interface appeared:

```bash
ip link show | grep eth4
# Should show UP and your configured IP
```

---

## Build it yourself

If this driver doesn't match your kernel, here's how to compile it.

### Requirements

- Linux or WSL (Ubuntu works fine)
- `gcc`, `make`

### Step 1 — Get the Synology toolchain

Find your platform at `https://archive.synology.com/download/ToolChain/toolkit/` (check `https://kb.synology.com/en-global/DSM/tutorial/What_kind_of_CPU_does_my_NAS_have` for your platform name).

```bash
wget "https://global.synologydownload.com/download/ToolChain/toolkit/7.3/avoton/ds.avoton-7.3.dev.txz"
mkdir -p ~/syno-toolkit
tar -xf ds.avoton-7.3.dev.txz -C ~/syno-toolkit

KDIR=~/syno-toolkit/usr/local/x86_64-pc-linux-gnu/x86_64-pc-linux-gnu/sys-root/usr/lib/modules/DSM-7.3/build

# Verify kernel version
cat $KDIR/include/generated/utsrelease.h
```

### Step 2 — Get the driver source

Download the ASUS XG-C100C Linux driver from the [ASUS support page](https://www.asus.com/networking-iot-servers/wired-networking/all-series/xg-c100c/helpdesk_download/). Extract `atlantic.tar.gz` from the Linux driver package.

Or use the [Aquantia/AQtion](https://github.com/Aquantia/AQtion) open source driver.

### Step 3 — Fix the compatibility shim

Synology's 3.10.108 kernel already has `ether_addr_copy` but the driver's compatibility header tries to define it for kernels older than 3.14, causing a redefinition error. Fix it:

```bash
sed -i 's/LINUX_VERSION_CODE < KERNEL_VERSION(3, 14, 0)) && !(RHEL_RELEASE_CODE)/LINUX_VERSION_CODE < KERNEL_VERSION(3, 10, 0)) \&\& !(RHEL_RELEASE_CODE)/' aq_compat.h
```

### Step 4 — Build

```bash
make -C $KDIR M=$(pwd) modules
modinfo atlantic.ko | grep vermagic  # must match your NAS kernel
```

---

## References

- [Original r/synology post](https://www.reddit.com/r/synology/comments/k4a5px/how_i_got_a_generic_cheap_aqc107_card_working_on/) — the DSM 6.2 guide this is based on
- [syno-stuff/aqc107](https://github.com/syno-stuff/aqc107) — DSM 6.2 Avoton driver
- [hyllm/dsm72drivers](https://github.com/hyllm/dsm72drivers) — DSM 7.2 drivers for other platforms
