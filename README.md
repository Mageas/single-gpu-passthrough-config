# VFIO Single GUP Passthrough Config

This is my personal config, for a tutorial, you can read [this](https://gitlab.com/Mageas/vfio-single-gup-passthrough).

### **Virt Hardware**

| Remove          |
|:----------------|
| `Display spice` |
| `Channel spice` |
| `Video QXL`     |
| `Sound ich*`    |
| `Serial 1`      |

| Add PCI |
|:--------------------|
| `0000:04:00:0` NVMe |
| `0000:0A:00:0` RTX 2070 Super |
| `0000:0A:00:1` RTX 2070 Super |
| `0000:0A:00:2` RTX 2070 Super |
| `0000:0A:00:3` RTX 2070 Super |
| `0000:05:00:0` USB Controller |

| Add USB |
|:--------------------|
| `009:002` Logitech G PRO |
| `009:003` PnP Audio Device |
| `009:004` Microdia Keyboard |

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
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=/run/user/1000/pulse/native"/>
  </qemu:commandline>
</domain>
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

### **Windows**

Enable Hyper-V
```ps1
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

Debloater
```ps1
iwr -useb https://git.io/debloat|iex
```

Install Chocolatey
```ps1
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

Install Apps
```ps1
choco install -y geforce-experience discord steam firefox
```

Optimizer
```
https://github.com/hellzerg/optimizer
```

Windows 10 Activator
```bat
@echo off
title Activate Windows 10 ALL versions for FREE!&cls&echo ============================================================================&echo #Project: Activating Microsoft software products for FREE without software&echo ============================================================================&echo.&echo #Supported products:&echo - Windows 10 Home&echo - Windows 10 Home N&echo - Windows 10 Home Single Language&echo - Windows 10 Home Country Specific&echo - Windows 10 Professional&echo - Windows 10 Professional N&echo - Windows 10 Education&echo - Windows 10 Education N&echo - Windows 10 Enterprise&echo - Windows 10 Enterprise N&echo - Windows 10 Enterprise LTSB&echo - Windows 10 Enterprise LTSB N&echo.&echo.&echo ============================================================================&echo Activating your Windows...&cscript //nologo slmgr.vbs /ckms >nul&cscript //nologo slmgr.vbs /upk >nul&cscript //nologo slmgr.vbs /cpky >nul&set i=1&wmic os | findstr /I "enterprise" >nul
if %errorlevel% EQU 0 (cscript //nologo slmgr.vbs /ipk NPPR9-FWDCX-D2C8J-H872K-2YT43 >nul&cscript //nologo slmgr.vbs /ipk DPH2V-TTNVB-4X9Q3-TJR4H-KHJW4 >nul&cscript //nologo slmgr.vbs /ipk WNMTR-4C88C-JK8YV-HQ7T2-76DF9 >nul&cscript //nologo slmgr.vbs /ipk 2F77B-TNFGY-69QQF-B8YKP-D69TJ >nul&cscript //nologo slmgr.vbs /ipk DCPHK-NFMTC-H88MJ-PFHPY-QJ4BJ >nul&cscript //nologo slmgr.vbs /ipk QFFDN-GRT3P-VKWWX-X7T3R-8B639 >nul&goto server) else wmic os | findstr /I "home" >nul
if %errorlevel% EQU 0 (cscript //nologo slmgr.vbs /ipk TX9XD-98N7V-6WMQ6-BX7FG-H8Q99 >nul&cscript //nologo slmgr.vbs /ipk 3KHY7-WNT83-DGQKR-F7HPR-844BM >nul&cscript //nologo slmgr.vbs /ipk 7HNRX-D7KGG-3K4RQ-4WPJ4-YTDFH >nul&cscript //nologo slmgr.vbs /ipk PVMJN-6DFY6-9CCP6-7BKTT-D3WVR >nul&goto server) else wmic os | findstr /I "education" >nul
if %errorlevel% EQU 0 (cscript //nologo slmgr.vbs /ipk NW6C2-QMPVW-D7KKK-3GKT6-VCFB2 >nul&cscript //nologo slmgr.vbs /ipk 2WH4N-8QGBV-H22JP-CT43Q-MDWWJ >nul&goto server) else wmic os | findstr /I "10 pro" >nul
if %errorlevel% EQU 0 (cscript //nologo slmgr.vbs /ipk W269N-WFGWX-YVC9B-4J6C9-T83GX >nul&cscript //nologo slmgr.vbs /ipk MH37W-N47XK-V7XM9-C7227-GCQG9 >nul&goto server) else (goto notsupported)
:server
if %i%==1 set KMS=kms7.MSGuides.com
if %i%==2 set KMS=kms8.MSGuides.com
if %i%==3 set KMS=kms9.MSGuides.com
if %i%==4 goto notsupported
cscript //nologo slmgr.vbs /skms %KMS%:1688 >nul&echo ============================================================================&echo.&echo.
cscript //nologo slmgr.vbs /ato | find /i "successfully" && (echo.&echo ============================================================================&echo.&echo #My official blog: MSGuides.com&echo.&echo #How it works: bit.ly/kms-server&echo.&echo #Please feel free to contact me at msguides.com@gmail.com if you have any questions or concerns.&echo.&echo #Please consider supporting this project: donate.msguides.com&echo #Your support is helping me keep my servers running everyday!&echo.&echo ============================================================================&choice /n /c YN /m "Would you like to visit my blog [Y,N]?" & if errorlevel 2 exit) || (echo The connection to my KMS server failed! Trying to connect to another one... & echo Please wait... & echo. & echo. & set /a i+=1 & goto server)
explorer "http://MSGuides.com"&goto halt
:notsupported
echo ============================================================================&echo.&echo Sorry! Your version is not supported.&echo.
:halt
pause >nul
```
```
https://github.com/vishalnagda1/windows-10-activator
```