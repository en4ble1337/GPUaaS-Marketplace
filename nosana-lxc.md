notes:
install nvidia + docker + nvidia docker + essentials
run as user with docker priv



notes: features and lxc mouunts are important for podman
lxc config
cat /etc/pve/lxc/6004.conf


arch: amd64

cores: 48

#features: nesting=1

features: nesting=1,keyctl=1

hostname: gpuaas-3

memory: 249856

net0: name=eth0,bridge=vmbr0,firewall=1,hwaddr=BC:24:11:E3:63:F2,ip=dhcp,tag=30,type=veth

ostype: ubuntu

rootfs: hdd1:vm-6004-disk-0,size=2T

swap: 0

tags: nosana

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

#mount entries

lxc.cgroup2.devices.allow: c 10:229 rwm

lxc.mount.entry: /dev/fuse dev/fuse none bind,optional,create=file

#original

lxc.mount.entry: /dev/loop0 dev/loop0 none bind,optional,create=file

lxc.mount.entry: /dev/loop1 dev/loop1 none bind,optional,create=file

lxc.mount.entry: /dev/loop2 dev/loop2 none bind,optional,create=file

lxc.mount.entry: /dev/loop3 dev/loop3 none bind,optional,create=file

lxc.mount.entry: /dev/loop4 dev/loop4 none bind,optional,create=file

lxc.mount.entry: /dev/loop5 dev/loop5 none bind,optional,create=file

lxc.mount.entry: /dev/loop6 dev/loop6 none bind,optional,create=file

lxc.mount.entry: /dev/loop7 dev/loop7 none bind,optional,create=file

lxc.mount.entry: /dev/loop-control dev/loop-control none bind,optional,create=file
