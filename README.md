# **My Fedora 34 VFIO/GPU Passthrough Guide**

This guide was originally a personal reference, so it is tailored to how I would ideally set up a Virtual Machine (VM) with GPU passthrough, with the intent of being able to play games in a virtualised environment. This guide relies on having two GPUs, one for the host, this can be a Dedicated GPU (dGPU) or Integrated GPU (iGPU), and one for the guest, almost always a Dedicated GPU. Getting GPU acceleration in a VM is easy, however, optimising for high and consistent framerates is something that requires a bit of tweaking.

<br>

### **Why would someone consider this?**
There are many reasons why one may consider virtualising their gaming, CAD or video editing workstation. I will briefly mention some of the reasons:

- **Privacy:** In the context of this guide, this is perhaps the #1 reason for perusing a setup like this. It is no lie that Windows 10 is a [privacy nightmare](https://www.privacytools.io/operating-systems/#win10). A nice, isolated environment, a cage, is the best place for this kind of software. Therefore, its access to your hardware and software is appropriately limited.

- **Security:** Sandboxing untrusted or vulnerable software is a good practice to have, as there is always a risk of something being compromised. By nature, the VM is isolated, so having this segmentation of physical resources greatly reduces the potential impact on the hypervisor/host. However, it is important to be aware that there are vulnerabilities and methods of "escaping" a VM, so this is by no means a silver bullet, as there are other security considerations to take into account (networking, hardware vulnerabilities, ... ). 

- **Cost:** Why build another computer just for games or CAD? You can throw in an extra GPU or two into your current PC, which most likely allows for this kind of setup. The same idea applies with servers, which is why virtualisation and software defined infrastructure is so popular in the enterprise/data center.

- **Flexibility:** Depending on how your VM disk is set up, it can be snapshotted, rolled back, backed up or destroyed. This makes it easy to revert Windows 10 to a previous state when something inevitably goes wrong after a while, or even reinstall windows without it affecting your host. (This guide does not show how to go about having a VM that can be snapshotted, but it is possible to set up with ZFS for example, however, using a Copy on Write (CoW) filesystem can come with its own performance overhead).

<br>

### **Prerequisites for this guide**
- 2x GPUs: A GPU for the Host (Fedora), and one for the VM (Windows 10)
  - For the host GPU I recommend an AMD GPU, since these drivers are open sources and play nicer with Linux
  - For the Guest GPU I recommend an Nvidia GPU as AMD GPUs are plagued with reset issues (although it's starting to look better on AMD's side of things)
- A motherboard that has proper IOMMU Groupings. So, hardware can be isolated and passed through to a VM on an individual basis.
- A CPU capable of Virtualisation Extensions, you may need to turn these options on in the BIOS. Any competent CPU made within the last 5 or so years will have these features.

**Quality of Life Prerequisites**
- An extra USB Controller for passthrough. Passing through a USB controller makes your life easier. Especially when you want to plug things like Bluetooth adapters into the VM and other USB devices. I strongly recommend getting a motherboard that has two USB controllers or suitable groupings to be used with a PCIe USB card. 

<br>

### **What will be achieved after following this guide:**
- A working VM with its own dedicated GPU (The main idea of this guide)
- Easy keyboard and mouse switching between the VM and Host
    - This also includes evdev persistence, so you can use a USB switch for external devices (such as a laptop) alongside this setup without losing keyboard and mouse access to the VM.
- USB Controller passthrough
- VM/Hypervisors Optimisations
    - CPU Core pinning
    - CPU Core isolation
    - Hyper-v Enlightenments
    - Static hugepages
- Guest Optimisations
    - Dealing with MSI Interrupts
    - De-bloating Windows 10

<br>

### **What this guide is not:**
- A tutorial to fully mask the use of KVM. You want Windows 10 to know it's in a VM for performance optimisations.
- A Guide for Nested Virtualisation with Hyper-V.
- A single GPU passthrough guide. This is a meme setup which serves almost no practical purpose in this context. Just dual boot.
- A vGPU Unlocking Guide.

<br>
<br>
<br>

## **First install all the required packages**
#
```
sudo dnf install @virtualization
```

<br>

## **Change GRUB settings**

#

Edit `/etc/default/grub` and add the following to `GRUB_CMDLINE_LINUX=`

- `intel_iommu=on` allows for iommu for intel (amd_iommu=on for amd)

- `kvm.ignore_msrs=1` Can be used to stop Windows 10 from blue screening from GPU-Z and other stuff that uses GPU

- `rd.driver.pre=vfio-pci` Enables the vfio-pci driver


- `systemd.unified_cgroup_hierarchy=0` Allows for CPU isolation (patch for cset needed) to go back to V1. V2 is enabled on >Fedora 31 be default. Only use this is you want dynamic CPU isolation.

- `transparent_hugepage=never default_hugepagesz=1G` So the RAM for the VM is allocated in one chunk for better performance of programs that use a large amount of memory. This probably isn't needed for most usecases, this needs to be benchmarked to see if theres any tangible benefit.

<br>

Everything to use in one line (omit as needed): `intel_iommu=on kvm.ignore_msrs=1 rd.driver.pre=vfio-pci systemd.unified_cgroup_hierarchy=0 transparent_hugepage=never default_hugepagesz=1G`

### Grub should look something like this:

```
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="resume=/dev/mapper/fedora_localhost--live-swap rd.lvm.lv=fedora_localhost-live/root rd.lvm.lv=fedora_localhost-live/swap rhgb quiet intel_iommu=on kvm.ignore_msrs=1 rd.driver.pre=vfio-pci systemd.unified_cgroup_hierarchy=0 transparent_hugepage=never default_hugepagesz=1G"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
```

### Reload grub

```
sudo grub2-mkconfig -o /etc/grub2-efi.cfg
```

<br>

## **Enable nested virtualisation**

#

Edit `/etc/modprobe.d/kvm.conf` (For intel or amd)

```
options kvm_intel nested=1
options kvm_amd nested=1
```

<br>

## **Add VFIO drivers to the initialramdisk**

#

Edit `/etc/dracut.conf.d/vfio.conf`

```
add_drivers+="vfio vfio_iommu_type1 vfio_pci"
```

Reload initial ramdisk

```bash
sudo dracut -f --kver `uname -r`
```

<br>

### **Restart the PC. IOMMU and the drivers should be enabled now.**

<br>

### Use this script to show IOMMU groups:

```bash
#!/bin/bash
for d in /sys/kernel/iommu_groups/_/devices/_; do
n=${d#*/iommu_groups/*}; n=${n%%/_}
printf 'IOMMU Group %s ' "$n"
  lspci -nns "${d##_/}"
done
```

Edit `/etc/modprobe.d/vfio.conf` with the ids you have gotten from the script (Use the GPU and the associated audio device). This is my GTX 1080 Ti.

```
options vfio-pci ids=1002:6938,1002:aad8
```

<br>

## **Set up the VM in Virt Manager**

#
 **Don't start right away after creating, some setting need to be changed after the initial setup in the virt manager GUI**

- VirtManager -> New VM, check customise before install option (USE Q35 Chipset)

- Overview tab -> Firmware -> UEFI

- CPUs -> Copy Host configuration, Set topology eg: 5c/10t is 1 Socket, 5 cores and 2 threads.

- Memory -> As you would like (Look later for hugepages)

- Disks -> Add hardware -> Controller: Type = SCSI, Model = VirtIO SCSI

- Disks -> Add hardware -> Storage -> Bus = SCSI, Cache mode = none, IO mode = threads (If multiple cores on host available for disk stuff. If just 1c/2t then "native" is good), Discard mode = unmap, Detect zeros = default, Storage format = raw (In create custom storage).

In the SCSI controller XML Tab, add the following underneath the controller tag:

```xml
<controller>
  <driver queues="8" iothread="1"/>
  ...
</controller>
```

For more info on above see:
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Examples_with_libvirt <br>
Reference for virtio disk settings:
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide/sect-virtualization_tuning_optimization_guide-virt_manager-virtual-disk_options

<br>

To show the boot disk as an SSD in Windows 10, put this at the bottom of the XML:
(Mainly a preference thing, not sure if it affects how windows treats the drive, this needs to be investigated)

To show disk IDs if using multiple disks: `virsh qemu-monitor-command --hmp 1 "info qtree"` (Replace 1 with VM name) https://blog.christophersmart.com/2019/12/18/kvm-guests-with-emulated-ssd-and-nvme-drives/

```xml
<qemu:commandline>
  <qemu:arg value='-set'/>
  <qemu:arg value='device.scsi0-0-0-0.rotation_rate=1'/>
</qemu:commandline>
```

When passing through a partition (using `virtio-scsi`) with a LUKS backend, make sure its mounted before the VM boots. In the XML it'll look something like this:

```xml
<disk type="block" device="disk">
      <driver name="qemu" type="raw" cache="none" io="native" discard="unmap"/>
      <source dev="/dev/disk/by-id/dm-name-yeet"/>
      <target dev="sdb" bus="scsi"/>
      <address type="drive" controller="0" bus="0" target="0" unit="1"/>
    </disk>
```
- NIC -> VirtIO
- Add Hardware -> PCI Host Device -> GPU (GPU & Audio) & USB Controller
  - Change USB Controller to USB3
- Remove unneeded hardware (ie: spice, audio, ...)
- **ADD THE CD DRIVES FOR WINDOWS 10 AND VIRTIO DRIVERS**

<br>

"Mask" KVM for Nvidia drivers to work, add Hyper-v Enlightenments and Clock Settings. Remember to add these in the correct tags of the `XML` file, and keep the correct indentations. Some of the items here also contribute to optimising the VM.  

```xml
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
...
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on">
        <direct state="on"/>
      </stimer>
      <reset state="on"/>
      <vendor_id state="on" value="other"/>
      <frequencies state="on"/>
      <reenlightenment state="on"/>
      <tlbflush state="on"/>
      <ipi state="on"/>
      <evmcs state="off"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    <vmport state="off"/>
    <ioapic driver="kvm"/>
  </features>
...
<clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
    <timer name="tsc" present="yes" mode="native"/>
</clock>
```

<br>

## **Correct CPU Information Passthrough**
#

Info on libvirt settings/tags: https://libvirt.org/formatdomain.html#elementsFeatures

In the `<cpu>` tag put, to show correct CPU info (This is useful for when the system wants information on the CPU you're using. This way it'll be the correct model and cache layout).

```xml
  <cpu mode="host-passthrough" check="none">
    ...
    <cache mode="passthrough"/>
  </cpu>
```

<br>

## **evdev keyboard and mouse switching**

#

https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Passing_keyboard/mouse_via_Evdev

See usb devices that you want to pass through (usualy the ones with event in the name)

```
ls -l /dev/input/by-id/
```

Put into the xml similar to this

```xml
<qemu:commandline>
<qemu:arg value='-object'/>
<qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/usb-Logitech_Gaming_Mouse_G400-event-mouse'/>
<qemu:arg value='-object'/>
<qemu:arg value='input-linux,id=kbd2,evdev=/dev/input/by-id/usb-04d9_USB_Keyboard-event-if02'/>
<qemu:arg value='-object'/>
<qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/usb-04d9_USB_Keyboard-event-kbd,grab_all=on,repeat'/>
</qemu:commandline>
```

Also put the following or add it via virt manager GUI

```xml
<input type='mouse' bus='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0e' function='0x0'/>
    </input>
    <input type='keyboard' bus='virtio'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x0f' function='0x0'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
```

To allow evdev for qemu edit `/etc/libvirt/qemu.conf` to look something like this:

```
user = "flux"
group = "kvm"
cgroup_device_acl = [
        "/dev/null", "/dev/full", "/dev/zero",
        "/dev/random", "/dev/urandom",
        "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
        "/dev/rtc","/dev/hpet", "/dev/sev",
        "/dev/input/by-id/usb-04d9_USB_Keyboard-event-kbd",
        "/dev/input/by-id/usb-04d9_USB_Keyboard-event-if02",
        "/dev/input/by-id/usb-Logitech_Gaming_Mouse_G400-mouse",
        "/dev/input/by-id/usb-Logitech_Gaming_Mouse_G400-event-mouse"
]
```

Make sure the user is a part of the input and kvm group

```
sudo usermod -aG kvm,input flux
```

Now restart libvirtd

```
systemctl restart libvitd
```

<br>

### **Try to boot the machine, you should get a SELinux error**
To make the rules for SELinux, we need to induce the error first, so the first boot wont work since the error will happen, this is ok, we will fix this now.

<br>

## **SELinux**

#

To get working with SELinux enabled use the audit to allow to allow the device capture, see below link
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-fixing_problems-allowing_access_audit2allow

Quick steps:

1. Check the error in `/var/log/audit/audit.log`
2. Run the `audit2allow -w -a` command for human readable output of why access denied
3. Run the `audit2allow -a` command to view the Type Enforcement rule that allows the denied access:
4. To use the rule displayed by `audit2allow -a`, run the `audit2allow -a -M <name of module>` command to make a Type Enforcement file (`.te`) in the current directory
5. Then install the module (the `.pp` file created alongside the `.te` file): `semodule -i mycertwatch.pp`

<br>

## **Persistent evdev**

#

Get persistent stuff working so the KB/M can be unplugged and replugged and still go to the VM. If you don't want this, skip to the end of this section.

See: https://github.com/aiberia/persistent-evdev

Make sure `python3-devel` `pip pyudev` and `sudo pip evdev` are installed
<br>

**Note: For me it only works with one keyboard and mouse device or else mouse wont swap back to host**

Place scripts in the right spots:

`config.json` & `persistent-evdev.py` can go anywhere

`persistent-evdev.service` goes in `/etc/systemd/system/` (Make sure this references the location of where you put those two previous files)

`60-persistent-input-uinput.rules` goes in `/etc/udev/rules.d/`

Make cache file (its actually a directory, the location is specified in config.json) `/var/cache/persistent-evdev`

Because we are making the keyboard and mouse as new devices (via proxy), restart the udev service

```
systemctl restart systemd-udevd
```

enable the new persistent-evdev.service

```
systemctl enable persistent-evdev
```

Confirm new devices exist

```
ls -l /dev/input/by-id
```

Should see output similar to this

```
lrwxrwxrwx. 1 root root 10 Jul 13 20:43 uinput-persist-keyboard0 -> ../event23
lrwxrwxrwx. 1 root root 9 Jul 13 20:43 uinput-persist-mouse0 -> ../event9
```

Then edit `/etc/libvirt/qemu.conf` and replace the other kb/m devices with these

```
    "/dev/input/by-id/uinput-persist-keyboard0",
    "/dev/input/by-id/uinput-persist-mouse0"
```

Also edit the `XML` file for the VM to reflect the name changes as shown above.

Restart libvirtd

```
systemctl restart libvirtd
```

### **Persistent evdev input done**

<br>

### **Test that the machine boots up then stop the machine**
Make sure you can see the Tiano Core screen from the output of the passthrough GPU, then force off the VM for now. If you see this, the evdev part is working. Good, move on.

<br>


## **Memory Balooning**
#

Disable memballoon (near the end of the XML). If left unchecked, it can be detrimental to the performance of a VFIO VM.

```xml
   <memballoon model='none'>
```

## **CPU Pinning and iothreads**
#
I leave core 0 and it's associated thread, 6, for the host as there are some kernel tasks that are always on these cores. The following shows which cores are paired with which thread

```
virsh capabilities
```

In the XML put the following (Core `0` and it's thread `6` stays on the host, everything else is going to the VM)

```xml
<vcpu placement="static">10</vcpu>
<iothreads>1</iothreads>
<cputune>
<vcpupin vcpu="0" cpuset="1"/>
<vcpupin vcpu="1" cpuset="7"/>
<vcpupin vcpu="2" cpuset="2"/>
<vcpupin vcpu="3" cpuset="8"/>
<vcpupin vcpu="4" cpuset="3"/>
<vcpupin vcpu="5" cpuset="9"/>
<vcpupin vcpu="6" cpuset="4"/>
<vcpupin vcpu="7" cpuset="10"/>
<vcpupin vcpu="8" cpuset="5"/>
<vcpupin vcpu="9" cpuset="11"/>
<emulatorpin cpuset="0,6"/>
<iothreadpin iothread="1" cpuset="0,6"/>
</cputune>
```

## **CPU Frequency Governor**
#

Adjust the linux CPU governor for max turbo 100% of the time (It can be added to qemu hook but I do it this way, so its effective on the host too). You can choose to skip this part if you want, it will use more power, but the latency of dynamically speeding up/slowing down the CPU will be reduced. 

```
sudo dnf install kernel-tools
sudo systemctl enable cpupower.service
sudo systemctl start cpupower.service
sudo cpupower frequency-set --governor performance
```

Check

```
cat /sys/devices/system/cpu/cpu\*/cpufreq/scaling_governor
```

Checking CPU Frequency live

```
watch -n1 'grep MHz /proc/cpuinfo'
```

<br>

## **Install Windows 10**
#
### **Boot up the VM and start the Windows 10 install proccess**

You will not see the virtual hard disk present so you'll need to make sure that the virtio driver disc is attached so you can browse it and install the VirtIO SCSI driver to see the virtual disk. Install Windows 10.

Once Windows 10 is installed, remember to install the following drivers From the virtio drivers ISO:

- VirtIO input drivers for evdev stuff (Under Human Interface Devices in controll device manager). Or else you might be faces with random input glitches.

- NetKVM driver for ethernet

- QEMU Guest agent

<br>

## **CPU Isolation**

#

To be able to dynamically isolate the pinned CPUs upon VM boot (and free the CPU on shutdown), use cpuset and patch it, or if you will be using the VM the majority of the time, just do it through grub. Doing it through GRUB is a better way to isolate the CPU, as it completely isolates the CPUs from the kernel (Mentioned in the Arch wiki page).
This is mainly needed if your Host will be under load as well as the Guest during normal operation. I found that this almost completely eliminates microstuttering. 

```
GRUB_CMDLINE_LINUX=" ... isolcpus=4-7 nohz_full=4-7 rcu_nocbs=4-7"
```

- https://github.com/lpechacek/cpuset (Build instructions in INSTALL file)

- https://rokups.github.io/#!pages/gaming-vm-performance.md

cpuset patch:

```diff
diff --git a/cpuset/commands/set.py b/cpuset/commands/set.py
index 9205a07..e299d70 100644
--- a/cpuset/commands/set.py
+++ b/cpuset/commands/set.py
@@ -410,8 +410,8 @@ def modify(name, cpuspec=None, memspec=None, cx=None, mx=None):
     log.debug('modifying cpuset "%s"', nset.name)
     if cpuspec: nset.cpus = cpuspec
     if memspec: nset.mems = memspec
-    if cx: nset.cpu_exclusive = cx
-    if mx: nset.mem_exclusive = mx
+    # if cx: nset.cpu_exclusive = cx
+    # if mx: nset.mem_exclusive = mx

 def active(name):
     """check that cpuset by name or cset is ready to be used"""
diff --git a/cset b/cset
index 9320532..83e4a0c 100755
--- a/cset
+++ b/cset
@@ -1,4 +1,4 @@
-#!/usr/bin/env python
+#!/usr/bin/env python2

 __copyright__ = """
 Copyright (C) 2007-2010 Novell Inc.
diff --git a/setup.py b/setup.py
index 5934cb0..54a2f18 100755
--- a/setup.py
+++ b/setup.py
@@ -10,7 +10,7 @@ setup(name = 'cpuset',
     license = 'GPLv2',
     author = 'Alex Tsariounov',
     author_email = 'alext@novell.com',
-    url = 'https://github.com/lpechacek/cpuset'
+    url = 'https://github.com/lpechacek/cpuset',
     description = 'Allows manipluation of cpusets and provides higher level functions.',
     long_description = \
         'Cpuset is a Python application to make using the cpusets facilities in the Linux\n'
```

Edit/make a qemu hook in `/etc/libvirt/hooks` the file will be called qemu so the full path is `/etc/libvirt/hooks/qemu`

These are automatically executed at boot/shutdown of VMs

Can use this tool to check cpu affinity for the mask: https://bitsum.com/tools/cpu-affinity-calculator/

The script to live in there is (Adjust the VM names as required, can use for multiple VMs):

```bash
#!/usr/bin/env bash

#
# Original author: Rokas Kupstys <rokups@zoho.com>
# Heavily modified by: Danny Lin <danny@kdrag0n.dev>
#
# This hook uses the `cset` tool to dynamically isolate and unisolate CPUs using
# the kernel's cgroup cpusets feature. While it's not as effective as
# full kernel-level scheduler and timekeeping isolation, it still does wonders
# for VM latency as compared to not isolating CPUs at all. Note that vCPU thread
# affinity is a must for this to work properly.
#
# Original source: https://rokups.github.io/#!pages/gaming-vm-performance.md
#
# Target file locations:
#   - $SYSCONFDIR/hooks/qemu.d/vm_name/prepare/begin/cset.sh
#   - $SYSCONFDIR/hooks/qemu.d/vm_name/release/end/cset.sh
# $SYSCONFDIR is usually /etc/libvirt.
#

TOTAL_CORES='0-11'
TOTAL_CORES_MASK=FFF            # 0-11, bitmask 0b11111111
HOST_CORES='0,6'                # Cores reserved for host
HOST_CORES_MASK=41              # 0,4, bitmask 0b00010001
VIRT_CORES='1-5,7-11'            # Cores reserved for virtual machine(s)

#VM_NAME="$1"
VM_NAME="$1"
VM_ACTION="$2/$3"

function shield_vm() {
    cset -m set -c $TOTAL_CORES -s machine.slice
    cset -m shield --kthread on --cpu $VIRT_CORES
}

function unshield_vm() {
    cset -m shield --reset
}

# For convenient manual invocation
if [[ "$VM_NAME" == "shield" ]]; then
    shield_vm
    exit
elif [[ "$VM_NAME" == "unshield" ]]; then
    unshield_vm
    exit
fi

if [[ "$VM_NAME" = "Win10" || "$VM_NAME" = "VM2" || "$VM_NAME" = "VM3" ]]; then

    if [[ "$VM_ACTION" == "prepare/begin" ]]; then
        echo "libvirt-qemu cset: Reserving CPUs $VIRT_CORES for VM $VM_NAME" > /dev/kmsg 2>&1
        shield_vm > /dev/kmsg 2>&1

        # the kernel's dirty page writeback mechanism uses kthread workers. They introduce
        # massive arbitrary latencies when doing disk writes on the host and aren't
        # migrated by cset. Restrict the workqueue to use only cpu 0.
        echo $HOST_CORES_MASK > /sys/bus/workqueue/devices/writeback/cpumask
        echo 0 > /sys/bus/workqueue/devices/writeback/numa

        echo "libvirt-qemu cset: Successfully reserved CPUs $VIRT_CORES" > /dev/kmsg 2>&1
    elif [[ "$VM_ACTION" == "release/end" ]]; then
        echo "libvirt-qemu cset: Releasing CPUs $VIRT_CORES from VM $VM_NAME" > /dev/kmsg 2>&1
        unshield_vm > /dev/kmsg 2>&1

        # Revert changes made to the writeback workqueue
        echo $TOTAL_CORES_MASK > /sys/bus/workqueue/devices/writeback/cpumask
        echo 1 > /sys/bus/workqueue/devices/writeback/numa

        echo "libvirt-qemu cset: Successfully released CPUs $VIRT_CORES" > /dev/kmsg 2>&1
    fi

fi
```

<br>

## **Hugepages**

#

Make sure transparent hugepage in grub is disabled as shown above in the grub configuration section. As I said not everyone needs this, and I have not been tested to see if there is a real world difference in a "gaming" scenario.

- In `/etc/sysctl.conf` put: `vm.nr_hugepages = 24`

- Depending on the size of the hugepages can use 1G or leave default (2M/2048kb)

  - If wanting 1G pages put the following in `/etc/default/grub` -> `GRUB_CMDLINE_LINUX= ... default_hugepagesz=1G`

Reload grub

```
sudo grub2-mkconfig -o /etc/grub2-efi.cfg
```

Change the memory backing to use hugepages

```xml
<memory unit="KiB">16777216</memory>
<currentMemory unit="KiB">16777216</currentMemory>
<memoryBacking>
<hugepages/>
</memoryBacking>
<vcpu placement="static">10</vcpu>
```

### **All optimisations from the host side has now been applied. Performance and latency will now be really good, the typical micro stutters that come with VFIO should now be completely mitigated. Time to optimise stuff on the Guest's side.**

<br>
<br>

## **Guest Optimisiations**
#

The idea is to achieve an average idle latency of the system of about under 50us. This is checked with latencymon.

## **MSI Interupts**

#

See explination here: https://forums.guru3d.com/threads/windows-line-based-vs-message-signaled-based-interrupts-msi-tool.378044/

And here: https://forum.nixhaven.com/index.php?threads/using-the-msi-utility-to-reduce-msr-inturrupts-reduce-latency-significantly.19/

Just in case link doesnt work:

- Download the MSI utility: https://github.com/CHEF-KOCH/MSI-utility/releases **This GH is down look for alternatives**

- Run it as an administrator

- Go to Device Manager

- Click in menu "View -> Resources by type".

- Expand "Interrupt request (IRQ)"

- Scroll down to "(PCI) 0x... (...) device name"

- Devices with positive number for IRQ (like "(PCI) 0x00000011 (17) ...") are in Line-based interrupts-mode. Devices with negative number for IRQ (like "(PCI) 0xFFFFFFFA (-6) ...") are in Message Signaled-based Interrupts-mode.

- Open the MSI_Utility as administrator and enable any devices that are in Line-Based Mode (positive numbers). Most recommended devices to check are GPUs and Intel XCI USB controllers.

- Reboot.

<br>

## **Debloat Windows 10 (As of 2004)**

#

**If you are just playing games or would like a _very_ lean Windows 10 setup follow these steps. This assumes you'll be responsible with how you use windows as some features/safety nets will be removed.**

<br>

Use the following tool to debloat/setup Windows 10: https://github.com/farag2/Windows-10-Setup-Script

### Disable Search Index Options

See: https://nixhaven.com/index.php?threads/optimizations-for-kvm-virtual.23/

Start by disabling Windows search indexing. This can take up precious I/O or CPU cycles.

- Open Start Menu and search indexing options, and select the result of the same name.

- Select Modify at the bottom to manage the indexing locations

- Uncheck all the locations.

- Check the Advanced options once you are done. Make sure that the options "index encrypted files" and "treat similar words with diacritics as different words" are not selected.

### AND disable content indexation

- Open File Explorer.

- Right-click on the drive and select properties from the context menu.

- Go to the General tab if it does not open automatically.

- Remove the checkmark from "Allow files on this drive to have contents indexed in addition to file properties".

- Confirm the Attribute changes by selecting "apply changes to drive, subfolders and files, and click ok.

- The process may take a while before it completes. It can run for minutes and even longer than that depending on the size of the drive. You may get an access denied error. I suggest you select "ignore all" when that happens to tell Windows that it should ignore any future access denied error automatically.

### Also

- Disable superfetch

- Disable Windows Defender

### **Done with guest optimisations**
