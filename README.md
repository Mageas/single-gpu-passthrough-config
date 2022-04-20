### **Garbage**

| Remove          |
|:----------------|
| `Display spice` |
| `Channel spice` |
| `Video QXL`     |
| `Sound ich*`    |
| `Serial 1`      |

### **Config Libvirt Hooks**

<table>
<tr>
<th>
/etc/libvirt/hooks/kvm.conf
</th>
</tr>

<tr>
<td>

```conf
# CONFIG
VM_MEMORY=13312

# VIRSH
VIRSH_GPU_VIDEO=pci_0000_09_00_0
VIRSH_GPU_AUDIO=pci_0000_09_00_1
VIRSH_USB=pci_0000_09_00_2
VIRSH_SERIAL_BUS=pci_0000_09_00_3
VIRSH_NVME_SSD=pci_0000_04_00_0
```

</td>
</tr>
</table>

### **Start/Stop Libvirt Hooks**

<details>
  <summary><b>Create Start Script</b></summary>

  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/prepare/begin/start.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash
# Helpful to read output when debugging
set -x

# Load variables
source "/etc/libvirt/hooks/kvm.conf"

# Stop display manager
systemctl stop lightdm.service

# Unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Unbind EFI-Framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# Avoid a Race condition
sleep 5

# Unload all Nvidia drivers
modprobe -r nvidia_drm
modprobe -r nvidia_modeset
modprobe -r nvidia_uvm
modprobe -r nvidia

# Unbind the GPU from display driver
virsh nodedev-detach $VIRSH_GPU_VIDEO
virsh nodedev-detach $VIRSH_GPU_AUDIO
virsh nodedev-detach $VIRSH_USB
virsh nodedev-detach $VIRSH_SERIAL_BUS
virsh nodedev-detach $VIRSH_NVME_SSD

# Load VFIO Kernel Module  
modprobe vfio-pci 
```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create Stop Script</b></summary>

  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/release/end/stop.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash
set -x

# Load variables
source "/etc/libvirt/hooks/kvm.conf"

# Unload VFIO-PCI Kernel Driver
modprobe -r vfio-pci
modprobe -r vfio_iommu_type1
modprobe -r vfio

# Re-Bind GPU to Nvidia Driver
virsh nodedev-reattach $VIRSH_GPU_VIDEO
virsh nodedev-reattach $VIRSH_GPU_AUDIO
virsh nodedev-reattach $VIRSH_USB
virsh nodedev-reattach $VIRSH_SERIAL_BUS
virsh nodedev-reattach $VIRSH_NVME_SSD

# Rebind VT consoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind

# Bind EFI-Framebuffer
nvidia-xconfig --query-gpu-info > /dev/null 2>&1
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

# Load all Nvidia drivers
modprobe nvidia_drm
modprobe nvidia_modeset
modprobe drm_kms_helper
modprobe drm
modprobe nvidia_uvm
modprobe nvidia

# Restart Display Manager
systemctl start lightdm.service
```

  </td>
  </tr>
  </table>
</details>

### **Audio Passthrough**

<table>
<tr>
<th>
/etc/libvirt/qemu.conf
</th>
</tr>

<tr>
<td>

```conf
...
user = "root"
group = "wheel"
...
```

</td>
</tr>
</table>

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
```

</td>
</tr>
</table>

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <qemu:commandline>
    ...
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=/run/user/1000/pulse/native"/>
  </qemu:commandline>
</devices>
```

</td>
</tr>
</table>

### **Video card driver virtualisation detection**

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
<features>
  ...
  <hyperv>
    ...
    <vendor_id state='on' value='buttplug'/>
  </hyperv>
  <kvm>
    <hidden state='on'/>
  </kvm>
  <ioapic driver="kvm"/>
  ...
</features>
...
```

</td>
</tr>
</table>

### **vBIOS Patching**

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    ...
  </source>
  <rom file="/home/mageas/.local/kvm/patched-vbios.rom"/>
  ...
</hostdev>
...
```

</td>
</tr>
</table>

### **CPU Pinning**

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <vcpu placement="static">14</vcpu>
  <iothreads>1</iothreads>
  <cputune>
    <vcpupin vcpu="0" cpuset="1"/>
    <vcpupin vcpu="1" cpuset="9"/>
    <vcpupin vcpu="2" cpuset="2"/>
    <vcpupin vcpu="3" cpuset="10"/>
    <vcpupin vcpu="4" cpuset="3"/>
    <vcpupin vcpu="5" cpuset="11"/>
    <vcpupin vcpu="6" cpuset="4"/>
    <vcpupin vcpu="7" cpuset="12"/>
    <vcpupin vcpu="8" cpuset="5"/>
    <vcpupin vcpu="9" cpuset="13"/>
    <vcpupin vcpu="10" cpuset="6"/>
    <vcpupin vcpu="11" cpuset="14"/>
    <vcpupin vcpu="12" cpuset="7"/>
    <vcpupin vcpu="13" cpuset="15"/>
    <emulatorpin cpuset="0,8"/>
    <iothreadpin iothread="1" cpuset="0,8"/>
  </cputune>
  ...
</domain>
```

</td>
</tr>
</table>


<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="7" threads="2"/>
    <cache mode="passthrough"/>
    <feature policy="require" name="topoext"/>
  </cpu>
  ...
</domain>
```

</td>
</tr>
</table>

### **Hyper-V Enlightenments**

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <qemu:commandline>
    ...
    <qemu:arg value="-rtc"/>
    <qemu:arg value="base=localtime"/>
    <qemu:arg value="-cpu"/>
    <qemu:arg value="host,host-cache-info=on,kvm=off,l3-cache=on,kvm-hint-dedicated=on,migratable=no,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time,hv_vendor_id=buttplug,+invtsc,+topoext"/>
  </qemu:commandline>
</devices>
```

</td>
</tr>
</table>

### **Hugepages**

<details>
  <summary><b>Create Alloc Script</b></summary>

  <table>
  <tr>
  <th>
  /etc/libvirt/hooks/qemu.d/VM_NAME/prepare/begin/alloc_hugepages.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

## Load the config file
source "/etc/libvirt/hooks/kvm.conf"

## Calculate number of hugepages to allocate from memory (in MB)
HUGEPAGES="$(($VM_MEMORY/$(($(grep Hugepagesize /proc/meminfo | awk '{print $2}')/1024))))"

echo "Allocating hugepages..."
echo $HUGEPAGES > /proc/sys/vm/nr_hugepages
ALLOC_PAGES=$(cat /proc/sys/vm/nr_hugepages)

TRIES=0
while (( $ALLOC_PAGES != $HUGEPAGES && $TRIES < 1000 ))
do
  echo 1 > /proc/sys/vm/compact_memory            ## defrag ram
  echo $HUGEPAGES > /proc/sys/vm/nr_hugepages
  ALLOC_PAGES=$(cat /proc/sys/vm/nr_hugepages)
  echo "Succesfully allocated $ALLOC_PAGES / $HUGEPAGES"
  let TRIES+=1
done

if [ "$ALLOC_PAGES" -ne "$HUGEPAGES" ]
then
  echo "Not able to allocate all hugepages. Reverting..."
  echo 0 > /proc/sys/vm/nr_hugepages
  exit 1
fi
```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create Dealloc Script</b></summary>

  <table>
  <tr>
  <th>
  /etc/libvirt/hooks/qemu.d/VM_NAME/release/end/dealloc_hugepages.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

echo 0 > /proc/sys/vm/nr_hugepages
```

  </td>
  </tr>
  </table>
</details>

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <memory unit="KiB">13631488</memory>
  <currentMemory unit="KiB">13631488</currentMemory>
  <memoryBacking>
    <hugepages/>
  </memoryBacking>
  ...
</domain>

```

</td>
</tr>
</table>

### **CPU Governor**

<details>
  <summary><b>Create Performance Script</b></summary>

  <table>
  <tr>
  <th>
  /etc/libvirt/hooks/qemu.d/VM_NAME/prepare/begin/cpu_mode_performance.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

## Enable CPU governor performance mode
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
for file in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo "performance" > $file; done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create OnDemand Script</b></summary>

  <table>
  <tr>
  <th>
  /etc/libvirt/hooks/qemu.d/VM_NAME/release/end/cpu_mode_ondemand.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

## Enable CPU governor on-demand mode
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
for file in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo "ondemand" > $file; done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

  </td>
  </tr>
  </table>
</details>