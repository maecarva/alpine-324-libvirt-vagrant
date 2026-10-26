Download alpine x64 limine cloudinit virtual

cp alpine-3.24.2-x86_64-cloudinit-r0.qcow2 box.img

```bash
$ qemu-img info box.img

image: box.img
file format: qcow2
virtual size: 265 MiB (277872640 bytes)
disk size: 168 MiB
cluster_size: 65536
Format specific information:
compat: 1.1
compression type: zlib
lazy refcounts: false
refcount bits: 16
corrupt: false
extended l2: false
Child node '/file':
filename: box.img
protocol type: file
file length: 168 MiB (175636480 bytes)
disk size: 168 MiB
```
# Build

## Resize virtual disk size

`qemu-img resize box.img 2G`

## SSH key :

```bash
ssh-keygen \
  -t ed25519 \
  -f ~/.ssh/alpine-box-builder \
  -N ""
```

```bash
BUILDER_KEY="$(cat ~/.ssh/alpine-box-builder.pub)"
```

## User data & meta data files

user-data

```
cat > user-data <<EOF
#cloud-config
hostname: alpine-box-builder

ssh_pwauth: false

users:
  - default

ssh_authorized_keys:
  - ${BUILDER_KEY}
EOF
```

meta-data

```
cat > meta-data <<EOF
instance-id: alpine-box-builder-001
local-hostname: alpine-box-builder
EOF
```

## build nocloud image

```
genisoimage \
  -output seed.iso \
  -volid cidata \
  -joliet \
  -rock \
  user-data meta-data
```

## Copy in libvirt pool

```bash
sudo cp box.img \
  /var/lib/libvirt/images/alpine324-builder.qcow2

sudo cp seed.iso \
  /var/lib/libvirt/images/alpine324-seed.iso

# permissions
sudo chown qemu:qemu \
  /var/lib/libvirt/images/alpine324-builder.qcow2 \
  /var/lib/libvirt/images/alpine324-seed.iso


# selinux reset secority context
sudo restorecon -v \
  /var/lib/libvirt/images/alpine324-builder.qcow2 \
  /var/lib/libvirt/images/alpine324-seed.iso
```

## Boot image

```bash
sudo virt-install \
  --name alpine324-builder \
  --memory 1024 \
  --vcpus 1 \
  --import \
  --disk path=/var/lib/libvirt/images/alpine324-builder.qcow2,format=qcow2,bus=virtio \
  --disk path=/var/lib/libvirt/images/alpine324-seed.iso,device=cdrom \
  --network network=default,model=virtio \
  --os-variant generic \
  --graphics none \
  --console pty,target_type=serial \
  --noautoconsole
```

### healthchecks

```bash
sudo virsh list
 Id   Name                State
-----------------------------------
 1    alpine324-builder   running

 sudo virsh net-dhcp-leases default
 Expiry Time           MAC address         Protocol   IP address           Hostname             Client ID or DUID
----------------------------------------------------------------------------------------------------------------------------------------------------------
 2026-10-06 13:07:43   52:54:00:17:5f:ac   ipv4       192.168.122.184/24   alpine-box-builder   ff:00:17:5f:ac:00:01:00:01:32:57:84:ea:52:54:00:17:5f:ac
```

## SSH into vm

```bash
ssh \
  -i ~/.ssh/alpine-box-builder \
  -o StrictHostKeyChecking=no \
  alpine@192.168.122.184
Warning: Permanently added '192.168.122.184' (ED25519) to the list of known hosts.
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org>.

Alpine release notes:
* <https://alpinelinux.org/posts/Alpine-3.21.8-3.22.6-3.23.6-3.24.2-released.html>

NOTE: 'sudo' is not installed by default, please use 'doas' instead.

You may change this message by editing /etc/motd.
alpine-box-builder:~$
```

# Setup

## install packages

```bash
doas apk add --no-cache sudo bash python3 util-linux
```

## vagrant user

Create user:

```bash
doas adduser -D -s /bin/bash vagrant
doas addgroup vagrant wheel
```

change password:
`echo 'vagrant:vagrant' | doas chpasswd`

configure sudo:

```bash
echo 'vagrant ALL=(ALL) NOPASSWD: ALL' \
  | doas tee /etc/sudoers.d/vagrant >/dev/null

doas chmod 0440 /etc/sudoers.d/vagrant
doas visudo -cf /etc/sudoers
```

SSH:
send default vagrant key from host to vm

```bash
scp \
  -i ~/.ssh/alpine-box-builder \
  -o StrictHostKeyChecking=no \
  /opt/vagrant/embedded/gems/gems/vagrant-2.4.9/keys/vagrant.pub.rsa \
  alpine@VMIP:/tmp/vagrant.pub
```

same for ` /opt/vagrant/embedded/gems/gems/vagrant-2.4.9/keys/vagrant.pub.ed25519`

in vm :

```bash
cat /tmp/vagrant.pub\
  | sudo tee -a /home/vagrant/.ssh/authorized_keys >/dev/null

sudo chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys
sudo chmod 600 /home/vagrant/.ssh/authorized_keys

rm /tmp/vagrant.pub
```

## SSH tweaks

```bash
echo 'UseDNS no' | doas tee /etc/ssh/sshd_config.d/99-vagrant.conf
doas rc-service sshd restart
```

## Check networking

```bash
alpine-box-builder:~$ cat /etc/network/interfaces
# This file is generated from information provided by the datasource. Changes
# to it will not persist across an instance reboot. To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}

auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

# control-alias eth0
iface eth0 inet6 dhcp
```

eth0 dhcp

## Disable could-init config from now

```bash
echo 'network: {config: disabled}' \
  | doas tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

```bash
for svc in cloud-init-local cloud-init cloud-config cloud-final cloud-init-hotplugd; do
  doas rc-update del "$svc" boot 2>/dev/null || true
  doas rc-update del "$svc" default 2>/dev/null || true
done
```

Check if `rc-update | grep cloud` return nothing

`doas rm -rf /var/lib/cloud/*`

## Final steps

Remove temp ssh key : `doas rm -rf /home/alpine/.ssh`
Change hostname: `echo 'alpine' | doas tee /etc/hostname`

`doas poweroff`

Unmount seed.iso:

```bash
sudo virsh domblklist alpine324-builder

vda   /var/lib/libvirt/images/alpine324-builder.qcow2
hda   /var/lib/libvirt/images/alpine324-seed.iso

sudo virsh detach-disk alpine324-builder hda --config
```

# Reboot

start : `sudo virsh start alpine324-builder`
getip : `sudo virsh net-dhcp-leases default`
ssh :

```bash
ssh \
  -o IdentitiesOnly=yes \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  -i /opt/vagrant/embedded/gems/gems/vagrant-2.4.9/keys/vagrant.key.rsa \
  vagrant@IP
```

Check output of theses commands:

```bash
whoami # vagrant
sudo -n whoami # root
python3 --version
ip a # eth0 with DHCP IP
ip route # internet access
rc-status # sshd and networking started
```

## Clean VM

Check if sshd can get host keys:
`grep -n 'ssh-keygen' /etc/init.d/sshd`
if ssh-keygen -A -> GOOD

Next :

```bash
sudo apk cache clean
sudo rm -rf /var/cache/apk/*

sudo find /var/log -type f -exec truncate -s 0 {} \;

rm -f ~/.bash_history ~/.ash_history

sudo rm -f  /root/.bash_history /home/alpine/.ash_history /home/alpine/.bash_history /home/vagrant/.ash_history /home/vagrant/.bash_history

sudo rm -rf /var/lib/cloud/*
sudo apk del cloud-init

sudo rm -f /etc/ssh/ssh_host_*

sync
sudo poweroff
```

# Create box

```bash
cd ~/vagrant-boxes/alpine324
sudo cp --sparse=always \
  /var/lib/libvirt/images/alpine324-builder.qcow2 \
  alpine-3.24.2-vagrant.qcow2

sudo chown "$USER:$USER" alpine-3.24.2-vagrant.qcow2


qemu-img info alpine-3.24.2-vagrant.qcow2
# check :
# file format: qcow2
# virtual size: 2 GiB
# no baking files
```

```bash
cd ~/vagrant-boxes
git clone --depth 1 \
  https://github.com/vagrant-libvirt/vagrant-libvirt.git
```

Box file

```bash
 ~/vagrant-boxes/vagrant-libvirt/tools/create_box.sh \
  alpine-3.24.2-vagrant.qcow2 \
  alpine-3.24.2-libvirt.box
```

Succes if :

```bash
==> Creating box, tarring and gzipping
./metadata.json
./Vagrantfile
./box.img
Total bytes written: 208926720 (200MiB, 569MiB/s)
==> alpine-3.24.2-libvirt.box created
==> You can now add the box:
==>   'vagrant box add alpine-3.24.2-libvirt.box --name alpine-3.24.2-libvirt'
```

# Import box into vagrant

```bash
vagrant box add \
  alpine-3.24.2-libvirt.box \
  --name mael/alpine324
```

`vagrant box list`
