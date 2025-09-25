# Vast.ai GPU Setup for Proxmox LXC

> [!NOTE]
> Before running the main installation script, you must install some essential system packages and Python libraries. This will prevent common errors during > the setup process. You can install all prerequisites by running the commands below. Additional resource for vast: https://github.com/en4ble1337/vasttools?tab=readme-ov-file

```bash
# Update package list and install pip and build-essential
sudo apt update
sudo apt install python3-pip build-essential -y

# Install the required Python 'requests' library
pip3 install requests
```

## Installation Command

Install Vast.ai without drivers and docker flag (assumption its already installed - part of the template):

```bash
wget https://console.vast.ai/install -O install; sudo python3 install <api> --no-partitioning --no-driver --no-docker; history -d $((HISTCMD-1))
```
> [!NOTE]
> --no-partitioning possibly remove

## Container Configuration

Container requires additional config to pass nvml bug check:

```
lxc.init.cmd: /sbin/init systemd.unified_cgroup_hierarchy=0
```

## Complete LXC Configuration Example

Working configuration for Proxmox LXC with GPU passthrough:

```conf
arch: amd64
cores: 46
features: nesting=1,keyctl=1
hostname: gpuaas-4
memory: 249856
net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=BC:24:11:D9:A8:87,ip=dhcp,tag=30,type=veth
ostype: ubuntu
rootfs: hdd1:vm-6005-disk-0,size=2T
swap: 0
tags: vast

# GPU Device Access
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 235:* rwm
lxc.cgroup2.devices.allow: c 510:* rwm

# NVIDIA Device Mounts
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file

# Security and Device Access
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop: 

# Additional Device Access
lxc.cgroup2.devices.allow: b 7:* rwm
lxc.cgroup2.devices.allow: c 10:237 rwm
lxc.cgroup2.devices.allow: c 10:229 rwm

# Loop Device Mounts
lxc.mount.entry: /dev/fuse dev/fuse none bind,optional,create=file
lxc.mount.entry: /dev/loop0 dev/loop0 none bind,optional,create=file
lxc.mount.entry: /dev/loop1 dev/loop1 none bind,optional,create=file
lxc.mount.entry: /dev/loop2 dev/loop2 none bind,optional,create=file
lxc.mount.entry: /dev/loop3 dev/loop3 none bind,optional,create=file
lxc.mount.entry: /dev/loop4 dev/loop4 none bind,optional,create=file
lxc.mount.entry: /dev/loop5 dev/loop5 none bind,optional,create=file
lxc.mount.entry: /dev/loop6 dev/loop6 none bind,optional,create=file
lxc.mount.entry: /dev/loop7 dev/loop7 none bind,optional,create=file
lxc.mount.entry: /dev/loop-control dev/loop-control none bind,optional,create=file

# Systemd Configuration
lxc.init.cmd: /sbin/init systemd.unified_cgroup_hierarchy=0
```

## Test Results

Tested on Proxmox 9.x with RTX 5090. Sample installation output:

```
2025-09-14 17:47:55 (2.07 MB/s) - 'install' saved [54351/54351]
=> Begin vast host software install
=> Configure Vast.ai daemon
=> Update Vast.ai daemon
=> Install docker
Error during VM test/install: No module named 'requests'
Continuing...
=> Test docker
Using default tag: latest
latest: Pulling from library/ubuntu
Digest: sha256:9cbed754112939e914291337b5e554b07ad7c392491dba6daf25eef1332a22e8
Status: Image is up to date for ubuntu:latest
docker.io/library/ubuntu:latest
    Testing nvidia-docker for NVML error (30 seconds)
NVML test ok
found 1 nv gpus
=> Run Vast.ai daemon
=> Wait for daemon to start...
Daemon Running                
=> Done!
    Pulling default pytorch image
    Now go to https://vast.ai/list to set your
    prices and list your machine for rental.
finish
```

## Notes

- Replace `<api>` with your actual Vast.ai API key
- The configuration allows for GPU passthrough in LXC containers
- NVML test passes successfully with this setup
- Ready for listing on Vast.ai marketplace after installation
> [!NOTE]
> ## Storage overlay issue and GPU toolkit for Priv LXC container

Since you're already using overlay2 storage driver, the issue is likely with the current Docker daemon configuration still having storage-opts that reference pquota. Let's check and fix this:

1. **Check current Docker daemon configuration:**
   ```bash
   cat /etc/docker/daemon.json
   ```

2. **Look for any storage-opts entries** that might be causing the issue. Based on the vast.ai installation script, it likely contains something like:
   ```json
   {
     "storage-opts": ["overlay2.size=10G"]
   }
   ```

3. **Remove or modify the problematic storage-opts.** Edit `/etc/docker/daemon.json` to remove any storage-opts entries, or replace the entire file with:
   ```json
   {
     "storage-driver": "overlay2",
     "live-restore": true,
     "registry-mirrors": [
       "https://registry-1.docker.io",
       "https://docker1.vast.ai",
       "https://docker2.vast.ai",
       "https://docker3.vast.ai",
       "https://docker4.vast.ai",
       "https://docker5.vast.ai"
     ]
   }
   ```

4. **Restart Docker:**
   ```bash
   systemctl restart docker
   ```

5. **Verify the configuration:**
   ```bash
   docker info | grep -A 10 "Storage Driver"
   ```

The key issue is that Docker is trying to use storage options (likely size limits) that require XFS with pquota, but in your LXC environment, even though you're using overlay2, the underlying filesystem doesn't have the required project quota support.

After making these changes, run the vast.ai self-test again:
```bash
./vast.py self-test machine 43356
```

This should resolve the `--storage-opt is supported only for overlay over xfs with 'pquota' mount option` error.


The error you're encountering is because Docker inside your LXC container doesn't have the NVIDIA Container Runtime configured properly. The `--runtime=nvidia` flag requires the nvidia-container-runtime to be installed and configured within your LXC container.

## Solution: Install and Configure NVIDIA Container Toolkit

Here's how to fix this issue step by step:

### Step 1: Install NVIDIA Container Toolkit in the LXC Container

Inside your LXC container (ID 6005), run the following commands:

```bash
# Configure the production repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Update package lists
sudo apt-get update

# Install the NVIDIA Container Toolkit
sudo apt-get install -y nvidia-container-toolkit
```

### Step 2: Configure Docker to Use NVIDIA Runtime

Use the nvidia-ctk command to automatically configure Docker:[1][2][3]

```bash
# Configure the container runtime
sudo nvidia-ctk runtime configure --runtime=docker

# Restart Docker daemon
sudo systemctl restart docker
```

### Step 3: Configure NVIDIA Container Runtime for LXC Environment

Since you're running Docker inside a **privileged** LXC container, you need to modify the NVIDIA container runtime configuration:[4][5]

```bash
# Edit the NVIDIA container runtime configuration
sudo nano /etc/nvidia-container-runtime/config.toml
```

Find the line with `no-cgroups` and set it to `true`:
```toml
no-cgroups = true
```

This setting is **essential** for running Docker with GPU support inside LXC containers.[5][4]

### Step 4: Verify the Configuration

Check that the nvidia runtime is now available:

```bash
docker info | grep -i runtime
```

You should see output similar to:
```
Runtimes: io.containerd.runc.v2 runc nvidia
```

### Step 5: Test the Configuration

Run the vast.ai bandwidth test command:

```bash
docker run --rm --runtime=nvidia --env NVIDIA_VISIBLE_DEVICES=0 vastai/test:bandwidth-test-nvidia --csv --device=0 --memory=pinned --mode=range --start=1073741824 --end=1073741824 --increment=1
```
