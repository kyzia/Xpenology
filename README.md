**Xpenology AMD hardware virtualization**

Hi guys, this is instruction for kvm-amd.ko and libvirtd launch om HP MicroServer GEN 10 ( AMD Opteron X3216 APU with svm ( AMD-V ))

TL/DR: I'm try to use virtualization with hardware acceleration on Hp MicroServer Gen 10 and succesfully launch Ubuntu 18.04 with it. What else needs to be done here:

 * launch virtual machines via internal VMM manager still failed (error "Failed to power on the virtual machine [somemachine] on the host [DiskStation]." )
 * i'm use raw for qemu (but you can make qcow2 image somewere else)
 * write some autostart scripts

**1. Install and update debian chroot**

You can do it in your own system or on Synology NAS.
If you preferred Xpenology - follow instruction from here ( install package from Synology Community ):

https://github.com/SynoCommunity/spksrc/wiki/Debian-Chroot

Chroot to installed package (you will need to login by ssh to your Synology, installing chroot takes long time. Check via "ps ajfx" in dom0 that all postinst actions are over):

```
okkk@SWARM:~$ ssh oleg@10.0.0.8
oleg@10.0.0.8's password: 
oleg@DiskStation:~$ sudo /var/packages/debian-chroot/scripts/start-stop-status chroot
Password:
root@DiskStation:/# apt-get update
root@DiskStation:/# apt-get upgrade
root@DiskStation:/# echo "Europe/Moscow" > /etc/timezone 
root@DiskStation:/# dpkg-reconfigure -f noninteractive tzdata
root@DiskStation:/# apt-get install locales
root@DiskStation:/# dpkg-reconfigure locales
```


**2. Prepare enviroment**

This instruction was taken from https://blog.4sag.ru/kompilyatsiya-modulej-dsm-6-2/ , all the same, but kernel version is different.
```
root@DiskStation:/# apt-get install mc make gcc build-essential kernel-wedge libncurses5 libncurses5-dev libelf-dev binutils-dev kexec-tools makedumpfile fakeroot lzma bc libssl-dev openssl vim netcat-openbsd 
root@DiskStation:/# mkdir /dsmsrc
root@DiskStation:/# cd /dsmsrc
```

Get sources:

```
root@DiskStation:/# wget --no-check-certificate https://sourceforge.net/projects/dsgpl/files/Synology%20NAS%20GPL%20Source/24922branch/apollolake-source/linux-4.4.x.txz
root@DiskStation:/# wget --no-check-certificate https://sourceforge.net/projects/dsgpl/files/DSM%206.2%20Tool%20Chains/Intel%20x86%20Linux%204.4.59%20%28Apollolake%29/apollolake-gcc493_glibc220_linaro_x86_64-GPL.txz

root@DiskStation:/# tar -xvf apollolake-gcc493_glibc220_linaro_x86_64-GPL.txz 
root@DiskStation:/# tar -xvf linux-4.4.x.txz 
root@DiskStation:/# cd linux-4.4.x
```

Fix Makefile

```
root@DiskStation:/# vim Makefile 
```

Change line:
EXTRAVERSION = 
to:
EXTRAVERSION = +
(if yo dont do this - after executing insmod for file kvm-amd.ko you get "version magic '4.4.59 SMP preempt mod_unload ' should be '4.4.59+ SMP preempt mod_unload '" message in dmesg)

```
root@DiskStation:/# cp synoconfigs/apollolake .config

root@DiskStation:/# dsm6make menuconfig ( 1. Go to Virtualization ->  KVM for AMD processors support and put M symbol here; 2. Go to one level up, to Device Drivers -> Network device support -> Universal TUN/TAP device driver support and put M symbol here if it does not exist; 3. Select Save in menu and save this config as .config)
```

Check diff with default config (tun device turned on by default, so tun doesnot visible in this diff):

```
root@DiskStation:/dsmsrc/linux-4.4.x# diff synoconfigs/apollolake .config
3c3
< # Linux/x86_64 4.4.59 Kernel Configuration
---
> # Linux/x86_64 4.4.59+ Kernel Configuration
5139c5139
< # CONFIG_KVM_AMD is not set
---
> CONFIG_KVM_AMD=m
```

Make alias:

```
alias dsm6make='make ARCH=x86_64 CROSS_COMPILE=/dsmsrc/x86_64-pc-linux-gnu/bin/x86_64-pc-linux-gnu-'
```

Comment unwanted check in virtext.h code (Xpenology write intel to boot_cpu_data.x86_vendor, and does'not want to launch svm virtualization with message like "has_svm: not amd"):

```
   84  vi arch/x86/include/asm/virtext.h
   
   /** Check if the CPU has SVM support
 *
 * You can use the 'msg' arg to get a message describing the problem,
 * if the function returns zero. Simply pass NULL if you are not interested
 * on the messages; gcc should take care of not generating code for
 * the messages on this case.
 */
static inline int cpu_has_svm(const char **msg)
{
        uint32_t eax, ebx, ecx, edx;

/**     if (boot_cpu_data.x86_vendor != X86_VENDOR_AMD) {
 *              if (msg)
 *                      *msg = "not amd";
 *              return 0;
 *      }
*/
```
   
**3.Make modules.**


kvm-amd.ko:
```
root@DiskStation:/dsmsrc/linux-4.4.x# dsm6make modules M=$(pwd)/arch/x86/kvm
```
Tun (and other network drivers):

```
root@DiskStation:/dsmsrc/linux-4.4.x# dsm6make modules M=$(pwd)/drivers/net
```

If you do something wrong - you can remove all prevously compiled files by commands:

kvm-amd.ko:

```
root@DiskStation:/dsmsrc/linux-4.4.x# make clean M=$(pwd)/arch/x86/kvm
  CLEAN   /dsmsrc/linux-4.4.x/arch/x86/kvm/.tmp_versions
  CLEAN   /dsmsrc/linux-4.4.x/arch/x86/kvm/Module.symvers
```
Tun.ko (and other network drivers):

```
root@DiskStation:/dsmsrc/linux-4.4.x# make clean M=$(pwd)/drivers/net
```

**4. Load modules**

Go to dom0 and execute insmod command in shell with sudo:

```
oleg@DiskStation:~$  sudo insmod /volume1/@appstore/debian-chroot/var/chroottarget/dsmsrc/linux-4.4.x/arch/x86/kvm/kvm-amd.ko
oleg@DiskStation:~$ sudo insmod /volume1/@appstore/debian-chroot/var/chroottarget/dsmsrc/linux-4.4.x/drivers/net/tun.ko
```

Check that modules was loaded property by "dmesg" command (you can launch this command without grep to see all related messages, check the time - when event happend )

amd-kvm:

```
oleg@DiskStation:~$ dmesg -T | grep "kvm"
[  100.876824] kvm: no hardware support
[ 1906.988772] kvm: Nested Virtualization enabled
[ 1906.988890] kvm: Nested Paging enabled
```

tun:

```
oleg@DiskStation:~$ dmesg -T | grep "tun:"

[ 1815.930575] tun: Universal TUN/TAP device driver, 1.6
[ 1815.930749] tun: (C) 1999-2004 Max Krasnyansky <maxk@qualcomm.com>
```

Check in loaded modules:

```
oleg@DiskStation:~$ lsmod | grep tun
tun                    19231  0 
ip6_udp_tunnel          1903  1 vxlan
udp_tunnel              2355  1 vxlan
tunnel4                 2261  1 sit
ip_tunnel              13200  1 sit
oleg@DiskStation:~$ lsmod | grep amd
kvm_amd                48571  0 
kvm                   318679  1 kvm_amd
```

**5. Generate, and copy ssh.pub key from your machine to Xpenology NAS**

Generate:

```
okkk@SWARM:~/.ssh$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/okkk/.ssh/id_rsa): synkey
....
okkk@SWARM:~/.ssh$ cat synkey.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQA=
```

Copy key to Synology station into root home dir:

```
oleg@DiskStation:~$ sudo su

ash-4.3# ls -la ~/.ssh/
total 12
drwxr-xr-x 2 root root 4096 Jun  1 13:14 .
drwx------ 6 root root 4096 Jun  1 13:14 ..
-rw-r--r-- 1 root root   33 Jun  1 13:14 authorized_keys

ash-4.3# cat ~/.ssh/authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQA=
```
*Warning* for external connection maybe you will need check libvirtd config in /etc/libvirt/libvirtd.conf


**6. Use openbsd syntax netcat (libvirtd required), and some packages and libs**


Symlink it from chroot:
```
oleg@DiskStation:~$ cd /bin/
oleg@DiskStation:/bin$ sudo ln -sf /volume1/@appstore/debian-chroot/var/chroottarget/bin/nc.openbsd .
oleg@DiskStation:/bin$ sudo ln -sf nc.openbsd nc
oleg@DiskStation:/bin$ sudo ln -sf /volume1/@appstore/debian-chroot/var/chroottarget/sbin/ebtables .

oleg@DiskStation:/bin$ cd /lib
oleg@DiskStation:/lib$ sudo ln -sf /volume1/@appstore/debian-chroot/var/chroottarget/lib/libbsd.so.0 .
oleg@DiskStation:/lib$ sudo ln -sf /volume1/@appstore/debian-chroot/var/chroottarget/lib/ebtables .
```

**7. Launch libvirtd and connect**


```
oleg@DiskStation:~$ sudo libvirtd
2020-06-01 10:17:12.581+0000: 21905: info : libvirt version: 1.2.17
2020-06-01 10:17:12.581+0000: 21905: error : dnsmasqCapsRefreshInternal:741 : Cannot check dnsmasq binary dnsmasq: No such file or directory
2020-06-01 10:17:12.646+0000: 21905: error : virFirewallValidateBackend:193 : direct firewall backend requested, but /sbin/ebtables is not available: No such file or directory
2020-06-01 10:17:12.646+0000: 21905: error : virFirewallApply:940 : internal error: Failed to initialize a valid firewall backend
2020-06-01 10:17:12.769+0000: 21905: error : virNodeSuspendSupportsTarget:332 : internal error: Cannot probe for supported suspend types
2020-06-01 10:17:12.770+0000: 21905: warning : virQEMUCapsInit:1036 : Failed to get host power management capabilities
2020-06-01 10:17:12.858+0000: 21905: error : virFirewallApply:940 : internal error: Failed to initialize a valid firewall backend
```

Connect from Virtual machine manager from your host, by link:
```
qemu+ssh://root@10.0.0.8/system
```

**8. Click on link and create new virtual machine**


Click on new connection and select - create new machine.

For ISO use link with path for your Xpenology for boot device, like:
```
/volume1/files/ubuntu-mate-18.04.1-desktop-amd64.iso
```

For storage create disk raw image (or, you cae create qcow2 image somewere, Xpenology doesnot have qemu-img tool frmo the box)

```
oleg@DiskStation:~$ dd if=/dev/zero of=/volume1/files/virtual_xpe/ubuntu18.04.raw  bs=1M count=20480

4096+0 records in
4096+0 records out
4294967296 bytes (4.3 GB) copied, 35.3655 s, 121 MB/s
```

Add /volume1/files/virtual_xpe as pool, select created raw image:

```
/volume1/files/virtual_xpe/ubuntu18.04.raw
```

**Warning!** Check device hdd type after create - device type need to be virtio. Ide controllers needs to be removed.

Specify additional options

Remove usb devices, use vnc graphic card.

Fix nic config - use openswitch for network, for nic use config (XML section) like ( see http://docs.openvswitch.org/en/latest/howto/libvirt/ ):

```
<interface type="bridge">
  <mac address="52:54:00:71:b1:b6"/>
  <source bridge="ovs_eth1"/>
  <virtualport type="openvswitch">
    <parameters interfaceid="742b4c03-cc91-4824-aa77-eff5f189cf9c"/>
  </virtualport>
  <target dev="vnet0"/>
  <model type="rtl8139"/>
  <alias name="net0"/>
  <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x0"/>
</interface>
```

Check that vnet0 was added to needed openswitch bridge: 

```
oleg@DiskStation:~$ sudo ovs-vsctl show
63b4b6f2-53ba-42e8-92a0-b7c84e3da6c9
    Bridge "ovs_eth1"
        Port "vnet0"
            Interface "vnet0"
        Port "eth1"
            Interface "eth1"
        Port "ovs_eth1"
            Interface "ovs_eth1"
                type: internal
    Bridge "ovs_eth0"
        Port "ovs_eth0"
            Interface "ovs_eth0"
                type: internal
        Port "eth0"
            Interface "eth0"
```

**9. Go to console to NAS, check qemu virtual machine:**

```
oleg@DiskStation:~$ ps ajfx | grep qemu
23514 24431 24430 23514 pts/16   24430 S+    1026   0:00  |           \_ grep --color=auto qemu
    1 23640 23638 23638 ?           -1 Sl       0   0:05 /usr/local/bin/qemu-system-x86_64 -name ubuntu18.04 -S -machine pc-q35-2.12,accel=kvm,usb=off,vmport=off -cpu Opteron_G5 -m 1024 -realtime mlock=off -smp 1,sockets=1,cores=1,threads=1 -uuid 1849696a-2f7b-48b8-affa-df947c048dd4 -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/ubuntu18.04.monitor,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=discard -no-hpet -no-shutdown -global PIIX4_PM.disable_s3=1 -global PIIX4_PM.disable_s4=1 -boot strict=on -device i82801b11-bridge,id=pci.1,bus=pcie.0,addr=0x1e -device pci-bridge,chassis_nr=2,id=pci.2,bus=pci.1,addr=0x1 -device ich9-usb-ehci1,id=usb,bus=pci.2,addr=0x2.0x7 -device ich9-usb-uhci1,masterbus=usb.0,firstport=0,bus=pci.2,multifunction=on,addr=0x2 -device ich9-usb-uhci2,masterbus=usb.0,firstport=2,bus=pci.2,addr=0x2.0x1 -device ich9-usb-uhci3,masterbus=usb.0,firstport=4,bus=pci.2,addr=0x2.0x2 -device virtio-serial-pci,id=virtio-serial0,bus=pci.2,addr=0x3 -drive file=/volume1/files/ubuntu-mate-18.04.1-desktop-amd64.iso,format=raw,if=none,media=cdrom,id=drive-sata0-0-0,readonly=on -device ide-cd,bus=ide.0,drive=drive-sata0-0-0,id=sata0-0-0,bootindex=1 -drive file=/volume1/files/virtual_xpe/ubuntu18.04.raw,format=raw,if=none,id=drive-virtio-disk0 -device virtio-blk-pci,scsi=off,bus=pci.2,addr=0x6,drive=drive-virtio-disk0,id=virtio-disk0 -netdev tap,fd=17,id=hostnet0 -device rtl8139,netdev=hostnet0,id=net0,mac=52:54:00:71:b1:b6,bus=pcie.0,addr=0x3 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 -chardev socket,id=charchannel0,path=/var/lib/libvirt/qemu/channel/target/ubuntu18.04.org.qemu.guest_agent.0,server,nowait -device virtserialport,bus=virtio-serial0.0,nr=1,chardev=charchannel0,id=channel0,name=org.qemu.guest_agent.0 -device usb-tablet,id=input0 -vnc 127.0.0.1:0 -device VGA,id=video0,vgamem_mb=16,bus=pcie.0,addr=0x1 -device ich9-intel-hda,id=sound0,bus=pci.2,addr=0x1 -device hda-duplex,id=sound0-codec0,bus=sound0.0,cad=0 -device virtio-balloon-pci,id=balloon0,bus=pci.2,addr=0x4 -msg timestamp=on
```

**10.Write some autostart scripts fo libvirtd and modules**

in progress.




That's all, folks!
