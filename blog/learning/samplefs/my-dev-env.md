# My Dev Env
Buildroot and Yocto seem good to be used to customize rootfs and kernel.But from the view of learning kernel, I think these platforms are not straight. So I just do something straight, also a little boring. This doc descipes what I am using to build kernel v3.13-rc8 to do something like that [samplefs](https://github.com/LiuPeibao/samplefs).

## 1 Build Kernel in Docker
Just more than I can say, docker is really a good chioce to build something, something maybe old-fashioned.
### 1.1 install
#install docker
```
apt-get install docker docker.io
```
#run without sudo
```
sudo groupadd docker
sudo usermod -aG docker $USER
su $USER
```
#if inner repo
```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
sudo vim /lib/systemd/system/docker.service
add this option to docker daemon start:
--insecure-registry $xxxxxxx:xxxxx

sudo systemctl daemon-reload
sudo systemctl start docker
```
#use centos7
```
docker pull centos:centos7
```
### 1.2 start
#create docekr
```
#! /bin/bash

set -e

workspace='/home/liu/project_local/linux-3.13-rc8'

function start_docker() {
    docker run -dt \
	       -u 0 \
	       -v $workspace:/home/liu/project_local/linux-3.13-rc8/ \
	       --name centos7-build-env \
	       centos:centos7 \
	       /bin/bash
}

start_docker
```
#start docker
```
docker start centos7-for-build
```
#enter docker with root
```
docker exec -it -u 0 centos7-for-build /bin/bash
```
### 1.3 basic image config
#install basic pkgs
```
yum -y install gcc
yum -y install vim
yum -y install git
yum -y install sudo
yum -y install make
yum -y install ncurses-devel
yum -y install bc
yum -y install gdb

#to compile perf
yum -y install flex
yum -y install bison
yum -y install elfutils-libelf-devel
yum -y install elfutils-devel
yum -y install libunwind-devel
yum -y install audit-libs-devel
yum -y install slang-devel
yum -y install gtk2-devel
yum -y install binutils-devel
yum -y install numactl-devel
yum -y install python-devel
yum -y install asciidoc
yum -y install xmlto
```
#down grade binutils. I have encontered with kernel boot up failure due to binutils.
```
sudo rpm -ivh binutils-2.25.1-22.base.el7.x86_64.rpm --force
```
#add myself
```
[root@98ff0dcac033 /]# useradd liu -u 1000 -d /home/liu
[root@98ff0dcac033 /]# passwd root
***
set as root
***
[root@98ff0dcac033 /]# passwd liu
***
set as kernel
***
[root@98ff0dcac033 /]# vi /etc/sudoers
***
add the following:
liu ALL=(ALL) NOPASSWD: ALL
***
root@c411134df9cb:/# su liu
[liu@020175495bca /]$ git config --global --add user.name "Liu Peibao"
[liu@020175495bca /]$ git config --global --add user.email "liupeibao@163.com"
[liu@020175495bca /]$ 
```
#enter docker with myself
```
docker exec -it -u liu centos7-for-build /bin/bash
```
### 1.4 build kernel
#clone kernel src
```
git clone https://github.com/torvalds/linux.git
```
#switch to 3.13-rc8
```
git checkout -b source-3.13-rc8 v3.13-rc8
```
#build x86_64 kernel
```
cd linux
mkdir build
make help
make --help
make x86_64_defconfig O=./build/
make kvmconfig O=./build/
make distclean
cd build
make menuconfig
make -j72
```
#build perf
```
cd build
make tools/perf
make tools/perf_install

#perf default installed in
ls -l ~/bin/perf
#add to usr/bin
sudo ln -s /home/liu/bin/perf /usr/bin/perf
```
#build module
```
cd build
make M=fs/samplefs
make M=fs/samplefs clean
```
#to enable more features on ftrace
```
make menuconfig

Kernel hacking  --->
[*] Tracers  ---> 
[*]   Kernel Function Tracer
[*]     Kernel Function Graph Tracer
```
#to use crash
```
DEBUG_KERNEL
DEBUG_INFO
```
## 2 Boot up Kernel by Qemu
### 2.1 rootfs
Both of ubuntu and centos are excellent.
#### 2.1.1 ubuntu 16.04
#create rootfs
```
cdimage.ubuntu.com/ubuntu-base/releases/16.04/release/ubuntu-base-16.04-core-amd64.tar.gz
dd if=/dev/zero of=ubuntu-base-16.04-core-amd64.rootfs.ext4 bs=1MB count=2048
mkfs.ext4 -O ^64bit,^metadata_csum ubuntu-base-16.04-core-amd64.rootfs.ext4
sudo mount ubuntu-base-16.04-core-amd64.rootfs.ext4 /mnt
sudo tar -xf ubuntu-base-16.04-core-amd64.tar.gz -C /mnt
sudo umount /mnt
```
#initialize root passward
```
#add root
cd build
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -hda ubuntu-base-16.04-core-amd64.rootfs.ext4 -append "root=/dev/sda rw init=/bin/bash"
#enter the system
passwd root
***
set as root
***
#to solve known boot up failure about ubuntu in qemu
ln -s /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@ttyS0.service
poweroff
# enter system with init
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -hda ubuntu-base-16.04-core-amd64.rootfs.ext4 -append "root=/dev/sda rw init=/sbin/init"
```
#basic configuration
```
# on host
sudo mount ubuntu-base-16.04-core-amd64.rootfs.ext4 /mnt/

#if needed, update the app sources
sudo cp /mnt/etc/apt/sources.list /mnt/etc/apt/sources.list.bak
sudo cp /etc/apt/sources.list /mnt/etc/apt/sources.list
cd /mnt
sudo chroot ./

#just as host
echo "search ***.***" >> /etc/resolv.conf
echo "nameserver ***.***.***.***" >> /etc/resolv.conf

#install pkgs
apt-get update
apt-get install vim
apt-get install ssh
apt-get install iproute2
apt-get install net-tools
apt-get install ifupdown2
apt-get install kmod
apt-get install iputils-ping
apt-get install crash

#configure network to allow ssh
echo "auto eth0" >> /etc/network/interfaces
echo "iface eth0 inet dhcp" >> /etc/network/interfaces
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
sync

cd
umount /mnt
```
#now boot up
```
qemu-system-x86_64 -net nic -net user,hostfwd=tcp::3456-:22 -kernel arch/x86/boot/bzImage -hda ubuntu-base-16.04-core-amd64.rootfs.ext4 -append "root=/dev/sda rw init=/sbin/init"
#with kvm
qemu-system-x86_64 -enable-kvm -m 1G -smp 2 -net nic -net user,hostfwd=tcp::3456-:22 -kernel arch/x86/boot/bzImage -hda ubuntu-base-16.04-core-amd64.rootfs.ext4 -append "root=/dev/sda rw init=/sbin/init"
```
#### 2.1.2 centos 7
#create rootfs
```
#pack the docker image
sudo tar -cf centos-7-core-amd64.tar / --exclude=/proc --exclude=/sys --exclude=/home/wrsadmin
dd if=/dev/zero of=centos-7-core-amd64.rootfs.ext4 bs=1MB count=2048
mkfs.ext4 -O ^64bit,^metadata_csum centos-7-core-amd64.rootfs.ext4 
sudo mount centos-7-core-amd64.rootfs.ext4 /mnt
sudo tar -xf centos7-docker-rootfs.tar -C /mnt
sudo umount /mnt
```
#boot up
```
qemu-system-x86_64 -enable-kvm -nographic -net nic -net user,hostfwd=tcp::3456-:22 -kernel arch/x86/boot/bzImage -hda centos-7-core-amd64.rootfs.ext4 -append "root=/dev/sda rw init=/sbin/init console=ttyS0"
```
#install pkgs
```
yum install net-tools
yum install initscripts
yum install NetworkManager
yum install openssl
yum install openssh-server
yum install crash
yum install strace
```
#since the rootfs from the docker, the following could be ignored
```
echo "search ***.***" >> /etc/resolv.conf
echo "nameserver ***.***.***.***" >> /etc/resolv.conf
```
#network
```
cd /etc/sysconfig/network-scripts
cat ifcfg-enp0s3
-----
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="dhcp"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
DEVICE="enp0s3"
ONBOOT="yes
-----
```
### 2.2 ssh
#ssh qemu
```
#login
ssh -p 3456 root@localhost
#copy files
scp -P 3456 fs/samplefs/samplefs.ko root@localhost:/root
```
