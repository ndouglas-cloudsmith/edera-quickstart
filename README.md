# Edera Quickstart
In this lab, we will setup a single-node tier of Edera for evaluating hardened container runtime protection. <br/>
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
Now you are completely set up with the robust, production-ready version of Docker. <br/>
You can confidently re-run ```docker login```:

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

**Disposable infrastructure only.** <br/>
Edera modifies your bootloader and there is no automated uninstall. <br/>
Only install on instances or VMs you are able to terminate and recreate.
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
Once the VM restarts, it will boot into the Edera hypervisor. <br/> 
Your ```Ubuntu 22.04``` OS will startup as an ```Edera-managed guest```.

Verify the install
===============
**NOTE: You might not be able to interact with the terminal for a short while during reboot**. <br/>
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
<br/>
### Applying TracingPolicies natively
The ```tetra``` CLI has built-in commands to upload policies directly to the local Tetragon server.
<br/><br/>
Add a policy
```bash
tetra tracingpolicy add kvm-trace.yaml
```
List currently active policies
```bash
tetra tracingpolicy list
```
The ```tetra getevents``` command communicates directly with the local Tetragon gRPC socket. <br/>
Tetragon's CLI can format it directly, though it defaults to showing the process context. <br/>
To see the specific ```ioctl``` arguments, using the compact printer with a ```grep``` is highly effective:
```bash
tetra getevents -o compact
```
Run this command to listen specifically for your **custom tracing policy** events:
```bash
tetra getevents --output json | grep -E "trace-edera-kvm
```
Delete a policy when you're done:
```bash
tetra tracingpolicy delete trace-edera-kvm
```

🧪 Experiment 1: Trigger some KVM activity
=======================

Since the policy is monitoring ```/dev/kvm``` interactions, you won't see anything until Edera actually tries to spin up or talk to a micro-VM.
```bash
docker run --rm -it alpine echo "Hello from Edera"
```
The ```sys_ioctl``` call handles all device I/O control across the system. <br/>
If the policy feels a bit too specific or we want to see everything Edera touches at the **syscall level** without writing massive YAML files, we could alternatively stream process actions and filter by the Edera runtime name directly:
```
tetra getevents --process | grep -iE "styrolite|krata|containerd-shim"
```
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

```ghcr.io/edera-dev/edera-check:stable``` has the ```U``` flag, meaning it is locked by an existing container.
If you try to remove it using ```docker rmi``` or a system prune, the engine will block the deletion to prevent breaking that container.


🏁 Finish
=========

To complete the challenge, press **Check**.
