---
layout:     post
title:      Install Xen on Ubuntu Server 15.10
date:       2015-10-24
summary:    A step-by-step tutorial for installing the Xen hypervisor on Ubuntu Server 15
categories: xen ubuntu virtualization
---
I want to run a [Mesos](https://mesos.apache.org/) cluster. I want to test [Asgard](https://github.com/Netflix/asgard), [docker](https://www.docker.com/), [Kubernetes](http://kubernetes.io/), and [Consul](https://www.consul.io/). I want to understand how docker, [LXD](http://www.ubuntu.com/cloud/tools/lxd), and [CoreOS](https://coreos.com/) work together (or do not). I have one spare machine.

This is a guide to install the [Xen hypervisor](http://www.xenproject.org/) on a fresh install of [Ubuntu Server 15.10](http://www.ubuntu.com/server). The end result is a working Xen installation with a running Ubuntu Server 15.04 guest  (aka Virtual Machine (VM)).

<!-- more -->

# Host Installation Notes
**Use Logical Volume Management (LVM) when configuring the disk and leave plenty of space free.**

![Select LVM partition option](/images/xen-ubuntu-choose-lvm.png)

I only used 25GB for the root LV, leaving nearly 1TB of room for guests. *(the screenshot below is from a different installation)*

![Set a small root size](/images/xen-ubuntu-lvm-size.png)

Only the OpenSSH Server collection needs to be installed.

![Install OpenSSH server](/images/xen-ubuntu-ssh-pkg.png)

**For the rest of the guide, `xen01` is the Xen host. All others are guests (VMs).**

# (Optional) Initial House Keeping
This section has optional items that overall make interacting with a remote system faster and easier.

## SSH Keys
Add your public key to the new Xen host to allow easier SSH access. Most operating systems include `ssh-copy-id` and can follow the *ssh-copy-id* section below. For OS X, however, follow the *Manual* method.

**NOTE:** OS X does not ship with `ssh-copy-id`, however, it is available through [Homebrew](http://brew.sh/). It can be installed with `brew install ssh-copy-id`.

### ssh-copy-id
Use `ssh-copy-id` to copy a public key (`-i`) to a target host (`xen01`).

{% highlight text linenos %}
$ ssh-copy-id -i ~/.ssh/id_rsa.pub xen01
/usr/local/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/local/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
cdcd@xen01's password:

Number of key(s) added:        1

Now try logging into the machine, with:   "ssh 'xen01'"
and check to make sure that only the key(s) you wanted were added.
$ ssh xen01
Welcome to Ubuntu 15.10 (GNU/Linux 4.2.0-16-generic x86_64)
{% endhighlight %}

### Manual
The manual way.

First, copy your public key.

{% highlight text linenos %}
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC85KQGATGvzLkwSQM1q31AmNcb06kCAmq157AGBkYTsS/kVpxTcmcNfcHHXmXlFEf5tXzkRlVxxRMM4xy0o/97LQlRi4c+4ZtVvsFmW/5qMxC2URbmW/5qMxC2URbssESn2BXSBqF7mimlAYwlIB5KMquPcuV1BW0vuqZ1PAQD934UTgngT8ziAWyNW+4cS5LBqtUu6Kv/Dp+uRNvl6Wj3o8Edu9VG8H0L47PbNxmjGaqIuDO1qjxXEfFdCbmW/5qMxC2URb2/betOnh4MW21kMr2vn8nmwUh2b+hIjE3OzlFuf8yXwAW2kJ+tesjlqyGCpOx4ZjEzfx87nWku18CymhV cdcd@here
{% endhighlight %}

Next, SSH to the target host.

{% highlight text linenos %}
$ ssh xen01
cdcd@xen01's password:
cdcd@xen01:~$
{% endhighlight %}

SSH-related items live under a user's home directory in the `.ssh` directory. It *must* be secured with mode `700`!

Public keys that are authorized to log into a user's account are stored inside the `authorized_keys` file. It also *must* be mode `700`. Paste the contents from the public key above into this file.

{% highlight text linenos %}
cdcd@xen01:~$ mkdir ~/.ssh && chmod 700 ~/.ssh
cdcd@xen01:~$ touch ~/.ssh/authorized_keys && chmod 700 ~/.ssh/authorized_keys
cdcd@xen01:~$ vi ~/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC85KQGATGvzLkwSQM1q31AmNcb06kCAmq157AGBkYTsS/kVpxTcmcNfcHHXmXlFEf5tXzkRlVxxRMM4xy0o/97LQlRi4c+4ZtVvsFmW/5qMxC2URbmW/5qMxC2URbssESn2BXSBqF7mimlAYwlIB5KMquPcuV1BW0vuqZ1PAQD934UTgngT8ziAWyNW+4cS5LBqtUu6Kv/Dp+uRNvl6Wj3o8Edu9VG8H0L47PbNxmjGaqIuDO1qjxXEfFdCbmW/5qMxC2URb2/betOnh4MW21kMr2vn8nmwUh2b+hIjE3OzlFuf8yXwAW2kJ+tesjlqyGCpOx4ZjEzfx87nWku18CymhV cdcd@here
{% endhighlight %}

Log out and `ssh` back to the host. A password prompt should not be displayed.

{% highlight text linenos %}
cdcd@xen01:~$ logout
$ ssh xen01
Welcome to Ubuntu 15.10 (GNU/Linux 4.2.0-16-generic x86_64)
{% endhighlight %}

## Update Packages
Update packages to the latest and greatest. This ensures the system has the latest software, which could include bug fixes, security fixes, and performance improvements.

{% highlight text linenos %}
cdcd@xen01:~$ sudo apt-get update && sudo apt-get -y upgrade
[ snipped ]
Reading package lists... Done
Reading package lists... Done
Building dependency tree       
Reading state information... Done
Calculating upgrade... Done
The following packages will be upgraded:
  python3-distupgrade ubuntu-release-upgrader-core
2 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Need to get 126 kB of archives.
After this operation, 0 B of additional disk space will be used.
[ snipped ]
{% endhighlight %}

## Network Configuration
If the Xen host was setup using `DHCP`, now is the time to swap to a static IP address. This will prevent issues later when a  new IP address is assigned.

**NOTE:** The below snippets use the interface `eno1` and a network of `192.168.1.0/24`. Change these values to match your system and network!

{% highlight text linenos %}
cdcd@xen01:~$ sudo vi /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eno1
iface eno1 inet static
  address 192.168.1.18
  netmask 255.255.255.0
  gateway 192.168.1.1
  # 8.8.x.x are Google
  dns-nameservers 192.168.1.1 8.8.8.8 8.8.4.4
{% endhighlight %}

## Reboot
Reboot to apply the network changes and allow updated software to refresh itself.

{% highlight text linenos %}
cdcd@xen01:~$ sudo init 6
Connection to xen01 closed by remote host.
{% endhighlight %}

# Installing Xen and Dependencies
This section mostly follows the [Beginners Guide](http://wiki.xenproject.org/wiki/Xen_Project_Beginners_Guide) available on the [Xen wiki](http://wiki.xenproject.org/wiki/Main_Page).

## Bridge Networking
Bridged networking allows guests (VMs) to access the local network directly instead of using NAT technology or being isolated to a network only accessible on the Xen host. This post uses `bridge-utils`, but this can also be accomplished through [Open vSwitch](http://wiki.xenproject.org/wiki/Xen_Networking#Open_vSwitch) as well.

Install the `bridge-utils` package.

{% highlight text linenos %}
cdcd@xen01:~$ sudo apt-get -y install bridge-utils
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  bridge-utils
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 28.6 kB of archives.
After this operation, 102 kB of additional disk space will be used.
Get:1 http://us.archive.ubuntu.com/ubuntu/ wily/main bridge-utils amd64 1.5-9ubuntu1 [28.6 kB]
Fetched 28.6 kB in 0s (81.7 kB/s)       
Selecting previously unselected package bridge-utils.
(Reading database ... 59183 files and directories currently installed.)
Preparing to unpack .../bridge-utils_1.5-9ubuntu1_amd64.deb ...
Unpacking bridge-utils (1.5-9ubuntu1) ...
Processing triggers for man-db (2.7.4-1) ...
Setting up bridge-utils (1.5-9ubuntu1) ...
{% endhighlight %}

Add a new bridge interface named `xenbr0` and confirm it is available.

{% highlight text linenos %}
cdcd@xen01:~$ sudo brctl addbr xenbr0
cdcd@xen01:~$ ifconfig -a
[ snipped lo/loopback ]
eno1      Link encap:Ethernet  HWaddr
          inet addr:192.168.1.18  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::16da:e9ff:fe4f:9aa0/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:636 errors:0 dropped:0 overruns:0 frame:0
          TX packets:342 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:83796 (83.7 KB)  TX bytes:52379 (52.3 KB)
          Interrupt:18 Memory:fbd00000-fbd20000

xenbr0    Link encap:Ethernet  HWaddr
          BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
{% endhighlight %}

Update `/etc/network/interfaces` to use the bridge interface instead of the ethernet interface. This means setting `eno1` from `static` to `manual` and moving the IP configuration to the new `xenbr0` device. The `bridge_ports` setting binds the specified interface (`eno1`) to the bridge.

The `bridge_` parameters are explained on the [debian wiki](https://wiki.debian.org/BridgeNetworkConnections#Useful_options_for_virtualised_environments). I commented the section below with snippets from the linked wiki page.

{% highlight text linenos %}
cdcd@xen01:~$ sudo vi /etc/network/interfaces
# The primary network interface
auto eno1
iface eno1 inet manual

# Xen Bridge Interface
auto xenbr0
iface xenbr0 inet static
  # bind specified ports to the bridge
  bridge_ports  eno1
  # disable Spanning Tree Protocol
  bridge_stp    off
  # no delay before a port becomes available
  bridge_maxwait        0
  # no forwarding delay
  bridge_fd     0
  address       192.168.1.18
  netmask       255.255.255.0
  gateway       192.168.1.1
  dns-nameservers 192.168.1.1 8.8.8.8 8.8.4.4
{% endhighlight %}

Reboot; ensure the bridge comes up and `ping` works.

{% highlight text linenos %}
cdcd@xen01:~$ sudo init 6
Connection to xen01 closed by remote host.
$ ping xen01
PING xen01 (xen01): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
Request timeout for icmp_seq 3
Request timeout for icmp_seq 4
Request timeout for icmp_seq 5
64 bytes from xen01: icmp_seq=6 ttl=64 time=33.443 ms
64 bytes from xen01: icmp_seq=7 ttl=64 time=2.516 ms
64 bytes from xen01: icmp_seq=8 ttl=64 time=2.612 ms
$ ssh xen01
cdcd@xen01:~$ ping www.google.com
PING www.google.com (216.58.216.132) 56(84) bytes of data.
64 bytes from sea15s01-in-f132.1e100.net (216.58.216.132): icmp_seq=1 ttl=51 time=24.9 ms
cdcd@xen01:~$ ifconfig
[ snipped lo/loopback ]
eno1      Link encap:Ethernet  HWaddr
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:123 errors:0 dropped:0 overruns:0 frame:0
          TX packets:76 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:15482 (15.4 KB)  TX bytes:10725 (10.7 KB)
          Interrupt:18 Memory:fbd00000-fbd20000

xenbr0    Link encap:Ethernet  HWaddr
          inet addr:192.168.1.18  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::16da:e9ff:fe4f:9aa0/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:123 errors:0 dropped:0 overruns:0 frame:0
          TX packets:76 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:13268 (13.2 KB)  TX bytes:10385 (10.3 KB)
{% endhighlight %}

## Checkpoint
At this point, the Xen host has:

* a fresh Ubuntu installation
* LVM support
* bridged networking configured and working

Now to install the hypervisor software, boot into the `xen` kernel, and configure Xen for optimal performance.

## Install Xen Software
This section follows the [Ubuntu Wiki article for Xen](https://help.ubuntu.com/community/Xen).

Use `apt-cache search xen-hypervisor` to find the latest version of Xen.

{% highlight text linenos %}
cdcd@xen01:~$ apt-cache search xen-hypervisor
xen-hypervisor-4.4-amd64 - Transitional package for upgrade
xen-hypervisor-4.5-amd64 - Xen Hypervisor on AMD64
{% endhighlight %}

Install the package with `sudo apt-get -y install xen-hypervisor-<VERSION>-<ARCH>`. The output below shows the installation of various Xen packages and `grub` entries.

{% highlight text linenos %}
cdcd@xen01:~$ sudo apt-get -y install xen-hypervisor-4.5-amd64
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following extra packages will be installed:
  acl cpu-checker grub-xen-bin grub-xen-host ipxe-qemu libaio1 libasound2 libasound2-data libasyncns0 libbluetooth3 libboost-system1.58.0 libboost-thread1.58.0 libbrlapi0.6
  libcaca0 libfdt1 libflac8 libjpeg-turbo8 libjpeg8 libnspr4 libnss3 libnss3-nssdb libogg0 libopus0 libpixman-1-0 libpulse0 libpython-stdlib librados2 librbd1 libsdl1.2debian
  libsndfile1 libspice-server1 libusbredirparser1 libvorbis0a libvorbisenc2 libxen-4.5 libxenstore3.0 libyajl2 msr-tools python python-minimal python2.7 python2.7-minimal
  qemu-block-extra qemu-system-common qemu-system-x86 qemu-utils seabios sharutils xen-utils-4.5 xen-utils-common xenstore-utils
Suggested packages:
  libasound2-plugins alsa-utils opus-tools pulseaudio python-doc python-tk python2.7-doc binutils binfmt-support samba vde2 sgabios ovmf debootstrap bsd-mailx mailx
Recommended packages:
  xen-hypervisor-4.5
The following NEW packages will be installed:
  acl cpu-checker grub-xen-bin grub-xen-host ipxe-qemu libaio1 libasound2 libasound2-data libasyncns0 libbluetooth3 libboost-system1.58.0 libboost-thread1.58.0 libbrlapi0.6
  libcaca0 libfdt1 libflac8 libjpeg-turbo8 libjpeg8 libnspr4 libnss3 libnss3-nssdb libogg0 libopus0 libpixman-1-0 libpulse0 libpython-stdlib librados2 librbd1 libsdl1.2debian
  libsndfile1 libspice-server1 libusbredirparser1 libvorbis0a libvorbisenc2 libxen-4.5 libxenstore3.0 libyajl2 msr-tools python python-minimal python2.7 python2.7-minimal
  qemu-block-extra qemu-system-common qemu-system-x86 qemu-utils seabios sharutils xen-hypervisor-4.5-amd64 xen-utils-4.5 xen-utils-common xenstore-utils
0 upgraded, 52 newly installed, 0 to remove and 0 not upgraded.
Need to get 16.8 MB of archives.
After this operation, 63.9 MB of additional disk space will be used.
[ snipped ]
Setting up xen-hypervisor-4.5-amd64 (4.5.1-0ubuntu1) ...
Including Xen overrides from /etc/default/grub.d/xen.cfg
WARNING: GRUB_DEFAULT changed to boot into Xen by default!
         Edit /etc/default/grub.d/xen.cfg to avoid this warning.
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.2.0-16-generic
Found initrd image: /boot/initrd.img-4.2.0-16-generic
Found linux image: /boot/vmlinuz-4.2.0-16-generic
Found initrd image: /boot/initrd.img-4.2.0-16-generic
Found linux image: /boot/vmlinuz-4.2.0-16-generic
Found initrd image: /boot/initrd.img-4.2.0-16-generic
done
Setting up xenstore-utils (4.5.1-0ubuntu1) ...
Setting up xen-utils-common (4.5.1-0ubuntu1) ...

Creating config file /etc/default/xen with new version
Setting up xen-utils-4.5 (4.5.1-0ubuntu1) ...
Setting up grub-xen-bin (2.02~beta2-29) ...
Setting up grub-xen-host (2.02~beta2-29) ...
Setting up libnss3-nssdb (2:3.19.2-1ubuntu1) ...
Setting up libnss3:amd64 (2:3.19.2-1ubuntu1) ...
Setting up librados2 (0.94.3-0ubuntu2) ...
Setting up librbd1 (0.94.3-0ubuntu2) ...
Setting up qemu-block-extra:amd64 (1:2.3+dfsg-5ubuntu9) ...
Setting up qemu-system-common (1:2.3+dfsg-5ubuntu9) ...
Setting up qemu-system-x86 (1:2.3+dfsg-5ubuntu9) ...
Setting up qemu-utils (1:2.3+dfsg-5ubuntu9) ...
Processing triggers for libc-bin (2.21-0ubuntu4) ...
Processing triggers for ureadahead (0.100.0-19) ...
Processing triggers for systemd (225-1ubuntu9) ...
{% endhighlight %}

### Configuration Changes
Update the Xen `grub` config to [limit the memory `dom0` consumes](http://wiki.xenproject.org/wiki/Tuning_Xen_for_Performance#Memory) to 512MB. Run `update-grub` to apply the changes.

{% highlight text linenos %}
cdcd@xen01:~$ sudo vi /etc/default/grub.d/xen.cfg
#
# Uncomment the following variable and set to 0 or 1 to avoid warning.
#
XEN_OVERRIDE_GRUB_DEFAULT=1

# The following two are used to generate arguments for the hypervisor:
#
GRUB_CMDLINE_XEN_DEFAULT="dom0_mem=512M,max:512M"
#GRUB_CMDLINE_XEN=""

[ snipped ]

cdcd@xen01:~$ sudo update-grub
Including Xen overrides from /etc/default/grub.d/xen.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.2.0-16-generic
Found initrd image: /boot/initrd.img-4.2.0-16-generic
Found linux image: /boot/vmlinuz-4.2.0-16-generic
Found initrd image: /boot/initrd.img-4.2.0-16-generic
Found linux image: /boot/vmlinuz-4.2.0-16-generic
Found initrd image: /boot/initrd.img-4.2.0-16-generic
done
{% endhighlight %}

Disable [auto-ballooning](http://xenbits.xen.org/docs/unstable/man/xl.conf.5.html) in `xl` by modifying `/etc/xen/xl.conf`.

{% highlight text linenos %}
cdcd@xen01:~$ sudo vi /etc/xen/xl.conf
autoballoon="off"
vif.default.bridge="xenbr0"
{% endhighlight %}

## Reboot
Reboot the system onto the Xen `grub` entry.

{% highlight text linenos %}
cdcd@xen01:~$ sudo init 6
Connection to xen01 closed by remote host.
{% endhighlight %}

# xl list
Upon restart, list the domains currently running and verify only `Domain-0` is up and operating.

{% highlight text linenos %}
cdcd@xen01:~$ sudo xl list
Name                                        ID   Mem VCPUs	State	Time(s)
Domain-0                                     0   511     8     r-----       8.2
{% endhighlight %}

`dom0` is running (`r-----`) with 512MB of memory and 8 vCPUs.

## Troubleshooting
If `xl list` did not return successfully then Xen is not correctly installed or running.

One possibility is that the Xen grub entry was not selected upon restart. This will require manual intervention during the server boot-up or modifying the grub config to change the `GRUB_DEFAULT` entry in `/etc/default/grub`.

# First Guest
According to [Xen terminology](http://wiki.xenproject.org/wiki/XenTerminology):

> [a guest is] an operating system that can run within the Xen environment

The first guest will be an Ubuntu 15.04 (vivid) installation. The steps below result in a console session to perform a typical Ubuntu installation. This *does not use* [preseeding](http://wiki.xen.org/wiki/Debian_Guest_Installation_Using_Debian_Installer#Preseeding) -- that is for another post.

This follows the [Xen article on the Ubuntu wiki](https://help.ubuntu.com/community/Xen#Manually_Create_a_PV_Guest_VM).

### Download Boot Media
A VM needs to load and run something; to perform a new installation that means loading and running the Ubuntu installer. The process below is for the Ubuntu net installer.

There are two steps:

1. create a directory to store the images (`/var/lib/xen/images/ubuntu/vivid`)
1. download `vmlinuz` and `initrd.gz` from a [mirror](https://launchpad.net/ubuntu/+archivemirrors)

{% highlight text linenos %}
cdcd@xen01:~$ sudo mkdir -p /var/lib/xen/images/ubuntu/vivid
cdcd@xen01:~$ cd /var/lib/xen/images/ubuntu/vivid
cdcd@xen01:~$ sudo wget http://mirror.pnl.gov/ubuntu/dists/vivid/main/installer-amd64/current/images/netboot/xen/initrd.gz
cdcd@xen01:~$ sudo wget http://mirror.pnl.gov/ubuntu/dists/vivid/main/installer-amd64/current/images/netboot/xen/vmlinuz
{% endhighlight %}

### Create Logical Volume for Guest
Create a new Logical Volume (`lvcreate`) to use for the guest's disk. The initial Volume Group (VG) should have been created during the installation of the Xen host. The new LV needs a name (`-n`) and a size (`-L`).

In the examples below, the resulting VM will be a [Mesos Master](https://mesos.apache.org/) and the name `mmaster0` is used with an installation size of `50G`. Your values will differ.

First, find the name of the Volume Group that will hold the new volume using the `vgs` command. The output below shows a VG named `xen01-vg` with 2 volumes already created and over 900GB of disk space free.

{% highlight text linenos %}
cdcd@xen01:~$ sudo vgs
VG       #PV #LV #SN Attr   VSize   VFree  
xen01-vg   1   2   0 wz--n- 930.77g 907.49g
{% endhighlight %}

To create a new volume on the VG from above, use the `lvcreate` command. The format is `lvcreate -L <SIZE> -n <NAME> /dev/<VOLUME GROUP NAME>`.

{% highlight text linenos %}
cdcd@xen01:~$ sudo lvcreate -L 50G -n mmaster01 /dev/xen01-vg
Logical volume "mmaster01" created.
cdcd@xen01:~$ sudo lvs
LV        VG       Attr       LSize
mmaster01 xen01-vg -wi-a----- 50.00g
root      xen01-vg -wi-ao----  7.38g
swap_1    xen01-vg -wi-ao---- 15.91g
{% endhighlight %}

### Create Guest Configuration
Copy the provided `/etc/xen/xlexample.pvlinux` configuration file and tailor it to the values from above. The values that must be changed are: `name`, `kernel`, `ramdisk`, and `disk`.

The suffix `pvlinux` refers to [paravirtualization](http://wiki.xen.org/wiki/Paravirtualization_(PV)), which is more lightweight and efficient than its [full virtualization (HVM)](http://wiki.xenproject.org/wiki/Xen_Project_Software_Overview#HVM) counterpart. However, it requires that the guest operating system has proper kernel and driver support (which Ubuntu has). Visit the Xen wiki for more [information on guest types](http://wiki.xenproject.org/wiki/Xen_Project_Software_Overview#Guest_Types).

{% highlight text linenos %}
cdcd@xen01:~$ cd /etc/xen
cdcd@xen01:/etc/xen$ sudo cp xlexample.pvlinux mmaster01.cfg
cdcd@xen01:/etc/xen$ sudo vi mmaster01.cfg
# Guest name
name = "mmaster01"

# Kernel image to boot
kernel = "/var/lib/xen/images/ubuntu/vivid/vmlinuz"

# Ramdisk (optional)
ramdisk = "/var/lib/xen/images/ubuntu/vivid/initrd.gz"

# Initial memory allocation (MB)
memory = 2048

# Number of VCPUS
vcpus = 2

# Network devices
vif = [ '' ]

# Disk Devices
disk = [ '/dev/xen01-vg/mmaster01,raw,xvda,rw' ]
{% endhighlight %}

### Run It

Create the new guest with `xl create` and immediately connect to the console (`-c`). Perform a standard installation and reboot the instance when instructed.

**NOTE:** To exit the console, use `CTRL+]` (`^]`)

{% highlight text linenos %}
cdcd@xen01:~$ sudo xl create -c /etc/xen/mmaster01.cfg
Parsing config from /etc/xen/mmaster01.cfg
{% endhighlight %}

### Finalize VM Config
The configuration from the step before points to the installation media, which is great for installing a new system. However, to actually run the installed system the boot manager on the guest's disk needs to be executed.

Xen provides [PyGrub](http://wiki.xen.org/wiki/PyGrub) -- a grub-like bootloader written in Python -- for this purpose (running a guest's boot manager). To use `pygrub`, the guest's configuration needs to updated to use a `bootloader` and to no longer use `kernel` and `ramdisk`. Let's do that now.

First, shut down the guest if it is still running:

{% highlight text linenos %}
cdcd@xen01:~$ sudo xl shutdown mmaster01
{% endhighlight %}

Next, locate the `pygrub` application:

{% highlight text linenos %}
cdcd@xen01:~$ find / -name 'pygrub' 2>/dev/null
/usr/lib/xen-4.5/bin/pygrub
{% endhighlight %}

Finally, update the guest configuration:

* comment out (`#`) or remove `kernel` and `ramdisk`
* add `bootloader`
* leave the rest as-is

{% highlight text linenos %}
cdcd@xen01:~$ sudo vi /etc/xen/mmaster01.cfg
# Guest name
name = "mmaster01"

# Kernel image to boot
#kernel = "/var/lib/xen/images/ubuntu/vivid/vmlinuz"

# Ramdisk (optional)
#ramdisk = "/var/lib/xen/images/ubuntu/vivid/initrd.gz"

# bootloader
bootloader = "/usr/lib/xen-4.5/bin/pygrub"

# Initial memory allocation (MB)
memory = 2048

# Number of VCPUS
vcpus = 2

# Network devices
vif = [ '' ]

# Disk Devices
disk = [ '/dev/xen01-vg/mmaster01,raw,xvda,rw' ]
{% endhighlight %}

### Final Run
And now re-run `xl create`.

{% highlight text linenos %}
cdcd@xen01:~$ sudo xl create -c /etc/xen/mmaster01.cfg
Parsing config from /etc/xen/mmaster01.cfg

pyGRUB  version 0.6
┌────────────────────────────────────────────────────────────────────────┐
│ Ubuntu                                                                 │
│ Ubuntu, with Linux 3.19.0-31-generic                                   │
│ Ubuntu, with Linux 3.19.0-31-generic (recovery mode)                   │
│                                                                        │
│                                                                        │
│                                                                        │
│                                                                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
 Use the ^ and ┴ keys to select which entry is highlighted.
 Press enter to boot the selected OS, 'e' to edit the
 commands before booting, 'a' to modify the kernel arguments
 before booting, or 'c' for a command line.

[ snipped boot-up log ]

Ubuntu 15.04 mmaster01 hvc0

mmaster01 login:
{% endhighlight %}
