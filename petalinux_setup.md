# KR260 PetaLinux Setup

This guide will walk you through updating the firmware of your Kria KR260 board and how to create a PetaLinux image on which you can run programs.

---
## Prerequisites

#### PetaLinux Installation

Download the PetaLinux 2022.2 installer from the [Xilinx Download Archives](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/archive.html)

SCP the file to your scratch workspace on the ECE machine you used to do the Vivado setup.

```console
[Local] $ scp petalinux-v2022.2-<date>-installer.run <user>@eceXXX.ece.local.cmu.edu:/scratch/<workspace>
```

```console
user@eceXXX:/scratch/<workspace>$ mkdir PetaLinux
user@eceXXX:/scratch/<workspace>$ chmod 755 ./petalinux-v2022.2-<date>-installer.run
user@eceXXX:/scratch/<workspace>$ ./petalinux-v2022.2-<date>-installer.run --dir PetaLinux
```

Accept the EULAs. The install should take a bit of time.

#### PetaLinux Upgrade

We (might) need to apply an update to the SDK that came with PetaLinux. Start by sourcing the PetaLinux environment, and then applying the upgrade

```console
user@eceXXX:/scratch/<workspace>$ source PetaLinux/2022.2/settings.sh
user@eceXXX:/scratch/<workspace>$ petalinux-upgrade -u http://petalinux.xilinx.com/sswreleases/rel-v2022/sdkupdate/2022.2_update1/ -p "aarch64" --wget-args "--wait 1 -nH --cut-dirs=4"
```

#### BSP Download

Next, you'll need to download the starter BSP (Board Support Package). Head back over to the [Xilinx Download Archives](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools/archive.html) and grab the Kria KR260 Starter Kit BSP.

Like before, you'll need to SCP this over to your scratch workspace. 

#### Ubuntu 22.04

Download the [Xilinx KR260 Ubuntu 22.04 image](https://ubuntu.com/certified/202202-29985).

You'll need a MicroSD card (minimum of 16GB) to flash Linux images to for the KR260. Use your favorite imager (I personally use Balena Etcher) and flash the downloaded image to your MicroSD card.

#### Firmware Download

Head over to [Kria SOM Boot Firmware Update](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/3020685316/Kria+SOM+Boot+Firmware+Update) and download the K26 Boot FW 1.02.

## Firmware Update

I've only done these steps with a Windows computer, YMMV on MacOS. 

Connections:
- Display Port cable from the KR260 to a compatible monitor
- Ethernet cable either to your wireless router or your computer. I will be connecting my KR260 directly to my Windows computer. Take note to connect the cable to either of the two RJ45 ports on the right side of the KR260. Those two are the ones that go to the ARM Core 
- Insert flashed MicroSD card to the KR260
- Connect Keyboard (and Mouse, you can get away without a mouse if you're good wtih keyboard shortcuts)
- Power on the KR260

Once booted up, you'll be required to set a password.

On your host computer, determine your IP address.

```console
C:\Users\varun>ipconfig

Windows IP Configuration


Ethernet adapter vEthernet (WSL):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::f026:a68b:be41:87ac%64
   IPv4 Address. . . . . . . . . . . : 172.23.144.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::869b:ba71:94ac:ec29%10
   Autoconfiguration IPv4 Address. . : 169.254.90.229
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . :

Ethernet adapter Bluetooth Network Connection:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . :
```

We specifically care about our IPv4 address for our ethernet adapter. Mine is `169.254.90.229`. We also need to know our subnet mask which is `255.255.0.0`.

Then we need to determine the IP address of the KR260.

```console
ubuntu@kria:~$ ifconfig
eth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 00:0a:35:16:cf:22  txqueuelen 1000  (Ethernet)
        RX packets 14  bytes 2250 (2.2 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 14  bytes 2308 (2.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 38  

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::5be3:5a87:f2d4:8c0d  prefixlen 64  scopeid 0x20<link>
        ether 00:0a:35:15:c8:8b  txqueuelen 1000  (Ethernet)
        RX packets 570  bytes 66473 (66.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 161  bytes 20970 (20.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 37  

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1887  bytes 138040 (138.0 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1887  bytes 138040 (138.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

My KR260 is directly connected to my Windows machine, so no IP assignment happens automatically. Take note of which of ethernet interfaces has the `RUNNING` flag set. In my case, it is eth1. 

Eth1 should be the top right RJ45 connector and Eth0 being the bottom right connector.

We need to now set the IPv4 address of the KR260. Since the subnet mask is `255.255.0.0`, we know that we need to start the IP address with `169.254` and we're free to set the last two octets to what we want.

```console
ubuntu@kria:~$ sudo ifconfig eth1 169.254.100.100 netmask 255.255.0.0
ubuntu@kria:~$ ifconfig
eth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 00:0a:35:16:cf:22  txqueuelen 1000  (Ethernet)
        RX packets 14  bytes 2250 (2.2 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 14  bytes 2308 (2.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 38  

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 169.254.100.100  netmask 255.255.0.0  broadcast 169.254.255.255
        inet6 fe80::5be3:5a87:f2d4:8c0d  prefixlen 64  scopeid 0x20<link>
        ether 00:0a:35:15:c8:8b  txqueuelen 1000  (Ethernet)
        RX packets 938  bytes 111651 (111.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 236  bytes 35871 (35.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 37  

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 4998  bytes 359593 (359.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4998  bytes 359593 (359.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Now we copy over the downloaded firmware file over to the KR260.

```console
[Host] $ scp K26-BootFW-01.02-06140626.bin ubuntu@169.254.100.100:
```
Then on the Kria:

```console
ubuntu@kria:~$ sudo xmutil bootfw_update -i K26-BootFW-01.02-06140626.bin
ubuntu@kria:~$ sudo xmutil bootfw_status
ubuntu@kria:~$ sudo shutdown -h now
```

Power cycle the Kria, and on the next boot. 

```console
ubuntu@kria:~$ sudo xmutil bootfw_update -v
```

This is necessary to validate the board can be powered on with the new firmware. If this is not completed, the board will fallback to the default firmware.

## PetaLinux Setup

On your ECE machine, head over to your scratch workspace.

```console
user@eceXXX:/scratch/<workspace>$ source PetaLinux/2022.2/settings.sh
user@eceXXX:/scratch/<workspace>$ petalinux-create --type project -s <.bsp> --name kr260_petalinux_os
```

Make note of where the HW platform `.xsa` is located that was created in the Vivado setup.

```console
user@eceXXX:/scratch/<workspace>$ find . -name "*.xsa"
./kr260_hw_platform/kr260_bd_wrapper.xsa <- We want this one
./kr260_petalinux_os/components/plnx_workspace/device-tree/device-tree/hardware_description.xsa
./kr260_petalinux_os/hardware/xilinx-kr260-starterkit-2022.2/kr260_starter_kit.xsa
./kr260_petalinux_os/project-spec/hw-description/system.xsa
```

Let's start configuring the PetaLinux image
```console
user@eceXXX:/scratch/<workspace>$ cd kr260_petalinux_os
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-config --get-hw-description=<.xsa>
```

A menu window will now pop up. Use the arrow keys to navigate around. Enter moves you down a level, \<Esc\> \<Esc\> will move you up a directory. Use `y` and `n` to enable or disable config items.

Let's start by setting up some basic config items
```
FPGA Manager ---> Fpga Manager [*] (enable)

Image Packaging Configuration ---> Root Filesystem Type ---> INITRD [*] (enable)
Image Packaging Configuration ---> INITRAMFS/INITRD Image name ---> petalinux-initramfs-image
Image Packaging Configuration ---> Copy final images to tftpboot [ ] (disable)
```

Exit out of the config menu. You can do this by hitting \<Esc\> \<Esc\> in the top level menu. Hit `Yes` to save the new configuration.

Now attempt to build the image with
```console
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-build
```

Now we will setup the root packages you will need. Feel free to add anything else you find in the config menu. These items are organized in in the order you find them in the config menu (i.e. you should work your way down the menu).

```console
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-config -c rootfs

Filesystem Packages ---> base ---> dnf ---> dnf [*]
Filesystem Packages ---> console ---> utils ---> git ---> git [*]
Filesystem Packages ---> console ---> utils ---> grep ---> grep [*]
Filesystem Packages ---> console ---> utils ---> vim ---> vim [*]
Filesystem Packages ---> libs ---> xrt ---> xrt [*]
Filesystem Packages ---> libs ---> xrt ---> xrt-dev [*]
Filesystem Packages ---> libs ---> zocl ---> zocl [*]
Filesystem Packages ---> libs ---> opencl-clhpp ---> opencl-clhpp-dev [*]
Filesystem Packages ---> libs ---> opencl-headers ---> opencl-headers [*]
Filesystem Packages ---> misc ---> package-group-core-buildessential [*]
Filesystem Packages ---> x11 ---> base ---> libdrm ---> libdrm [*]
Filesystem Packages ---> x11 ---> base ---> libdrm ---> libdrm-tests [*]
Filesystem Packages ---> x11 ---> base ---> libdrm ---> libdrm-kms [*]
Petalinux Package Groups ---> packagegroup-petalinux ---> packagegroup-petalinux [*]
Petalinux Package Groups ---> packagegroup-petalinux-gstreamer ---> packagegroup-petalinux-gstreamer [*]
Petalinux Package Groups ---> packagegroup-petalinux-opencv ---> packagegroup-petalinux-opencv [*]
Petalinux Package Groups ---> packagegroup-petalinux-v4lutils ---> packagegroup-petalinux-v4lutils [*]
Petalinux Package Groups ---> packagegroup-petalinux-x11 ---> packagegroup-petalinux-x11 [*]
```

Finally, we need to now build everything. 

```console
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-build
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-build --sdk
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-package --boot --u-boot --force
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-package --wic --images-dir images/linux/ --bootfiles "ramdisk.cpio.gz.u-boot,boot.scr,Image,system.dtb,system-zynqmp-sck-kr-g-revB.dtb" --disk-name "sda"
```

Now copy to your local host the `petalinux-sdimage.wic` file that gets generated.

```console
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ [ -f ./images/linux/petalinux-sdimage.wic ] && echo "./images/linux/petalinux-sdimage.wic"
```

Now, flash the `.wic` to a MicroSD card using your method of choice. Go ahead and boot up your KR260 with the MicroSD card. The user is `petalinux` for reference.


