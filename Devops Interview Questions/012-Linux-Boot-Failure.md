# 12. Linux Server Boot Failure After Kernel Update

## Scenario
At 6:00 AM on a Saturday, you receive a critical alert: "Server unreachable: legacy-app-01 (192.168.1.200)." This is a bare-metal server hosting a legacy application that processes financial transactions. It cannot be containerized because it depends on proprietary hardware (HSM for encryption) and runs on a custom-compiled kernel module. Yesterday at 5 PM, a security team member applied a kernel security patch (`kernel-4.18.0-425.19.2.el8.x86_64`) and rebooted the server. The server failed to boot and is showing a kernel panic on the physical console. The application hasn't been accessible since the reboot. You need to recover the server without losing data or requiring a full reinstall.

## Interviewer Question
"After a kernel security patch, a production server fails to boot with a kernel panic. This is a bare-metal server hosting a legacy application that cannot be easily containerized. How do you recover?"

## What I Should Think About
- **Boot process stages**: BIOS → GRUB → kernel → initramfs → init/systemd
- **Kernel panic**: What's the exact error message?
- **GRUB options**: Can you boot into a previous kernel?
- **initramfs**: Was it regenerated correctly for the new kernel?
- **Hardware compatibility**: Does the new kernel support the HSM hardware?
- **Custom kernel module**: Was it compiled against the new kernel?
- **Data safety**: Ensure the filesystem is intact before attempting fixes
- **Physical access**: Do you have IPMI/iLO/BMC access?

## Ideal Answer

**Phase 1: Gain Console Access**

Since the server is bare-metal and unreachable over the network, you need physical or out-of-band console access:

```bash
# Via IPMI/BMC
ipmitool -I lanplus -H 192.168.1.201 -U admin -P password sol activate

# Or via a KVM over IP (like Dell iDRAC, HP iLO)
# Access via web interface at https://192.168.1.201
```

**Phase 2: Read the Kernel Panic Message**

The console shows:
```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
Please append a correct "root=" boot option

Kernel Offset: 0x2400000 from 0xffffffff81000000 (relocation range: 0xffffffff80000000-0xffffffffbfffffff)
```

This means the kernel can't find the root filesystem. This is typically caused by:
1. The initramfs doesn't include the necessary filesystem/drivers
2. The root device UUID has changed
3. The kernel module for the disk controller isn't loaded

**Phase 3: Boot into Rescue Mode / Previous Kernel**

```bash
# At the GRUB menu, press 'e' to edit the boot entry
# Or press 'c' for GRUB command line

# Option 1: Boot previous kernel
# In GRUB menu, select the previous kernel version and boot

# Option 2: Edit the boot entry to use init=/bin/bash
# Add to the kernel line:
# init=/bin/bash

# Option 3: Boot into single-user mode
# In GRUB, edit the kernel line and add:
# rd.break or single or init=/bin/sh

# Option 4: Use a rescue disk
# Boot from a live USB/CD or network rescue image
```

**Phase 4: Fix the Issue (from rescue mode)**

```bash
# If booting into rescue mode:

# Mount root filesystem
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot

# If using LVM
vgchange -ay
mount /dev/mapper/vg0-root /mnt

# Chroot into the system
chroot /mnt

# Check and regenerate initramfs
dracut --force --regenerate-all  # RHEL/CentOS
update-initramfs -u -k all       # Debian/Ubuntu

# Check GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg  # RHEL/CentOS
update-grub                           # Debian/Ubuntu

# If the issue is the custom kernel module:
# Check if it was compiled for the new kernel
ls /lib/modules/$(uname -r)/extra/
# Recompile the module for the new kernel
# Or temporarily blacklist it and boot

# Check if the HSM module is causing issues
lsmod | grep hsm
modprobe hsm_module  # Test loading
```

**Phase 5: Handle the Custom Kernel Module**

```bash
# Check if the custom module was compiled for the new kernel
ls /lib/modules/4.18.0-425.19.2.el8.x86_64/extra/

# If not, compile it
cd /usr/src/hsm-module-1.0
make KERNEL_SRC=/lib/modules/$(uname -r)/build
cp hsm.ko /lib/modules/$(uname -r)/extra/
depmod -a

# Or temporarily disable it
echo "blacklist hsm_module" > /etc/modprobe.d/blacklist-hsm.conf
```

**Phase 6: Reboot and Verify**

```bash
# Exit chroot
exit

# Reboot
reboot

# After boot, verify
uname -r
lsmod | grep hsm
systemctl status legacy-app
```

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                 Server Boot Sequence                      │
│                                                          │
│  BIOS/UEFI                                               │
│    │                                                     │
│    ▼                                                     │
│  GRUB2 Bootloader                                        │
│    │                                                     │
│    ├── Kernel 4.18.0-425.19.2 (NEW) ← PANIC            │
│    │   └── initramfs → can't find root FS               │
│    │       └── Missing: HSM driver, filesystem driver    │
│    │                                                     │
│    └── Kernel 4.18.0-425.13.1 (OLD) ← WORKING          │
│        └── initramfs → finds root FS ✓                  │
│                                                          │
│  Recovery Path:                                          │
│  1. Boot into rescue mode (GRUB → single user)          │
│  2. Mount root filesystem                               │
│  3. Chroot into system                                  │
│  4. Regenerate initramfs with dracut                     │
│  5. Recompile HSM kernel module                         │
│  6. Reboot                                              │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │ HSM Hardware (Proprietary)                       │    │
│  │ Requires custom kernel module: hsm_driver.ko     │    │
│  │ Must be compiled against running kernel version  │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

## Investigation

1. **Get console access**: Use IPMI/BMC/KVM to view the server's console output.
2. **Read the kernel panic message**: The exact error message tells you what failed during boot.
3. **Check GRUB menu**: See if the previous kernel is available as a boot option.
4. **Check the initramfs**: If the error is "unable to mount root fs," the initramfs may be missing drivers.
5. **Check custom kernel modules**: Verify that the HSM driver was compiled for the new kernel.
6. **Check GRUB configuration**: Ensure the root device UUID is correct in GRUB and `/etc/fstab`.
7. **Check hardware compatibility**: Verify that the new kernel supports the HSM hardware.
8. **Check boot logs (if accessible)**: `journalctl -b -1` to see the last successful boot logs.

## Commands

```bash
# 1. Access IPMI console
ipmitool -I lanplus -H BMC_IP -U admin -P password sol activate

# 2. At GRUB menu, edit boot entry
# Press 'e' on the kernel entry
# Add: rd.break or single or init=/bin/bash

# 3. From rescue mode, mount filesystems
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

# 4. Chroot
chroot /mnt

# 5. Check kernel version
uname -r
ls /boot/vmlinuz-*

# 6. Check initramfs
ls /boot/initramfs-*
lsinitrd /boot/initramfs-$(uname -r).img | grep hsm

# 7. Regenerate initramfs
dracut --force --regenerate-all
# Or specifically include the HSM module:
dracut --force --add-drivers hsm_module

# 8. Check GRUB configuration
cat /boot/grub2/grub.cfg | grep menuentry
grub2-editenv list

# 9. Regenerate GRUB config
grub2-mkconfig -o /boot/grub2/grub.cfg

# 10. Check and compile custom kernel module
cd /usr/src/hsm-driver-1.0
make clean
make KERNEL_SRC=/lib/modules/$(uname -r)/build
cp hsm.ko /lib/modules/$(uname -r)/extra/
depmod -a
modprobe hsm  # Test loading

# 11. If all else fails, boot previous kernel
# In GRUB menu, select the old kernel version

# 12. Check filesystem integrity
fsck -y /dev/sda2

# 13. Check disk health
smartctl -a /dev/sda

# 14. After successful boot, verify
uname -r
systemctl status legacy-app
dmesg | grep hsm
```

## Root Cause

| Root Cause | Detection | Resolution |
|---|---|---|
| initramfs missing HSM driver | `lsinitrd` doesn't show hsm module | Regenerate initramfs with `dracut --add-drivers hsm` |
| HSM module not compiled for new kernel | `/lib/modules/<new-kernel>/extra/` is empty | Recompile the module against the new kernel's headers |
| GRUB pointing to wrong root UUID | GRUB root= UUID doesn't match `/etc/fstab` | Update GRUB with correct UUID: `grub2-mkconfig` |
| Kernel panic from missing dependency | Kernel panic message shows missing symbol | Identify the missing module and include it in initramfs |
| Hardware incompatibility | New kernel doesn't support HSM hardware | Use previous kernel, or patch the driver |

## Immediate Mitigation

1. **Boot the previous kernel**: At GRUB menu, select the old kernel version. This gets the server back online immediately.
2. **If GRUB doesn't show old kernels**: Boot from a rescue disk, mount the filesystem, and restore the GRUB configuration.
3. **Once booted**: Disable automatic kernel updates to prevent recurrence.
4. **Verify application**: Ensure the legacy application starts and the HSM is accessible.

## Permanent Fix

1. **Test kernel updates in staging first**: Apply the kernel update to a non-production server with the same configuration.
2. **Maintain kernel module build infrastructure**: Ensure the HSM module can be rebuilt for any kernel version.
3. **Keep previous kernels**: Don't remove old kernels from GRUB. Set `installonly_limit` to 3.
4. **Automate initramfs regeneration**: After any kernel module change, automatically run `dracut --force`.
5. **Document the recovery procedure**: Create a runbook for kernel panic recovery.

## Monitoring

- **Server availability**: Monitor that the server is reachable after reboot.
- **Application health**: Verify the legacy application is serving traffic.
- **Kernel version tracking**: Track which kernel version is running on each server.
- **HSM connectivity**: Monitor that the HSM is accessible and responding.
- **Boot time**: Monitor boot duration to detect boot issues early.

## Security

- **Kernel security patches**: Don't skip kernel security patches — find a way to test and apply them safely.
- **Boot security**: Ensure GRUB is password-protected to prevent unauthorized boot option changes.
- **Secure Boot**: If using UEFI Secure Boot, ensure custom kernel modules are signed.
- **Recovery access**: Secure IPMI/BMC access with strong credentials and network isolation.

## Production Considerations

- **RTO impact**: This server has been down since 5 PM yesterday — over 13 hours. This is a significant outage.
- **Data integrity**: Verify that the filesystem and database are intact after the recovery.
- **HA**: This is a single bare-metal server — no HA. Consider implementing a standby server.
- **Legacy application**: This application needs a modernization plan to reduce dependency on bare-metal hardware.
- **Change management**: Kernel updates to production servers should go through a change management process with testing.

## Senior-Level Answer

"I'd access the server via IPMI/BMC to view the kernel panic message. If GRUB shows the previous kernel, I'd boot into it immediately to restore service. If not, I'd boot from a rescue disk, mount the root filesystem, chroot in, and regenerate the initramfs with `dracut --force --add-drivers hsm_module` to include the HSM driver. I'd also verify the HSM kernel module is compiled for the new kernel. After recovery, I'd implement a kernel update testing process and ensure previous kernels are always available in GRUB as a fallback."

## Architect-Level Answer

"This incident exposes a critical single point of failure: a legacy application on bare-metal hardware with a custom kernel module that breaks on kernel updates. The long-term solution is threefold: First, modernize the application architecture to decouple it from the HSM hardware — use an HSM proxy service or cloud HSM that provides a hardware-independent API. Second, implement a proper testing pipeline for kernel updates: replicate the production environment in a staging server, test the kernel update, verify the HSM module, then roll out with a canary approach. Third, plan for application modernization to containerize or migrate the legacy application, eliminating the bare-metal dependency entirely. For immediate resilience, implement a standby server with the same configuration that can be brought online if the primary fails."

## Follow-Up Questions

1. "Explain the Linux boot process in detail. What happens at each stage from power-on to login prompt?"
2. "What is initramfs and why is it needed? How does it differ from initrd?"
3. "How does GRUB2 work? What's the difference between `grub2-mkconfig` and manual GRUB configuration?"
4. "How would you implement automated kernel update testing in a CI/CD pipeline for bare-metal servers?"
5. "What is the difference between a kernel panic and a kernel oops? How does the system handle each?"
