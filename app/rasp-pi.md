KVM on Raspberry Pi
=========

Tailscale command:
```
ssh nigel@100.95.180.101
```

## virsh - CLI for managing KVM/QEMU VMs
We using this tool to manage ```domains``` via the ```libvirt``` management tool.
<br/><br/>
Lists the VMs on the host.
```
virsh list --all
```
Fetches basic metadata and status info for a specific VM:
```
virsh dominfo ubuntu2404-test
```
Lists all virtual storage volumes (disk images like ```.qcow2``` or ```.raw``` files)<br/>
The volumes existi inside a specific storage pool named ```default```.
```
virsh vol-list default
```
Displays the current DHCP network leases assigned to VMs.<br/>
Our VM is connected to the virtual network named ```default```.
```
virsh net-dhcp-leases default
```

Extract the raw XML config file of the VM<br/>
Filters it to look at the ```bootloader```, firmware and CD ROM configs.
```
virsh dumpxml ubuntu2404-test | grep loader -A5
virsh dumpxml ubuntu2404-test | grep -A10 -B2 cdrom
```

<img width="3056" height="1810" alt="terminal" src="https://github.com/user-attachments/assets/0b2509f5-f742-41cd-afc8-05b045d85a62" />

Start by listing the files located in the directory where ```libvirt``` (**KVM/QEMU**) often stores boot images, kernels, or ISO files. <br/>
The second command extracts the boot sector config and layout from the ISO image file.<br/> 
It essentially asks: *"How does a computer firmware (like **[UEFI](https://github.com/edera-dev/sprout)**) read the ISO to kick off the install process?"*
```
sudo ls -lh /var/lib/libvirt/boot/
sudo xorriso -indev /var/lib/libvirt/boot/ubuntu-24.04.4-live-server-arm64.iso -report_el_torito as_mkisofs
```
"**[El Torito](https://bugzilla.redhat.com/show_bug.cgi?id=1525458)**" is the standard extension spec that allows a CD-ROM/ISO to be bootable.<br/>
The ```as_mkisofs``` modifier tells ```xorriso``` to output the boot config formatted as the exact CLI params you would need if you wanted to recreate/modify bootable ISOs using the ```mkisofs``` tool.
<br/><br/>
People usually run this command when they want to remaster a custom Ubuntu ISO (like creating an unattended "```Autoinstall```" image)<br/>
and need to know precise boot images, partition layouts, and emulation settings required to keep new ISO bootable for ```ARM64/UEFI```.
<br/><br/>
Install the VM based on ISO provided and associated config:
```
sudo virt-install \
  --name ubuntu2404-test \
  --memory 3072 \
  --vcpus 2 \
  --disk pool=default,size=20,format=qcow2 \
  --os-variant ubuntu24.04 \
  --cdrom /var/lib/libvirt/boot/ubuntu-24.04.4-live-server-arm64.iso \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial
```

Together, these commands are used to completely and permanently delete a VM and its associated data from a KVM/QEMU host.<br/>
First, ```virsh destroy``` forcefully halts the running VM (like pulling the power plug) because a running VM cannot be deleted.<br/>
Next, ```virsh undefine --nvram``` removes the VM’s XML config file from ```libvirt``` alongside its NVRAM file<br/>
(which stores UEFI boot settings), essentially wiping the VM from the system's registry.<br/><br/>
Finally, ```virsh vol-delete``` permanently deletes the actual virtual hard drive file (```ubuntu2404-test.qcow2```)<br/>
from the ```default``` storage pool, freeing up the disk space on the physical host.
```
virsh destroy ubuntu2404-test
virsh undefine ubuntu2404-test --nvram
virsh vol-delete ubuntu2404-test.qcow2 --pool default
```

Downloading a fresh ISO from Canonical registry:
```
cd ~/isos
wget -O ubuntu-24.04.4-live-server-arm64.iso \
https://cdimage.ubuntu.com/releases/24.04/release/ubuntu-24.04.4-live-server-arm64.iso
ls -lh ubuntu-24.04.4-live-server-arm64.iso
sudo ls -lh /var/lib/libvirt/boot/
```

Use ```virsh edit ubuntu2404-test``` to safely modify the underlying XML configuration file of that specific VM, allowing you to fine-tune hardware settings.. such as adjusting CPU topologies, pinning RAM, or adding virtual hardware components like a custom CD-ROM drive, while also ensuring that ```libvirt``` instantly validates the syntax before applying changes to prevent VMs becoming unbootable.
```
virsh edit ubuntu2404-test
```
