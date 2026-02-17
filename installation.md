# Creating an Ubuntu Virtual Machine with SDL / Libvirt / Vulkan / Qemu Venus

**Warning**:

* This is a basic procedure that can certainly be improved. I haven't spent endless time optimizing everything.
* I tested and validated this procedure with an **Ubuntu 25.10 host** and an **Ubuntu 25.10 guest** virtual machine.
* I am connected with my local user (not as root).

This document describes how to create an Ubuntu virtual machine on an Ubuntu host using Libvirt, Vulkan, and Qemu/KVM Venus to achieve 3D acceleration within the virtual machine. We will use SDL for the display to benefit from the highest possible performance.

* Refresh rate equivalent to the screen managed by the local GPU.
* Vulkan / OpenGL 3D acceleration inside the virtual machine.
* **Warning**: Due to the use of SDL, clipboard sharing between the host and the virtual machine does not work.

## I. Virsh command helper

```bash
virsh list --all
virsh dumpxml <vm name>

```

## II. Host Configuration (Your PC)

### A. Package Installation

```bash
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager libvirglrenderer1  spice-vdagent apparmor-utils vulkan-tools mesa-vulkan-drivers 

```

### B. Configuration

Here, we will primarily grant the necessary rights to `libvirt`, which does **NOT** run with the privileges of the user executing the virtual machine, allowing it to access devices and the screen.

#### 1. Actions

```bash
sudo aa-disable /etc/apparmor.d/usr.sbin.libvirtd 
sudo setfacl -m u:libvirt-qemu:x /run/user/1000
sudo setfacl -m u:libvirt-qemu:rw /run/user/1000/pipewire-0

```

```bash
sudo usermod -aG render $USER
sudo usermod -aG kvm $USER
sudo usermod -aG render libvirt-qemu
sudo usermod -aG video libvirt-qemu

```

```bash
xhost +local:

```

#### 2. AppArmor Config

```bash
sudo vi /etc/apparmor.d/abstractions/libvirt-qemu

```

* Add the following to the end:

```
  /dev/dri/renderD* rw,
  /dev/dri/card* rw,
  /usr/share/vulkan/icd.d/ r,
  /usr/share/vulkan/icd.d/*.json r,
  /run/user/1000/pipewire-0 rw,

```

#### 3. Qemu Config

```bash
sudo nano /etc/libvirt/qemu.conf

```

* Edit `seccomp_sandbox`:

```
seccomp_sandbox = 0

```

#### 4. Restart daemon

```bash
sudo systemctl restart apparmor
sudo systemctl restart libvirtd

```

#### 5. WARNING: After every REBOOT

* After every reboot, you must re-execute:

```bash
sudo setfacl -m u:libvirt-qemu:x /run/user/1000
sudo setfacl -m u:libvirt-qemu:rw /run/user/1000/pipewire-0
xhost +local:
sudo systemctl restart apparmor
sudo systemctl restart libvirtd

```

* **Do not use `virt-manager` anymore** (use `virsh` instead) to modify the VM configuration.

It is no longer capable of handling the XML configuration. You must use `virsh`.

Virt-manager will delete manual additions or add devices incompatible with this configuration.

## III. Creating a Virtual Machine

Create a virtual machine using `virt-manager`.

You must create a Linux virtual machine. There are no drivers for Windows or MacOS.

Therefore, you cannot benefit from 3D acceleration with Vulkan/Venus on Windows or MacOS.

## IV. Guest VM Configuration

### A. Edit VM configuration with virsh

After creating the virtual machine, we will edit its configuration with `virsh`. Virt-manager does not yet know how to modify the parameters we need.

```bash
virsh edit <vm name>

```

#### 1. Changes

Replace:

```xml
<domain type='kvm'>

```

with:

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>

```

Replace:

```xml
# Spice

    <audio id='1' type='spice'/> 

```

with:

```xml
# Pipewire

    <audio id='1' type='pipewire'/> 

```

#### 2. Optional: Check / Modify boot device

This check is necessary if you haven't finished the VM installation in point III, because `virt-manager` will "eject" the virtual CDROM, and you won't be able to boot from it to install the virtual machine.

```xml
# CDROM

    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/ubuntu-25.10-desktop-amd64.iso'/> # Boot CDROM ISO
      <target dev='sda' bus='sata'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>

```

```xml
# Boot

  <os>
    <type arch='x86_64' machine='pc-q35-10.1'>hvm</type>
    <boot dev='cdrom'/> # "cdrom" must be present
    <boot dev='hd'/>
  </os>

```

#### 3. Delete

* Delete video devices:

```xml
# video devices

    <video>
            xxxx
    </video>

```

* Delete graphics devices:

```xml
# graphics devices

    <graphics type='spice' autoport='yes'>
            xxxx
    </graphics>

```

* Delete Spice-related devices:

```xml
# Spices relates

    <channel type='spicevmc'>
      <target type='virtio' name='com.redhat.spice.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='2'/>
    </channel>

    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='2'/>
    </redirdev>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='3'/>
    </redirdev>

```

#### 4. Add

* Add just before the last `</devices>` tag:

```xml
# Graphics/videos devices

    <graphics type='egl-headless'>
      <gl rendernode='/dev/dri/renderD128'/>
    </graphics>

    <video>
      <model type='none'/>
    </video>    

```

* Add just before the last `</domain>` tag:

```xml
# Qemu overwrite

  <qemu:commandline>
    <qemu:arg value='-device'/>
    <qemu:arg value='virtio-vga-gl,id=video1,max_outputs=1,blob=true,hostmem=8G,venus=true'/>
    <qemu:arg value='-vga'/>
    <qemu:arg value='none'/>
    <qemu:arg value='-display'/>
    <qemu:arg value='sdl,gl=on,show-cursor=on'/>
    <qemu:env name='DISPLAY' value=':0'/>
    <qemu:env name='WAYLAND_DISPLAY' value='wayland-0'/>
    <qemu:env name='XDG_RUNTIME_DIR' value='/run/user/1000'/>
  </qemu:commandline>

```

### B. XML (VM Configuration)

* Configuration Example

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>ubuntu25.10</name>
  <uuid>e9bc3cf3-82fd-4bf2-a00b-23325f9551fe</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://ubuntu.com/ubuntu/25.10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit='KiB'>8388608</memory>
  <currentMemory unit='KiB'>8388608</currentMemory>
  <memoryBacking>
    <source type='memfd'/>
    <access mode='shared'/>
  </memoryBacking>
  <vcpu placement='static'>4</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-10.1'>hvm</type>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <vmport state='off'/>
  </features>
  <cpu mode='host-passthrough' check='none' migratable='on'/>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' discard='unmap'/>
      <source file='/var/lib/libvirt/images/ubuntu25.10.snapshot1'/>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <controller type='usb' index='0' model='qemu-xhci' ports='15'>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
    </controller>
    <controller type='pci' index='0' model='pcie-root'/>
    <controller type='pci' index='1' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='1' port='0x10'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0' multifunction='on'/>
    </controller>
    <controller type='pci' index='2' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='2' port='0x11'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x1'/>
    </controller>
    <controller type='pci' index='3' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='3' port='0x12'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x2'/>
    </controller>
    <controller type='pci' index='4' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='4' port='0x13'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x3'/>
    </controller>
    <controller type='pci' index='5' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='5' port='0x14'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x4'/>
    </controller>
    <controller type='pci' index='6' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='6' port='0x15'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x5'/>
    </controller>
    <controller type='pci' index='7' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='7' port='0x16'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x6'/>
    </controller>
    <controller type='pci' index='8' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='8' port='0x17'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x7'/>
    </controller>
    <controller type='pci' index='9' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='9' port='0x18'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0' multifunction='on'/>
    </controller>
    <controller type='pci' index='10' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='10' port='0x19'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x1'/>
    </controller>
    <controller type='pci' index='11' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='11' port='0x1a'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x2'/>
    </controller>
    <controller type='pci' index='12' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='12' port='0x1b'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x3'/>
    </controller>
    <controller type='pci' index='13' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='13' port='0x1c'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x4'/>
    </controller>
    <controller type='pci' index='14' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='14' port='0x1d'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x5'/>
    </controller>
    <controller type='sata' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1f' function='0x2'/>
    </controller>
    <controller type='virtio-serial' index='0'>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
    </controller>
    <interface type='network'>
      <mac address='52:54:00:db:4f:39'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </interface>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
    </channel>
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='egl-headless'>
      <gl rendernode='/dev/dri/renderD128'/>
    </graphics>
    <sound model='ich9'>
      <audio id='1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1b' function='0x0'/>
    </sound>
    <audio id='1' type='pipewire'/>
    <video>
      <model type='none'/>
    </video>
    <watchdog model='itco' action='reset'/>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
    </memballoon>
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
      <address type='pci' domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
    </rng>
  </devices>
  <qemu:commandline>
    <qemu:arg value='-device'/>
    <qemu:arg value='virtio-vga-gl,id=video1,max_outputs=1,blob=true,hostmem=8G,venus=true'/>
    <qemu:arg value='-vga'/>
    <qemu:arg value='none'/>
    <qemu:arg value='-display'/>
    <qemu:arg value='sdl,gl=on,show-cursor=on'/>
    <qemu:env name='DISPLAY' value=':0'/>
    <qemu:env name='WAYLAND_DISPLAY' value='wayland-0'/>
    <qemu:env name='XDG_RUNTIME_DIR' value='/run/user/1000'/>
  </qemu:commandline>
</domain>

```
