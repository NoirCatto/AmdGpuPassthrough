# KVM with GPU-passthrough on a Full-AMD system (Arch linux/Win11)

##  Table of Contents

* [Introduction & considerations](#Introduction%20&%20considerations)
	* [System Setup](#System%20Setup)
		* [Hardware](#Hardware)
		* [Software](#Software)
* [Step by step](#Step%20by%20step)
	* [1. BIOS settings](#Step%201%20BIOS%20settings)
	* [2. IOMMU groups](#Step%202%20Determine%20IOMMU%20groups)
	* [3. Releasing the dGPU](#Step%203%20Releasing%20the%20dGPU)
		* [3.1 Vfio bind/unbind script preparations](#Step%203.1%20Vfio%20bind/unbind%20script%20preparations)
		* [3.2 Bind script](#Step%203.2%20Bind%20script)
		* [3.3 Unbind script](#Step%203.3%20Unbind%20script)
	* [4. VM setup](#Step%204.%20VM%20setup)
		* [4.1 GUI setup](#Step%204.1%20GUI%20setup)
		* [4.2 XML edits](#Step%204.2%20XML%20edits)
			* [4.3 Fixes](#4.3%20Fixes)
			* [4.4 Mouse & Keyboard](#4.4%20Mouse%20&%20Keyboard)
			* [4.5 Audio passthrough](#4.5%20Audio%20passthrough)
	* [5. Win 11 install + TPM check skip](#Step%205%20Win%2011%20install%20+%20TPM%20check%20skip)
	* [6. Network Bridge (optional)](#Step%206%20Network%20Bridge%20(optional))
* [Troubleshooting](#Troubleshooting)
	* [1. QEMU Error/1:1 IOMMU mapping](#QEMU%20Error/1%20to%201%20IOMMU%20mapping)

## Introduction & considerations

May this page serve as my attempt at documenting my process of setting up a KVM with a GPU passthrough on a full-AMD system.

<img src="./img/dualmon.jpg" width="500"  alt="Photo of two monitors in a vertical setup. Monitor on the top displays a desktop with Arch linux logo. Monitor on the bottom displays Windows 11 properties menu."/>

*Please note: This is not a tutorial - even if I may style it as one.
May the information here help you, and feel free to open up an issue if you think any of it is wrong or outdated. But don't complain if you end up breaking your setup because you religiously followed the steps with no second thought.*

## System Setup

I'm running two-monitors:

- Top one plugged into the motherboard (USB4 --> DisplayPort)
- Bottom one plugged into the dGPU (DisplayPort -> DisplayPort)

During daily use I have a two-monitor Linux setup. When I need to use Windows, I use a script to detach the dGPU along with the bottom monitor, which is then used by the Windows VM.

Additionally, I passthrough an NVMe drive to the VM for optimal performance + the ability to dual-boot if I need to (anti-cheats aren't fond of VMs).

### Hardware
- CPU:
	- AMD Ryzen 9 9950X3D
- GPU:
	- Radeon RX 9080XT (Sapphire Nitro+)
- Motherboard:
	- Gigabyte X870E AORUS PRO X3D ICE
- Memory:
	- Corsair Vengeance DDR5 6000MHz CL30 (2x32GB)
- Storage
	- Kingston KC3000 4TB - M.2 NVMe (host)
	- Samsung 980PRO 2TB - M.2 NVMe (guest) 

### Software
- Host:
	- Arch Linux w/ GNOME desktop enviroment
	- ly (display manager, replacing gdm)
	- virt-manager
	- qemu-full
	- libvirt
- Guest:
	- Windows 11 Pro (25H2)
	- virtio-win

***

## Step by step
### Step 1: BIOS settings

We need to make sure virtualization and `IOMMU` is turned on.

On Gigabyte X870E motherboards, look for `SVM Mode`. Otherwise, for AMD processors look for `AMD-Vi` and for Intel processors `VT-d`.
Afterwards, enable `IOMMU`.
If you're planning on sharing resources accross multiple VMs, also enable `SR-IOV`.

Boot into the host and confirm IOMMU is enabled: `sudo dmesg | grep IOMMU`
And CPU virtualization too:
- For AMD: `sudo dmesg | grep AMD-Vi`
- For Intel: `sudo dmesg | grep VT-d`

It's also a good idea to force enable the Integrated graphics, to make sure we always have host display.

### Step 2: Determine IOMMU groups

First we need to check which IOMMU groups are our devices in.
Devices residing within the same IOMMU group cannot be separated, so we need to pass them together.

We can check the IOMMU groups with a simple bash script[^iommu]:

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

<br>

My output looks like this:
```bash
...
IOMMU Group 15:
	03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 48 [Radeon RX 9070/9070 XT/9070 GRE] [1002:7550] (rev c0)
IOMMU Group 16:
	03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 48 HDMI/DP Audio Controller [1002:ab40]
...
IOMMU Group 19:
	06:00.0 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] 600 Series Chipset PCIe Switch Downstream Port [1022:43f5] (rev 01)
	07:00.0 Non-Volatile memory controller [0108]: Samsung Electronics Co Ltd NVMe SSD Controller PM9A1/PM9A3/980PRO [144d:a80a]
...
```

I am lucky with my dGPU residing in it's own IOMMU group.
The NVMe resides in one group with it's PCI slots' bridge, so I will be passing them together.

If your GPU shares an IOMMU group with an unwanted device, you might be able to get around it using an ACS Override Patch. It has it's own risks involved, but since I've had no need for it, I will not go into details here.

### Step 3: Releasing the dGPU

*This part took me the longest to figure out. So first some explanation:*
We need to unbind the dGPU from the `amdgpu` driver and bind it to `vfio-pci` instead.[^vfiopass]

To do this, first we need to unload the `amdgpu` kernel module - which, becase we're on a full-AMD setup, also kills our iGPU graphics. We'll need to restart this module afterwards, to get host display working again.

In the process, we also have to kill and restart our display manager to free up the `amdgpu` module so we can unload it.

Let's make a script to perform all of these actions in order.

#### Step 3.1: Vfio bind/unbind script preparations

Create a directory where you'll throw your scripts and configs. I made one in user's home folder: `~\Virtualization`

Now create a file `vfio_pci.conf`

```bash
#!/bin/bash

gpu="0000:03:00.0" #dgpu
aud="0000:03:00.1" #dgpu audio
ssd="0000:07:00.0" #nvme
bri="0000:06:00.0" #pcieport
gpu_vd="$(cat /sys/bus/pci/devices/$gpu/vendor) $(cat /sys/bus/pci/devices/$gpu/device)"
aud_vd="$(cat /sys/bus/pci/devices/$aud/vendor) $(cat /sys/bus/pci/devices/$aud/device)"
ssd_vd="$(cat /sys/bus/pci/devices/$ssd/vendor) $(cat /sys/bus/pci/devices/$ssd/device)"
bri_vd="$(cat /sys/bus/pci/devices/$bri/vendor) $(cat /sys/bus/pci/devices/$bri/device)"


function bind_vfio {
  echo "$gpu" > "/sys/bus/pci/devices/$gpu/driver/unbind"
  echo "$aud" > "/sys/bus/pci/devices/$aud/driver/unbind"
  echo "$gpu_vd" > /sys/bus/pci/drivers/vfio-pci/new_id
  echo "$aud_vd" > /sys/bus/pci/drivers/vfio-pci/new_id
  echo "$ssd" > "/sys/bus/pci/devices/$ssd/driver/unbind"
  echo "$bri" > "/sys/bus/pci/devices/$bri/driver/unbind"
  echo "$ssd_vd" > /sys/bus/pci/drivers/vfio-pci/new_id
  echo "$bri_vd" > /sys/bus/pci/drivers/vfio-pci/new_id
}

function unbind_vfio {
  echo "$gpu_vd" > "/sys/bus/pci/drivers/vfio-pci/remove_id"
  echo "$aud_vd" > "/sys/bus/pci/drivers/vfio-pci/remove_id"
  echo 1 > "/sys/bus/pci/devices/$gpu/remove"
  echo 1 > "/sys/bus/pci/devices/$aud/remove"
  echo "$ssd_vd" > "/sys/bus/pci/drivers/vfio-pci/remove_id"
  echo "$bri_vd" > "/sys/bus/pci/drivers/vfio-pci/remove_id"
  echo 1 > "/sys/bus/pci/devices/$ssd/remove"
  echo 1 > "/sys/bus/pci/devices/$bri/remove"
  echo 1 > "/sys/bus/pci/rescan"
}
```

Replace the UUIDs with the ones you got in [Step 2](#Step%202%20Determine%20IOMMU%20groups) (but keep the extra zeroes)

#### Step 3.2: Bind script

In the same directory, create a file `bind_vfio.sh`
We'll run it whenever we want to release the dGPU.

```bash
#!/bin/bash

## Log everything to bind.log, for debugging
set -x
exec 3>&1 4>&2
trap 'exec 2>&4 1>&3' 0 1 2 3
exec 1>bind.log 2>&1
## Ignore hang signals caused by killing display manager
trap '' HUP

## Load the config file. Replace with direct path if it fails.
source "vfio_pci.conf"

## Load vfio drivers
modprobe vfio
modprobe vfio_pci
modprobe vfio_iommu_type1

## Stop display manager to free up amdgpu driver
systemctl stop ly.service

## Unload radeon drivers (this will turn off ALL displays)
modprobe -r amdgpu
modprobe -r snd_hda_intel

## Unbind dGPU from amdgpu driver, and bind to vfio instead
bind_vfio

## Load radeon drivers (iGPU display should turn back on)
## If you have black screen, put 'video=efifb:off' in GRUB_CMDLINE_LINUX_DEFAULT
modprobe  amdgpu
modprobe  snd_hda_intel

## Start display manager
systemctl start ly.service
```

I replaced Gnome's `GDM` display manager with `ly`, so you may need to tweak it a little.

You can test the script by running it with `sudo ./bind_vfio.sh`.
If everything went correctly, your displays should turn off, and after a few seconds, the display connected to iGPU should turn back on and you'll be greeted with a login screen.

You can confirm the drivers got attached properly with `lspci -k | grep -A 7 VGA `
Look for `Kernel driver in use: vfio_pci`

If your display came on but you're greeted with a black screen:
- Open `/etc/default/grub`
- Put `video=efifb:off` in `GRUB_CMDLINE_LINUX_DEFAULT`
- Reload grub config with `sudo grub-mkconfig -o /boot/grub/grub.cfg`

You may also try switching to TTY with Ctrl+F3/Ctrl+F2
*Note: From my experience, switching to TTY doesn't work properly with GDM*

If none of your displays came on, look for `bind.log` in the directory you ran the script from and look for any errors or freezes. (Check if all commands from the script above executed)[^logging]

#### Step 3.3: Unbind script

In theory, you can just reboot and everything will go back to normal.

But here's a proper script for unbinding vfio if you prefer to use it instead:
Create `unbind_vfio.sh` in the same directory

```bash
#!/bin/bash

## Log everything to unbind.log, for debugging
set -x
exec 3>&1 4>&2
trap 'exec 2>&4 1>&3' 0 1 2 3
exec 1>unbind.log 2>&1
## Ignore hang signals caused by killing display manager
trap '' HUP

## Load the config file
source "vfio_pci.conf"

## Unload vfio drivers
modprobe -r vfio
modprobe -r vfio_pci
modprobe -r vfio_iommu_type1

## Unbind dGPU from vfio driver, and bind to default driver instead
unbind_vfio

## Restart the display manager to re-attach the second monitor
systemctl restart ly.service
```

After running `sudo ./unbind_vfio.sh` your second display should come awake and you'll be greeted with a login screen.

### Step 4: VM setup

We'll be using `virt-manager` under QEMU.
Download necessary packages first:

- `yay -S libvirt qemu-full virt-manager`

As well as virtio-win .iso for the guest[^virtio-win]:
- https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md

We can now start virt-manager and create our desired VM
- *Note: Before we begin, go to Edit --> Preferences --> Turn on* ***'Enable XML editing'***

#### Step 4.1: GUI setup
Create a new virtual machine

<img src="./img/1.png" width="500" />

Select "Local install media"

<img src="./img/2.png" width="500" />

Select your installation .iso

<img src="./img/3.png" width="500" />

Configure RAM and CPU settings.

<img src="./img/4.png" width="500" />

For my setup I am passing through an NVMe drive, so I uncheck "Enable storage for this virtual machine"

<img src="./img/5.png" width="500" />

Select "Customise configuration before install"

<img src="./img/6.png" width="500" />

Under "CPUs" select the `host-passthrough` model

<img src="./img/7.png" width="1000" />

Remove "SPLICE" and Video displays, as well as any other devices you don't need.

<img src="./img/8.png" width="1000" />

Under NIC select `virtio` device model for better network performance

<img src="./img/9.png" width="1000" />

Next click "Add hardware" --> PCI Host Device, and select your dGPU

<img src="./img/10.png" width="600" />

Repeat for every PCI device you want to pass-through.
*Note: The NVMe PCI bridge may not show here. In this case just skip it.*

<img src="./img/11.png" width="1000" />

Add the virtio-win.iso
"Add Hardware" --> Storage --> Select or create custom storage

<img src="./img/13.png" width="600" />

You can also choose to pass-through any USB devices.
*Note: These USB devices will be unavailable to the host for as long as the VM is powered on. For a switchable host/guest keyboard look [section 4.4](#4.4%20Mouse%20&%20Keyboard).*

<img src="./img/12.png" width="600" />

You might be able to run the VM with just these steps, however I recommend looking through the next section first.


#### Step 4.2: XML edits
###### 4.3 Fixes

Click on "Overview" --> XML

Radeon 9070 XT has an unpleasant bug with the Windows AMD drivers when ran in pass-through, which results in this nasty image:[^warbled]

<img src="./img/glitch.jpg" width="500" />

Thankfully, we can fix it.[^warbled-fix]

Let's have the hypervisor hide it's existence.
Inside the `hyperv` section, add a `vendor_id` tag with `state="on"` and `value` with any string up to 12 characters long

```xml
<features>
	...
	<hyperv mode="custom">
	  ...
	  <vendor_id state="on" value="whatever"/>
	  ...
	</hyperv>
	...
</features>
```

Additionally, instruct kvm to hide it's existence by adding this snippet right underneath the `hyperv` section.

```xml
<features>
	<hyperv mode="custom">
	...
	</hyperv>
	<kvm>
	  <hidden state="on"/>
	</kvm>
</features>
```


###### 4.4 Mouse & Keyboard

We can use the same Mouse and Keyboard for both host and guest machines and switch between them with a keybind, thanks to evdev.

How to set it up:
- Run `ls /dev/input/by-id`

Look for `event-kbd` and `event-mouse`:

```
...
usb-Corsair_Corsair_Gaming_K66_Keyboard_1100A031AECE00A5576C95AFF5001945-event-kbd
...
usb-ROCCAT_ROCCAT_Kova_Aimo-event-mouse
...
```

In the XML, in the `devices` section, add two `input` tags of type `evdev`
Set `source` to `dev="/dev/input/by-id/<id_of_your_device>`

```xml
<devices>
	...
	<input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
	<input type="evdev">
	  <source dev="/dev/input/by-id/usb-ROCCAT_ROCCAT_Kova_Aimo-event-mouse"/>
	</input>
	<input type="evdev">
	  <source dev="/dev/input/by-id/usb-Corsair_Corsair_Gaming_K66_Keyboard_1100A031AECE00A5576C95AFF5001945-event-kbd" grab="all" repeat="on"/>
	</input>
	...
</devices>
```

Now, when we boot up the VM, we can switch inputs between host and guest by hitting both Left Ctrl and Right Ctrl at the same time!

There *should* be a way to rebind this, but I haven't found it yet.

~

We can improve the input responsiveness further, by switching input type from PS/2 to virtio.[^evdev][^singlegpupass]

Add above the `ps2` input section:
```xml
<devices>
	...
    <input type='mouse' bus='virtio'/>
    <input type='keyboard' bus='virtio'/>
	<input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
	...
</devices>
```

###### 4.5 Audio passthrough

For audio passthrough I'm using *PipeWire*, though [alternative solutions](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_audio_from_virtual_machine_to_host_via_PipeWire_directly) exist.[^audiopass]

Edit `/etc/libvirt/qemu.conf`
- Find and uncomment `user = ""`
- Type your username in brackets, eg. `user = "master"`
- Reload libvirt `sudo systemctl restart libvirtd.service`

Find your user ID with `id -u <your_username>`
```bash
id -u master
1000
```

Add in `devices`, and change `1000` to your user ID
```xml
<devices>
	...
	<audio id="1" type="pipewire" runtimeDir="/run/user/1000"/>
	...
</devices>
```

*Note: There will always be considerable audio latency with this approach. While it is convenient for desktop usage, I would not recommend it for gaming.*

### Step 5: Win 11 install + TPM check skip

Windows installation should be no different from installing on an actual PC.
However, the installation media may scream at you because of Windows':
- TPM requirement check
- RAM requirement check
- Secure Boot check

It may also ask you to connect to the internet to create an account. Here's how to bypass all of those!

1. Boot into installation media
2. Press Shift + F10 to bring up CMD
3. Type in regedit and hit Enter
4. Navigate to `HKEY_LOCAL_MACHINE\SYSTEM\Setup`
5. Create new registry key and name it `LabConfig`
6. Within `LabConfig` create three DWORDs, `BypassTPMCheck`, `BypassRAMCheck` and `BypassSecureBootCheck`
7. Set all of their values to 1

Now you should be able to progress further with the installation.
During version setup, I recommend picking `Win 11 Pro`, as the network bypass trick may not work on lower Windows versions.

1. Progress until the user creation window. Installer will prompt you to use an online account.
2. Press Shift + F10
3. Type in `OOBE\BYPASSNRO`

An option should appear to create a local account.
Tested on `Win11_25H2_EnglishInternational_x64`

### Step 6: Network Bridge (optional)

By default, guest VM will be in a different subnet than host PC.
This is problematic if we need to access LAN devices (network drives, other PCs, VR headsets, etc.)

We can move guest onto the same subnet by creating a network bridge[^netbridge]

For this, I'll be using a NetworkManager GUI app:
- `yay -S nm-connection-editor`

Launch the app and select your ethernet connection

<img src="./img/n1.png" width="500" />

Click the gear icon to go into settings.
Go to IPv4 Settings and under `Method` select `Disabled`
Repeat for IPv6.
*Note: This will temporary kill your internet connection. If you have anything other than DHCP, make sure to write down your connection settings.*

<img src="./img/n2.png" width="500" />

Click Save.
Next, click on the Plus (+) icon to create a new connection.
Select `Bridge` and click "Create"

<img src="./img/n3.png" width="500" />

Name the interface, then under `Bridged connections` click "Add"

<img src="./img/n4.png" width="500" />

Select `Ethernet` and click "Create"

<img src="./img/n5.png" width="500" />

Under `device` select your Ethernet interface
Under `Link negotiation` select `Manual`

<img src="./img/n6.png" width="500" />

Go to IPv4 Settings. Under `Method` select `Automatic (DHCP)`
(or enter settings you've previously had on the Ethernet interface)

<img src="./img/n7.png" width="500" />

Click "Save".
In a moment, your internet connection should resume.

~

Now open virt-manager
Under your VM's settings, go to `NIC` and change `Network source` to `Bridge device`.
In the `Device name` field, type in the name of your previously created interface.

<img src="./img/n8.png" width="1000" />

The VM should now be in the same subnet as your host device

## Troubleshooting

### QEMU Error/1 to 1 IOMMU mapping
**Symptoms:**
- When attempting to start the VM, QEMU fails with error: 
<br><br> `(...) 0000:03:00.0 failed to setup container for group 15: Failed to set group` <br> `container: Invalid argument`
<br><br>The ID and group match up with my GPU's PCI ID and IOOMU group.
<br><br>
- Running `sudo dmesg | grep IOMMU` I found in the output:
<br><br>`vfio-pci 0000:03:00.0: Firmware has requested this device have a 1:1 IOMMU mapping, rejecting configuring the device without a 1:1 mapping. Contact your platform vendor.`

This has begun happening after I updated BIOS on my motherboard *(Gigabyte X870E AORUS PRO X3D ICE)*

**Solution:**<br><br>
Boot into BIOS. Several features need to be turned off/disabled[^1to1mapping]:
- Kernel DMA Protection
- Pre-boot DMA Protection
- SMEE
- TSME

Some of these may be buried deep under the advanced AMD settings. With these features turned off, everything began working correctly once more.


## Sources & Footnotes:
https://github.com/bryansteiner/gpu-passthrough-tutorial/tree/master

[^iommu]: [Checking IOMMU groups (wiki.archlinux.org)](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Ensuring_that_the_groups_are_valid)
[^vfiopass]:[PCI Passthrough via OMVF (wiki.archlinux.org)](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Isolating_the_GPU)
[^logging]: [Logging bash script actions (serverfault.com)](https://serverfault.com/questions/103501/how-can-i-fully-log-all-bash-scripts-actions)
[^virtio-win]: [Creating Windows virtual machines using virtIO drivers (docs.fedoraproject.org)](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/#virtio-win-direct-downloads)
[^warbled]: [Radeon 9070 XT pass-through problem (reddit.com)](https://www.reddit.com/r/VFIO/comments/1jefu1o/amd_radeon_rx_9070_gpu_passthrough_problem/)
[^warbled-fix]: [Radeon 9070 XT pass-through fixed (reddit.com)](https://www.reddit.com/r/VFIO/comments/1iblu6v/i_want_to_thank_you_guys/)
[^evdev]: [Evdev Passthrough Explained — Cheap, Seamless VM Input (passthroughpo.st)](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/)
[^singlegpupass]:[Complete Single GPU Passthrough (github.com)](https://github.com/QaidVoid/Complete-Single-GPU-Passthrough)
[^audiopass]: [Audio Passthrough (wiki.archlinux.org)](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_audio_from_virtual_machine_to_host_via_PipeWire_directly)
[^netbridge]: [Network bridge (wiki.archlinux.org)](https://wiki.archlinux.org/title/Network_bridge)
[^1to1mapping]: [RX 6800 XT GPU Passthrough Not Working Despite Successful vfio-pci Binding (reddit.com)](https://www.reddit.com/r/VFIO/comments/1p1sw5p/help_needed_rx_6800_xt_gpu_passthrough_not/)