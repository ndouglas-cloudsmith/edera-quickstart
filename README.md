# Edera Quickstart
In this lab, we will setup a single-node tier of Edera for evaluating hardened container runtime protection. <br/>
**[EderaON](https://on.edera.dev)** runs each container inside its own lightweight virtual machine, giving you hardware-level isolation that stops container escapes from reaching other workloads or the host.

Pre-flight checks
===============

The quickest way to see both the **architecture** and the **operating system** is by running:
```bash
uname -a
```
This will print your system information in a single line, including:
- Kernel name
- Network node hostname
- Kernel release date
- Operating system
- Machine architecture <br/>
(for example: ```x86_64``` or ```aarch64```).

### To see the Architecture only:
```bash
uname -m
```
*Alternatively, you can use* ```lscpu``` *to see full CPU details.*

### To see the Operating System details only:
```bash
cat /etc/os-release
```
If your system has ```systemd``` installed: <br/>
(common on most modern VMs/servers)
```bash
hostnamectl
```
This gives you a beautifully formatted layout showing the **OS**, **Kernel**, and **Architecture** all at once.

Install Docker
===============
I put together a simple bash install script for the **Docker runtime**:
```bash
chmod +x install_docker.sh
./install_docker.sh
```
Verify the install was successful & Docker is **Running**:
```bash
systemctl status docker
```
Now you are completely set up with the robust, production-ready version of Docker. <br/>
Let's start by creating a Docker container to prove there's no isolation from the host:

The ```uname -r``` command shows the Kernel ```release``` version you are running
```bash
docker run --rm docker.io/library/alpine:latest uname -r
```
The ```--rm``` flag automatically cleans up and removes the container after it exits. <br/>
Alternatively, you can shell into the container and play around with ```uname``` command:
```bash
docker run --rm -it docker.io/library/alpine:latest sh
``````
Running the same command on the host should match this output exactly.
```bash
uname -r
```
This simple exercise proves the **[Shared Kernel Problem](https://edera.dev/stories/the-shared-kernel-is-the-real-problem-in-container-security)** in containers and Kubernetes.

Get your license
===============

Check that you are using a valid Edera license key:
```bash
cat /var/lib/edera/protect/license.key
```

Authenticate Docker using the license file
```bash
docker login -u license -p "$(cat /var/lib/edera/protect/license.key)" images.edera.dev
```

Install Edera
===============

**Disposable infrastructure only.** <br/>
Edera modifies your bootloader and there is no automated uninstall. Only install on instances or VMs you can terminate and recreate.
```bash
EDERA_LICENSE_KEY="$(cat /var/lib/edera/protect/license.key)" /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/edera-dev/learn/main/getting-started/edera-on-installer/scripts/install.sh)" -- --verbose
```

Once the VM restarts, it will boot into the Edera hypervisor, and your **Ubuntu 22.04** OS will lift up as an Edera-managed guest.

Verify the install
===============
**NOTE: You cannot interact with the terminal during reboot**. <br/>
Wait for the red popup message: ```Connection lost, trying to reconnect...```
<br/><br/>
Run the ```uname``` command to verify that you are running the custom Edera kernel generated during the installation process: ```Edera/Xen``` kernel
```bash
uname -r | grep 'edera'
```
**Expected:** ```6.x.y-edera```
<br/><br/>
Verify the services are running:
```bash
ps auxww | grep protect
```
### systemd-detect-virt
This utility checks the **[CPUID](https://en.wikipedia.org/wiki/CPUID)** and system interfaces to see if the environment is virtualised:
```bash
systemd-detect-virt | grep -E 'kvm'
```
Edera zones currently run on a **[Xen](https://edera.dev/stories/why-edera-built-on-xen-a-secure-container-foundation)** Hypervisor.  Starting in summer 2026, the same zone-based isolation will also run on **[KVM](https://docs.edera.dev/technical-overview/architecture/kvm/)** (Kernel-based Virtual Machines), preserving identical security guarantees while meeting teams where their infrastructure already is.
###  Check dmesg for Hypervisor boot logs:
Inspect the ring buffer to see hypervisor handoff messages:
```bash
dmesg | grep -iE "hypervisor|xen|edera"
```
Look for boot lines indicating kernel is booting as guest <br/>
(like ```booting paravirtualised kernel on Xen``` or ```hvc0``` devices).

### Check the protect- systemd services
The installer enabled several Edera management daemons.
```bash
systemctl status protect-daemon protect-network
```
Confirm ```Xen``` is present
```bash
ls /proc/xen
```
**Expected:** ```capabilities  privcmd  xenbus```
<br/><br/>
Check the daemon is running:
```bash
systemctl is-active protect-daemon | grep --color=always -E "active|$"
```
**Expected:** ```active```
<br/><br/>
```activating``` is not the same as ```active```. <br/>
The daemon should become ```active``` within seconds. <br/>
If everything is **active**, you can proceed with the lab.
<br/><br/>
If it stays in ```activating```, it failed to start. <br/>
A missing or invalid license key is a common cause. <br/>
Check logs with ```sudo journalctl -u protect-daemon -n 50```.

Launch a zone
===============
Launching a zone typically takes less than a minute. <br/>
If it takes longer, check logs in **Terminal 2** with: <br/>
```sudo journalctl -u protect-daemon -n 50```
```bash
protect zone launch -n test-zone --wait
protect zone list
```

To get more info about a specific Zone in YAML output:
```bash
protect zone list --output yaml | grep --color=always -E "ZONE_VIRTUALIZATION_BACKEND_AUTOMATIC|$"
```

A zone in ```ready``` state is running and available. <br/>
If not, check the logs to see why the activation failed:
```bash
journalctl -u protect-daemon -n 20 | sed \
  -e 's/\bINFO\b/\x1b[32m&\x1b[0m/g' \
  -e 's/\bWARN\b/\x1b[31m&\x1b[0m/g'
```

Run a workload
===============
Launch an interactive shell inside the zone:
```bash
protect workload launch \
  --zone test-zone \
  --name alpine-shell \
  -t -a \
  docker.io/library/alpine:latest sh
```

Once inside, run ```uname -r``` to confirm you’re running in an isolated zone with its own kernel:
```bash
uname -r
```
The ```6.18.XX``` output from ```uname -r``` is the version of the **dedicated zone kernel** that Edera booted specifically for your ```test-zone```.
- Exiting the container and running ```uname -r``` again should show ```6.18.XX-edera``` - proving the container is not on the shared kernel
- In a traditional container, running ```uname -r``` inside a container simply returns the host's kernel version, because all containers share a single host kernel.
- The output confirms that ```6.18.XX``` is not your host's shared kernel, but a completely isolated kernel running inside a **Type-1 hypervisor micro-VM**.
<br/><br/>
Type ```exit``` to leave the shell.

### Create long-lived workloads
```bash
protect workload launch --zone test-zone --name alpine-long -- docker.io/library/alpine:latest sleep 3600
protect workload launch --zone test-zone --name ubuntu-test docker.io/library/ubuntu:latest sleep 10
```
Check that the workload is running:
```bash
protect workload list
protect zone list
```

Understanding Zones
=======================

Create a throttled zone to **prevent CPU/Memory abuse** - ```cryptojacking``` etc..
```bash
protect zone launch --name throttled-zone \
  --min-cpus 1 --max-cpus 2 --target-cpus 1 \
  --min-memory 128 --max-memory 512 --target-memory 256 \
  --resource-adjustment-policy dynamic
```

```PVH``` (Paravirtualisation Light) is a virtualisation mode that combines the best parts of two older virtualisation methods:
- ```PV``` (Paravirtualisation) and
- ```HVM``` (Hardware Virtual Machine).
```
sudo protect zone launch --name dark-zone --network-backend none
sudo protect zone launch --name hardware-isolated-zone --virt-backend pvh
```

| Virtualisation Mode | Description | Performance & Security |
| ------------- | ------------- | ------------- |
| ```PV```  | Guest OS is modified to know it's virtualised. Makes direct software calls (**[hypercalls](https://wiki.xenproject.org/wiki/Hypercall)**) instead of hardware emulation.  | Fast, but has a larger security attack surface due to complex software interfaces. |
| ```HVM```  | Fully simulates a real physical machine using hardware extensions (like ```Intel VT-x``` or ```AMD-V```). <br/>Guest OS doesn't know it's virtualised.  | Very secure isolation, but slow boot times and performance overhead due to emulating old hardware (like ```BIOS``` or ```PCI buses```).  |
| ```PVH```  | Uses hardware virtualization features (like ```HVM```) to boot the OS securely, but skips all the slow, unneeded legacy hardware emulation by using lightweight PV interfaces for basic tasks. | Extremely fast boot times <br/> Minimal Overhead <br/> And a tiny security attack surface. |

Why use ```--virt-backend pvh``` for a hardware-isolated-zone?
=======================
When launching a secure, hardware-isolated zone, security and speed are paramount. By choosing ```pvh``` as your backend:
1. Because there's no emulated **[QEMU device model](https://www.qemu.org/2018/02/09/understanding-qemu-devices)**, there are **fewer vulnerabilities** for attackers to exploit to break out of zones.
2. Zone boots almost instantly & runs at **near-native hardware speeds** while remaining completely isolated from the ```host``` & ```zones```.

<br/>

List the ```Zones```:
```bash
protect zone list
```

Checking the logs, we can get a **WARNING** <br/>
```runtime failed to create zone: failed to create domain: xen platform error: xencall issue encountered: kernel error: EINVAL: Invalid argument```
```bash
journalctl -u protect-daemon -n 20 | sed \
  -e 's/\bINFO\b/\x1b[32m&\x1b[0m/g' \
  -e 's/\bWARN\b/\x1b[31m&\x1b[0m/g'
```

The ```hardware-isolated-zone``` should be in a ```failed``` state. <br/>
List the YAML output for the ```hardware-isolated-zone``` Zone to understand what caused the failure.
```bash
protect zone list --output yaml | grep -B 15 -A 20 "'hardware-isolated-zone'"
```

Manage the ```host``` of **Edera Protect**:
```bash
protect host --help
```

Get general info about the ```host```
```bash
protect host status
```

Display information about the ```host``` **CPU topology**
```bash
protect host cpu-topology
```

Display the **Hypervisor Console**  ```hv-console``` output
```bash
protect host hv-console
```

Enable Kernel Verbose Logging
=======================

You can optionally **[enable kernel verbose logging](https://docs.edera.dev/guides/standalone/no-kubernetes/#optional-enable-kernel-verbose-logging)** <br/>
To see the detailed logs from the zone’s kernel:
```bash
protect zone launch --name zone-test --kernel-verbose
protect zone logs zone-test
```

To launch with a specific kernel version:
```bash
protect zone launch \
  --name zone-test \
  --kernel-verbose \
  --kernel ghcr.io/edera-dev/zone-kernel:latest
```

Once your zone is up, you can launch workloads using standard container images:
```
protect workload launch \
  --zone zone-test \
  --name nginx-test \
  docker.io/library/nginx:alpine
```

Check which images are cached:
```
protect image list
```
Verify the workload is running:
```
protect workload list
```

Install Tetragon
=======================

Tetragon will be managed as a ```systemd``` service. <br/>
Tetragon supports ```amd64``` & ```arm64``` architectures.

1. First download the latest binary tarball, using ```curl``` for example to download the ```amd64``` release:
```bash
curl -LO https://github.com/cilium/tetragon/releases/download/v1.7.0/tetragon-v1.7.0-amd64.tar.gz
```

2. Extract the downloaded archive, and start the install script:
```bash
tar -xvf tetragon-v1.7.0-amd64.tar.gz
cd tetragon-v1.7.0-amd64/
sudo ./install.sh
cd ..
```
The final output should be: <br/>
```Tetragon installed successfully!``` <br/><br/>
3. Finally, you can check the Tetragon ```systemd``` service.
```bash
systemctl status tetragon
```
The output should be similar to: <br/>
```tetragon.service - Tetragon eBPF-based Security Observability and Runtime Enforcement```

Install tetra-cli
=======================

This install method retrieves the adapted archived for your environment, extracts it and installs it into ```/usr/local/bin```:
```bash
curl -L https://github.com/cilium/tetragon/releases/latest/download/tetra-linux-amd64.tar.gz | tar -xz
sudo mv tetra /usr/local/bin
```

In **Terminal 2**, check that ```tetra``` CLI is ```running```:
```bash
tetra status | grep -E running
```

**NOTE**: Look for the parent processes. Docker will spawn ```containerd-shim```, which in turn will hand off execution to Edera's runtime components (```styrolite``` or ```krata```).

🧪 Experiment 1: Trigger some KVM activity
===============
The ```tetra``` CLI has built-in commands to upload policies directly to the local Tetragon server.
<br/><br/>
Add a the KVM monitoring ```policy```:
```bash
cd /app/
tetra tracingpolicy add kvm-trace.yaml
```
List currently ```active``` policies
```bash
tetra tracingpolicy list
```
The ```tetra getevents``` command communicates directly with the local Tetragon gRPC socket.
Tetragon's CLI can format it directly, though it defaults to showing the process context.
To see the specific **[IOCTL](https://en.wikipedia.org/wiki/Ioctl)** arguments, using the compact printer with a ```grep``` is highly effective:
```bash
tetra getevents -o compact
```
Delete a policy when you're done:
```bash
tetra tracingpolicy delete trace-edera-kvm
```

🧪 Experiment 2: Edera Filesystem (FIM) activity
=======================

Add a the FIM monitoring ```policy```:
```bash
tetra tracingpolicy add edera-fs.yaml
tetra tracingpolicy list
```

Since the policy is monitoring ```/dev/kvm``` interactions, you won't see anything until Edera actually tries to spin up or talk to a micro-VM. <br/>
In **Terminal 1** run:
```bash
docker run --rm -it alpine echo "Hello from Edera"
```
The ```sys_ioctl``` call handles all device I/O control across the system. <br/><br/>
In **Terminal 2**, If the policy feels a bit too specific or we want to see everything Edera touches at the **syscall level** without writing massive YAML files, we could alternatively stream process actions and filter by the Edera runtime name directly:
```bash
tetra getevents -o compact --process "styrolite|krata|containerd-shim"
```

🧪 Experiment 3: Untrusted Code Execution
=======================
I haven't actually built out these scenarios. So you can move on,
```bash
docker image ls
```

```ghcr.io/edera-dev/edera-check:stable``` has the ```U``` flag, meaning it is locked by an existing container.
If you try to remove it using ```docker rmi``` or a system prune, the engine will block the deletion to prevent breaking that container.



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
Finally, ```virsh vol-delete``` permanently deletes the actual virtual hard drive file (```ubuntu2404-test.qcow2```) from the ```default``` storage pool, freeing up the disk space on the physical host.
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
