# Edera Quickstart
In this lab, we will provide a free, single-node tier of Edera for evaluating hardened container runtime protection on your own infrastructure. <br/>
<br/>
**EderaON** runs each container inside its own lightweight virtual machine, giving you hardware-level isolation that stops container escapes from reaching other workloads or the host.

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
### Set up Docker's Official Repository
```bash
apt-get update
apt-get install -y ca-certificates curl gnupg
```

Next, create the directory for Docker's archive keyring and download their official GPG key:
```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
```

Now, add the Docker repository to your ```apt``` sources:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine
```bash
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify the Installation

```bash
systemctl status docker
```
Now you are completely set up with the robust, production-ready version of Docker. You can confidently re-run ```docker login```:

Get your license
===============
Run this command first to store the license key in your current terminal session:
```bash
export EDERA_LICENSE_KEY="LICENSE-KEY-123456789-V3"
```

Authenticate:
```bash
docker login -u license -p $EDERA_LICENSE_KEY images.edera.dev
```

Install Edera
===============
The installer requires the ```nft``` binary to configure networking:
```bash
apt-get update && apt-get install -y nftables
```

**Disposable infrastructure only.** Edera modifies your bootloader and there is no automated uninstall. Only install on instances or VMs you can terminate and recreate.
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/edera-dev/learn/main/getting-started/edera-on-installer/scripts/install.sh)" -- --verbose
```

**Note:** Because ```$EDERA_LICENSE_KEY``` is now globally available in your environment for this session, you no longer need to prefix the installation command with ```EDERA_LICENSE_KEY=....``` .
The script will automatically pick it up.

### Not seeing any changes ?
Since everything is already configured and patched on your disk, you just need to manually trigger the ```reboot``` from your host shell:

```
reboot
```
Once the VM restarts, it will boot into the Edera hypervisor, and your Ubuntu 22.04 OS will lift up as an Edera-managed guest.

Verify the install
===============
**NOTE: You might not be able to interact with the terminal for a short while during reboot**. Wait for the red popup message: ```Connection lost, trying to reconnect...```
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
```
sudo protect zone launch -n test-zone --wait
sudo protect zone list
```
A zone in ```ready``` state is running and available.

Run a workload
===============
Launch an interactive shell inside the zone:
```
sudo protect workload launch \
  --zone test-zone \
  --name alpine-shell \
  -t -a \
  docker.io/library/alpine:latest sh
```

Once inside, run ```uname -r``` to confirm you’re running in an isolated zone with its own kernel:
```
uname -r
```
**Expected:** ```6.18.18```
<br/><br/>
Type ```exit``` to leave the shell.

🧪 Experiment 1: Container Escape
=======================

Sample command to build a Docker image using a ```Dockerfile```:

```bash
cd /app
docker build -t my-service .
```

🧪 Experiment 2: Untrusted Code Execution
=======================

I haven't actually built out these scenarios. So you can move on,

```bash
docker image ls
```

```ghcr.io/edera-dev/edera-check:stable``` has the ```U``` flag, meaning it is locked by an existing container. If you try to remove it using ```docker rmi``` or a system prune, the engine will block the deletion to prevent breaking that container.

🏁 Finish
=========

To complete the challenge, press **Check**.
