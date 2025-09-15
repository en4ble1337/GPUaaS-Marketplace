# Vast.ai GPU Setup for Proxmox LXC

> [!NOTE]
> Before running the main installation script, you must install some essential system packages and Python libraries. This will prevent common errors during > the setup process. You can install all prerequisites by running the commands below:

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
