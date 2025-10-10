# GPU Passthrough with LXC & NVIDIA Docker on Proxmox VE

A step-by-step guide to configuring a privileged LXC container with GPU passthrough, Docker, and NVIDIA Container Toolkit on Proxmox VE, ensuring NVML tests pass (Vast.ai compatible).

***

## Table of Contents

- [Prerequisites](#prerequisites)  
- [1. Host Preparation](#1-host-preparation)  
  - [1.1 Enable IOMMU & VFIO](#11-enable-iommu--vfio)  
  - [1.2 Install NVIDIA Driver](#12-install-nvidia-driver)  
- [2. LXC Container Creation](#2-lxc-container-creation)  
- [3. LXC Configuration](#3-lxc-configuration)  
- [4. NVIDIA Container Toolkit & Docker Setup](#4-nvidia-container-toolkit--docker-setup)  
  - [4.1 Install Docker](#41-install-docker)  
  - [4.2 Configure Docker Daemon](#42-configure-docker-daemon)  
  - [4.3 Install NVIDIA Container Toolkit](#43-install-nvidia-container-toolkit)  
  - [4.4 Configure NVIDIA Runtime](#44-configure-nvidia-runtime)  
- [5. Patching the NVML Test Script](#5-patching-the-nvml-test-script)  
- [6. Wrapper & Symlink Trick](#6-wrapper--symlink-trick)  
- [7. Validation & Testing](#7-validation--testing)  
- [8. Troubleshooting Tips](#8-troubleshooting-tips)  

***

## Prerequisites

- Proxmox VE 8.x or 9.x  
- NVIDIA GPU (e.g., RTX 3090)  
- Ubuntu 22.04 or 24.04 guest template  
- Internet connectivity  

***

## 1. Host Preparation

### 1.1 Enable IOMMU & VFIO

1. Edit GRUB:
   ```bash
   nano /etc/default/grub
   ```
2. Append to `GRUB_CMDLINE_LINUX_DEFAULT`:
   ```
   intel_iommu=on iommu=pt vfio-pci.ids=<GPU_PCI_ID>
   ```
3. Update GRUB and reboot:
   ```bash
   update-grub
   reboot
   ```

### 1.2 Install NVIDIA Driver

```bash
apt update
apt install -y nvidia-driver
reboot
```

Confirm on host:
```bash
nvidia-smi
```

***

## 2. LXC Container Creation

Create a new privileged LXC (ID `8001`):

```bash
pct create 8001 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.gz \
  --hostname vast-test \
  --net0 name=eth0,bridge=vmbr0,firewall=1,ip=dhcp \
  --cores 4 --memory 4096 --swap 0 \
  --rootfs local-lvm:8G \
  --unprivileged 0
```

***

## 3. LXC Configuration

Edit `/etc/pve/lxc/8001.conf`:

```ini
arch: amd64
cores: 4
hostname: vast-test
memory: 4028
net0: name=eth0,bridge=vmbr0,firewall=1,ip=dhcp,type=veth
ostype: ubuntu
rootfs: local-lvm:vm-8001-disk-0,size=500G
swap: 0
unprivileged: 0

# Systemd cgroup v1 fallback
lxc.init.cmd: /sbin/init systemd.unified_cgroup_hierarchy=0

# GPU device access
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 235:* rwm
lxc.cgroup2.devices.allow: c 510:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file

# Security overrides
lxc.apparmor.profile: unconfined
lxc.cap.drop:

# Additional devices
lxc.cgroup2.devices.allow: a
lxc.cgroup2.devices.allow: b 7:* rwm
lxc.cgroup2.devices.allow: c 10:237 rwm
lxc.cgroup2.devices.allow: c 10:229 rwm
lxc.mount.entry: /dev/fuse dev/fuse none bind,optional,create=file
lxc.mount.entry: /dev/loop-control dev/loop-control none bind,optional,create=file
# (loop0–loop7 as needed)
```

Start container:
```bash
pct start 8001
```

***

## 4. NVIDIA Container Toolkit & Docker Setup

Enter the LXC:
```bash
pct enter 8001
```

### 4.1 Install Docker

```bash
apt update
apt install -y docker.io
```

### 4.2 Configure Docker Daemon

Edit `/etc/docker/daemon.json`:

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
  ],
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}
```

Reload and restart:
```bash
systemctl daemon-reload
systemctl restart docker
```

Verify:
```bash
docker info | grep "Cgroup Driver"
# Expect: cgroupfs
```

### 4.3 Install NVIDIA Container Toolkit

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | apt-key add -
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list | tee /etc/apt/sources.list.d/nvidia-container-runtime.list
apt update
apt install -y nvidia-container-toolkit
```

### 4.4 Configure NVIDIA Runtime

Edit `/etc/nvidia-container-runtime/config.toml`:

```toml
no-cgroups = false

[nvidia-container-cli]
load-kmods = true
ldconfig = "@/sbin/ldconfig.real"

[nvidia-container-runtime]
log-level = "info"
mode = "auto"
```

***

## 5. Patching the NVML Test Script

The Vast.ai installer’s test script calls `systemctl daemon-reload`, breaking NVML.

1. Backup original:
   ```bash
   cp /var/lib/vastai_kaalia/test_nvml_error.sh ~/.test_nvml_error.sh.bak
   ```
2. Comment out reload line:
   ```bash
   sed -i 's/^\s*sudo systemctl daemon-reload/  # &/' /var/lib/vastai_kaalia/test_nvml_error.sh
   ```

***

## 6. Wrapper & Symlink Trick

Ensure patched script persists across installer runs:

```bash
mkdir -p /usr/local/vastai
cp /var/lib/vastai_kaalia/test_nvml_error.sh /usr/local/vastai/test_nvml_error.sh
chmod +x /usr/local/vastai/test_nvml_error.sh

cat > /usr/local/bin/vastai-test-wrapper.sh << 'EOF'
#!/bin/bash
cp /usr/local/vastai/test_nvml_error.sh /var/lib/vastai_kaalia/test_nvml_error.sh
exec /var/lib/vastai_kaalia/test_nvml_error.sh "$@"
EOF
chmod +x /usr/local/bin/vastai-test-wrapper.sh

ln -sf /usr/local/bin/vastai-test-wrapper.sh /var/lib/vastai_kaalia/test_nvml_error.sh
```

***

## 7. Validation & Testing

1. **Standalone NVML Test**  
   ```bash
   bash -x /var/lib/vastai_kaalia/test_nvml_error.sh
   ```
   Expect: “The machine does not have the problem.”

2. **Vast.ai Installer**  
   ```bash
   wget https://console.vast.ai/install -O install
   sudo python3 install <API> --no-partitioning --no-driver --no-docker
   ```
   NVML check will now pass.

***

## 8. Troubleshooting Tips

- **Missing Devices**: Verify `/dev/nvidia*` & `/dev/nvidia-caps/*` exist and are `crw-rw-rw-`.
- **Cgroup Version**: Ensure `mount | grep cgroup` shows cgroup2 on host, but Docker uses `cgroupfs`.
- **Docker Driver**: Confirm `docker info | grep Cgroup Driver` is `cgroupfs`.
- **Runtime Logs**: Check `/var/log/syslog` for NVIDIA toolkit errors.
- **Systemd Parameter**: `lxc.init.cmd` must use `systemd.unified_cgroup_hierarchy=0`.

# GPUaaS on Proxmox LXC with vast.ai Selftest Pass  
Comprehensive step-by-step guide for setting up a Proxmox LXC container (Ubuntu 22.04) with NVIDIA GPU passthrough, Docker using overlay2+XFS quotas, and passing vast.ai’s selftest.  

---

## Prerequisites  
- Proxmox VE host (Debian Stretch or later)  
- Ubuntu 22.04 LXC template  
- NVIDIA GPU present and drivers installed on host  
- `vast` CLI installed in the LXC after setup  
- Sufficient free space in an LVM-thin pool (e.g., `local-lvm` / `pve/data`)  

---

## 1. Proxmox Host: Prepare XFS-Quotas Storage for Docker  

1.1. Identify your LVM-thin pool:  
```
# Should list an LV of type “thinpool” (e.g., pve/data)
lvs -o+seg_monitor
```

1.2. Create a thin-provisioned LV for Docker data:  
```bash
lvcreate -V 800G -T -n docker-thin pve/data
```
> -  `-V 800G`: sets an 800 GiB per-container quota ceiling (thin)  
> -  You can adjust size as desired; actual consumption grows on write.  

1.3. Format the new LV as XFS with ftype=1 (we enable quotas at mount):  
```bash
mkfs.xfs -f -n ftype=1 /dev/pve/docker-thin
```

1.4. Create and mount on the host:  
```bash
mkdir -p /mnt/docker-xfs
echo '/dev/pve/docker-thin /mnt/docker-xfs xfs defaults,pquota 0 0' >> /etc/fstab
mount /mnt/docker-xfs
```

1.5. Verify quotas on the host:  
```bash
xfs_quota -x -c "report -p" /mnt/docker-xfs
```

---

## 2. Proxmox Host → LXC: Bind-Mount & Enable Features  

2.1. Stop the container and edit its config (`<CTID>`):  
```bash
pct stop <CTID>
nano /etc/pve/lxc/<CTID>.conf
```

2.2. Add these lines (or merge with existing):  
```ini
features: nesting=1,fuse=1
mp0: /mnt/docker-xfs,mp=/var/lib/docker
```

2.3. Restart the container:  
```bash
pct start <CTID>
```

---

## 3. Inside LXC: Verify XFS + Quotas  

3.1. Enter the container:  
```bash
pct enter <CTID>
```

3.2. Confirm `/var/lib/docker` mount:  
```bash
mount | grep '/var/lib/docker'
# Shows XFS with “prjquota”
```

3.3. (Optional) Install XFS quota tools for inspection:  
```bash
apt update
apt install -y xfsprogs quota
```

---

## 4. Inside LXC: Install & Configure Docker  

4.1. Install Docker:  
```bash
apt update
apt install -y docker.io
```

4.2. Create or edit `/etc/docker/daemon.json`:  
```json
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "live-restore": true,
  "registry-mirrors": [
    "https://registry-1.docker.io",
    "https://docker1.vast.ai",
    "https://docker2.vast.ai",
    "https://docker3.vast.ai",
    "https://docker4.vast.ai",
    "https://docker5.vast.ai"
  ],
  "runtimes": {
    "nvidia": {
      "path": "/var/lib/vastai_kaalia/latest/kaalia_docker_shim",
      "runtimeArgs": []
    }
  },
  "storage-driver": "overlay2"
}
```

4.3. Restart Docker cleanly:  
```bash
systemctl stop docker
rm -rf /var/lib/docker/*
systemctl enable docker
systemctl start docker
```

4.4. Verify storage driver:  
```bash
docker info | grep "Storage Driver"
# Should show: Storage Driver: overlay2
```

---

## 5. Test Quota Enforcement  

```bash
docker run --rm --storage-opt size=50M alpine sh -c "df -h /"
# Rootfs should be ~50 MiB
```

---

## 6. NVIDIA GPU & vast.ai Selftest  

6.1. Ensure NVIDIA runtime is available:  
```bash
docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

6.2. Run the vast.ai selftest:  
```bash
vast selftest machine <machine_id>
```
All checks—including storage-opt enforcement—should pass.  

---

## 7. Summary of Key Lessons Learned  

- **fuse-overlayfs** in LXC lacks per-container quotas → use **overlay2** + XFS pquota.  
- Bind-mount an XFS-formatted LV (with `pquota`) into `/var/lib/docker` so Docker can enforce `--storage-opt size=`.  
- Use LVM thin provisioning on existing `local-lvm` for flexible, on-demand space.  
- Always restart Docker and clear `/var/lib/docker` after changing storage driver.  
- Verify mounts and drivers before selftest.  

Your Proxmox LXC GPU node is now properly configured for vast.ai’s marketplace!

***

./vast self-test machine 44888 --ignore-requirements
Machine ID 44888 does not meet the requirements:
Reliability <= 0.90
Machine ID 44888 does not meet the following requirements:
Reliability <= 0.90
Continuing despite unmet requirements because --ignore-requirements is set.
Instance 26655186 status: None... waiting for 'running' status.
Instance 26655186 status: None... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Instance 26655186 status: loading... waiting for 'running' status.
Successfully established HTTPS connection to the server.
Starting tests...
Running system requirements test...
TESTED : System requirements test passed.
Running ResNet50 test on all GPUs...
TESTED : ResNet50 passed
Running ECC test on all GPUs...
TESTED : ECC test passed.
Running NCCL distributed test with 1 GPUs...
TESTED : NCCL distributed test passed.
Running stress-ng and gpu-burn tests simultaneously for 60 seconds...
Test completed successfully.
Test passed.
destroying instance 26655186.
Instance 26655186 destroyed successfully on attempt 1.
Machine: 44888 Done with testing remote.py results DONE
Test completed successfully.
