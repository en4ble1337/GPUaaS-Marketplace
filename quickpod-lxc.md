cat /etc/pve/lxc/6006.conf
```bash
#try running speedtest first then run the following
#/var/lib/quickpod/update_scripts.sh
arch: amd64
cores: 46
features: nesting=1,keyctl=1
hostname: gpuaas-6
memory: 249856
net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=BC:24:11:A7:50:B3,ip=dhcp,tag=30,type=veth
ostype: ubuntu
rootfs: hdd1:vm-6006-disk-0,size=2T
swap: 0
tags: quickpod
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
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop: 
lxc.cgroup2.devices.allow: b 7:* rwm
lxc.cgroup2.devices.allow: c 10:237 rwm
lxc.mount.entry: /dev/loop-control dev/loop-control none bind,optional,create=file
lxc.mount.entry: /dev/loop0 dev/loop0 none bind,optional,create=file
lxc.mount.entry: /dev/loop1 dev/loop1 none bind,optional,create=file
lxc.mount.entry: /dev/loop2 dev/loop2 none bind,optional,create=file
lxc.mount.entry: /dev/loop3 dev/loop3 none bind,optional,create=file
lxc.mount.entry: /dev/loop4 dev/loop4 none bind,optional,create=file
lxc.mount.entry: /dev/loop5 dev/loop5 none bind,optional,create=file
lxc.mount.entry: /dev/loop6 dev/loop6 none bind,optional,create=file
lxc.mount.entry: /dev/loop7 dev/loop7 none bind,optional,create=file
lxc.init.cmd: /sbin/init systemd.unified_cgroup_hierarchy=0
```

cat /etc/docker/daemon.json
```bash
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "args": [],
      "runtimeArgs": []
    }
  }
}

/var/lib/quickpod/machine_open_port_range
```
