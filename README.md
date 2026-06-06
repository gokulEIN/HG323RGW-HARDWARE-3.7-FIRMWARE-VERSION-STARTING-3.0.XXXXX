# HG323 RGW 3.7 - Custom Optimized Firmware

## Hardware Specifications

**Device:** HG323RGW GPON ONT Router - Hardware Version 3.7
**SoC:** Realtek RTL9602C (RLX5281, MIPS 34Kc @ 500-700 MHz)
**RAM:** 128MB DDR2
**Flash:** SPI NOR Flash
**WAN:** GPON (ITU-T G.984/G.988 compliant)
**LAN:** 4x GbE ports
**WLAN:** 802.11n 2.4GHz
**Kernel:** Linux 2.6.30.9 (Realtek Luna SDK)
**Filesystem:** SquashFS with LZMA compression

## Custom Firmware Features

### Working Features

✅ **GPON Connectivity**
- Full ITU-T G.984/G.988 OMCI support
- Auto-registration with ISP OLT
- BSNL GPON tested and working

✅ **WiFi (802.11n)**
- Fixed to Channel 12 (no auto-scan on boot)
- WPA/WPA2 security
- Full 802.11n functionality
- Modified `/etc/rc.d/rc35` to disable channel scanning

✅ **Network Services**
- IPv4 routing and NAT
- IPv6 support (all required modules included)
- DHCP server/client
- DNS proxy
- PPPoE support

✅ **Management**
- Web interface (HTTP/HTTPS)
- Telnet access
- CLI tools and BusyBox utilities

✅ **OMCI Protocol**
- omci_app daemon
- libomci_mib.so core library
- All feature modules intact
- TR-142 diagnostic module (rtk_tr142.ko)

## Removed Components

### Binaries Removed from /bin/

**Printing Services (~500KB)**
- cupsd, lpadmin, lpstat, lp, startcupsd
- libcups.so, libcups.so.1

**File Transfer (~300KB)**
- ftp, ftpd, tftp, tftpd

**UPnP Services (~200KB)**
- miniupnpd, upnpctrl
- libmini_upnp.so

**Diagnostics & Testing (~2.8MB)**
- diag (2.8MB - not needed for normal operation)
- radvdump (22KB - IPv6 RA dump)
- 11N_UDPserver (15KB - WiFi test tool)
- udpechoserver (7.2KB - Echo test)

### Kernel Modules Removed from /lib/modules/

**EPON Drivers (~39KB)**
- epon_drv.ko (9.6KB)
- epon_mpcp.ko (22KB)
- epon_polling.ko (7.2KB)
- *Note: This device uses GPON, not EPON*

**SCSI Support (~3KB)**
- scsi_wait_scan.ko (2.5KB)
- *Note: No USB storage support needed*

## Known Issues

### ⚠️ WPS Button Interrupt Storm

**Symptoms:**
- IRQ 15 fires 800-1,000 times per second
- Load average shows ~1.99 (high)
- Shared interrupt: `gpio_wps_button_interrupt` + `wlan0`

**Root Cause:**
- WPS button GPIO interrupt handler compiled into kernel (uImage)
- Hardware resistors R19/C39 removed, causing GPIO floating state
- `/proc/wps_button` interface exists but doesn't control interrupt handler

**Impact:**
- Cosmetic issue only - all functions work normally
- System remains stable despite high load
- No performance degradation in real-world usage

## Installation Instructions

### Method 1: Web Interface Upload

1. Access router web interface (default: http://192.168.1.1)
2. Login with admin credentials
3. Navigate to firmware upgrade section
4. Upload `firmware_slim_v12.tar`
5. Wait 3-5 minutes for upgrade and reboot

### Method 2: U-Boot TFTP Recovery

If primary partition is bricked:

```bash
# Enter U-Boot console during boot (press ESC)
# Set server IP (your TFTP server)
setenv serverip 192.168.1.100

# Set device IP
setenv ipaddr 192.168.1.1

# Download and flash to partition 1
tftp 0x80000000 firmware_slim_v12.tar
flash_write 0x80000000 0x20000 <size_in_hex>

# Boot from partition 1
setenv boot_part 1
saveenv
reset
```

### Method 3: Copy Between Partitions

From working partition to bricked partition (U-Boot):

```bash
# Copy from partition 0 to partition 1
nand read 0x80000000 0x200000 0x700000
nand erase 0x900000 0x700000
nand write 0x80000000 0x900000 0x700000

# Or use MTD tools from running system:
dd if=/dev/mtd5 of=/dev/mtd6 bs=1M
```

## MTD Partition Layout

```
dev:    size   erasesize  name
mtd0: 00020000 00010000 "boot"
mtd1: 00010000 00010000 "env"
mtd2: 00010000 00010000 "env2"
mtd3: 00010000 00010000 "config"
mtd4: 00010000 00010000 "k0"
mtd5: 00200000 00010000 "r0"      # Rootfs Partition 0
mtd6: 00200000 00010000 "r1"      # Rootfs Partition 1
mtd7: 00010000 00010000 "linux"
mtd8: 00010000 00010000 "rootfs"
```

## Configuration Files Modified

### /etc/rc.d/rc35 - WiFi Configuration
**Modified:** Channel selection logic
**Change:** Disabled auto-scan, fixed to channel 12
**Original behavior:** Scanned for free channel on every boot
**New behavior:** Always uses channel 12

```bash
# Original line (around line 450):
# CH=$(get_free_channel)

# Modified to:
CH=12
```

## Technical Details

### Filesystem Structure

```
/
├── bin/          # Essential binaries and utilities
├── dev/          # Device nodes (41 devices from devices.pf)
├── etc/          # Configuration files
│   ├── rc.d/     # Startup scripts (rc0-rc99)
│   └── scripts/  # Helper scripts
├── lib/          # Shared libraries
│   ├── modules/  # Kernel modules (.ko files)
│   └── features/ # OMCI feature modules
├── proc/         # Kernel interface (mounted at runtime)
├── sys/          # Sysfs interface (mounted at runtime)
├── usr/          # User programs
└── var/          # Variable data (runtime)
```

### Boot Process

1. U-Boot loads kernel (uImage) from flash
2. Kernel mounts squashfs rootfs from MTD partition
3. `/etc/preinit` runs early initialization
4. `/etc/init.d/rcS` executes scripts in `/etc/rc.d/` (rc0→rc99)
5. Key services start:
   - rc15: Network initialization
   - rc25: GPON/OMCI stack
   - rc30: WiFi initialization
   - rc35: WiFi channel configuration
   - rc50: Web interface
   - rc60: DHCP server

### GPON/OMCI Stack

**Components:**
- `omci_app` - Main OMCI daemon
- `libomci_mib.so` - MIB database library
- `/lib/features/` - Feature modules:
  - libfh_omciapi.so - Fiber home API
  - libvs_omciapi.so - Vendor-specific API
  - libtr69_omciapi.so - TR-069 integration

**Kernel Modules:**
- `rtk_tr142.ko` - TR-142 diagnostics (35KB) - **REQUIRED**

### WiFi Stack

**Drivers:**
- Built into kernel (uImage)
- 802.11n support compiled in

**Configuration:**
- `/etc/wlan/` - WiFi configuration files
- `/proc/wlan0/` - Runtime WiFi control

**Interface:** wlan0

## Build Process

### Building from Source

```bash
# Extract base firmware
cd "/home/gokul/Documents/Hg323 rgw 3.7"
tar -xf firmware_v23_full_protection.tar

# Extract rootfs
unsquashfs -d rootfs_v23 rootfs_FS

# Make modifications
cd rootfs_v23

# Remove unwanted binaries
rm -f bin/diag bin/cupsd bin/lpadmin bin/lpstat bin/lp bin/startcupsd
rm -f bin/ftp bin/ftpd bin/tftp bin/tftpd
rm -f bin/miniupnpd bin/upnpctrl
rm -f bin/radvdump bin/11N_UDPserver bin/udpechoserver

# Remove unwanted libraries
rm -f lib/libcups.so* lib/libmini_upnp.so

# Remove EPON modules
rm -f lib/modules/epon_drv.ko lib/modules/epon_mpcp.ko lib/modules/epon_polling.ko
rm -f lib/modules/scsi_wait_scan.ko

# Remove backup files
rm -f lib/features/*.orig

# Fix WiFi channel
sed -i 's/CH=$(get_free_channel)/CH=12/' etc/rc.d/rc35

# Rebuild squashfs
cd ..
mksquashfs rootfs_v23 rootfs_FS_new -noappend -b 65536 -comp lzma -processors 1

# Package firmware
cp firmware_v23_full_protection.tar firmware_slim_v12.tar
tar --delete -f firmware_slim_v12.tar rootfs_FS
tar -rf firmware_slim_v12.tar rootfs_FS_new
mv rootfs_FS_new rootfs_FS
tar -rf firmware_slim_v12.tar rootfs_FS

# Verify checksum
md5sum firmware_slim_v12.tar
```

### Build Requirements

- Linux system (tested on Debian/Ubuntu)
- squashfs-tools package installed
- Standard GNU tar
- At least 100MB free disk space

## Network Configuration

### Default Settings
- **IP Address:** 192.168.1.1/24
- **DHCP Range:** 192.168.1.100-192.168.1.200
- **Admin User:** admin
- **Admin Pass:** (device-specific, check label)

### GPON Configuration
- **Mode:** Auto (ITU-T G.984)
- **OMCI:** Enabled
- **Registration:** Automatic
- **VLAN:** ISP-dependent

## Support & Resources

### Useful Commands

```bash
# System info
cat /proc/cpuinfo
cat /proc/meminfo
free

# Network interfaces
ifconfig -a
brctl show

# Kernel modules
lsmod
cat /proc/modules

# Running processes
ps -w
top

# Flash partitions
cat /proc/mtd

# Kernel messages
dmesg
cat /proc/kmsg

# GPIO status
cat /proc/gpio
cat /proc/interrupts
```

### Debugging

```bash
# Enable verbose OMCI logging
echo 1 > /proc/omci/debug

# WiFi debug
iwconfig wlan0
iwlist wlan0 scan

# Check kernel symbols
cat /proc/kallsyms | grep <symbol>
```

## Credits

**Firmware Base:** HG323 RGW 3.7
**Optimization:** Custom build - June 2026
**Testing:** BSNL GPON network (India)
**Platform:** Realtek Luna SDK (Linux 2.6.30.9)

## License

This is custom firmware based on proprietary vendor firmware. Use at your own risk. No warranty provided.

## Disclaimer

⚠️ **WARNING:** Flashing custom firmware may:
- Void warranty
- Brick device if done incorrectly
- Cause unexpected behavior
- Violate ISP terms of service

Always keep backup of original firmware and know how to use U-Boot recovery.

---

**Last Updated:** June 6, 2026
**Status:** Production Ready
