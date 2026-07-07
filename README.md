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
Now you are completely set up with the robust, production-ready version of Docker.

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
sudo systemctl is-active protect-daemon | grep --color=always -E "active|$"
```
**Expected:** ```active```
<br/><br/>
```activating``` is not the same as ```active```. <br/>
The daemon should become ```active``` within seconds.
<br/><br/>
If it stays in ```activating```, it failed to start. <br/>
A missing or invalid license key is a common cause. <br/>
Check logs with ```sudo journalctl -u protect-daemon -n 50```.
<br/><br/>
If everything is **active**, you can proceed with the lab.

Launch a zone
===============
Launching a zone typically takes less than a minute. <br/>
If it takes longer, check logs in **Terminal 2** with: <br/>
```sudo journalctl -u protect-daemon -n 50```
```bash
sudo protect zone launch -n test-zone --wait
sudo protect zone list
```

To get more info about a specific Zone in YAML output:
```bash
sudo protect zone list --output yaml
```

A zone in ```ready``` state is running and available. <br/>
If not, check the logs to see why the activation failed:
```bash
sudo journalctl -u protect-daemon -n 20
```

Run a workload
===============
Launch an interactive shell inside the zone:
```bash
sudo protect workload launch \
  --zone test-zone \
  --name alpine-shell \
  -t -a \
  docker.io/library/alpine:latest sh
```

Once inside, run ```uname -r``` to confirm you’re running in an isolated zone with its own kernel:
```bash
uname -r
```
**Expected:** ```6.18.18```
<br/><br/>
Type ```exit``` to leave the shell.

### Create long-lived workloads
```
sudo protect workload launch --zone test-zone --name alpine-long -- docker.io/library/alpine:latest sleep 3600
sudo protect workload launch --zone test-zone --name ubuntu-test docker.io/library/ubuntu:latest sleep 10
```
Check that the workload is running:
```
sudo protect workload list
```

Understanding Zones
=======================

Create a throttled zone to **prevent CPU/Memory abuse** - ```cryptojacking``` etc..
```
sudo protect zone launch --name throttled-zone \
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
2. The zone boots almost instantly and runs at **near-native hardware speeds** while remaining completely isolated from the ```host``` & other ```zones```.

<br/>

List ```Zones```:
```
sudo protect zone list --output yaml
```

Manage the ```host``` of **Edera Protect**:
```
sudo protect host --help
```

Get general info about the ```host```
```
sudo protect host status
```

Display information about the ```host``` **CPU topology**
```
sudo protect host cpu-topology
```

Display the **Hypervisor Console**  ```hv-console``` output
```
sudo protect host hv-console
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
sudo systemctl status tetragon
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
```
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

🏁 Finish
=========

To complete the challenge, press **Check**.
