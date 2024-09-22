This guide is mostly a copy-paste of the previous petalinux guide with additional interjections on what's necessary. Please make sure that the `Prerequisites` section from petalinux_setup.md are run first.

Head over to: https://github.com/Xilinx/Vitis-AI/tree/3.0/dpu and download the reference design for the Kria K26.

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

Now we will configure the kernel with the necessary package:
```console
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-config -c kernel

Device Drivers ---> Misc devices ---> Xilinux Deep learning Processing Unit (DPU) Driver [*] (enable)
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
Filesystem Packages ---> misc ---> xauth [*]
Filesystem Packages ---> x11 ---> base ---> libdrm ---> libdrm [*]
Filesystem Packages ---> x11 ---> base ---> libdrm ---> libdrm-tests [*]
Filesystem Packages ---> x11 ---> base ---> libdrm ---> libdrm-kms [*]
Petalinux Package Groups ---> packagegroup-petalinux ---> packagegroup-petalinux [*]
Petalinux Package Groups ---> packagegroup-petalinux-gstreamer ---> packagegroup-petalinux-gstreamer [*]
Petalinux Package Groups ---> packagegroup-petalinux-opencv ---> packagegroup-petalinux-opencv [*]
Petalinux Package Groups ---> packagegroup-petalinux-v4lutils ---> packagegroup-petalinux-v4lutils [*]
Petalinux Package Groups ---> packagegroup-petalinux-x11 ---> packagegroup-petalinux-x11 [*]
```

We want to copy: recipes-apps, recipes-vitis-ai, and then copy/merge recipes-kernel from DPUCZDX8G_VAI_v3.0/prj/Vivado/sw/meta-vitis/ into <petalinux-os>/project-spec/meta-user

Update `/project-spec/meta-user/conf/petalinuxbsp.conf` with,
```console
IMAGE_INSTALL:append = " vitis-ai-library "
IMAGE_INSTALL:append = " vitis-ai-library-dev "
```

Update `/project-spec/meta-user/conf/user-rootfs.conf` with,

```console
CONFIG_vitis-ai-library
CONFIG_vitis-ai-library-dev
CONFIG_vitis-ai-library-dbg
CONFIG_dnf
CONFIG_nfs-utils
```

Finally, we need to now build everything. 

```console
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-build
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-build --sdk
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-package --boot --u-boot --force
user@eceXXX:/scratch/<workspace>/kr260_petalinux_os$ petalinux-package --wic --images-dir images/linux/ --bootfiles "ramdisk.cpio.gz.u-boot,boot.scr,Image,system.dtb,system-zynqmp-sck-kr-g-revB.dtb" --disk-name "sda"
```

Now copy to your local host the `petalinux-sdimage.wic` file that gets generated.

Now, flash the `.wic` to a MicroSD card using your method of choice. Go ahead and boot up your KR260 with the MicroSD card. The user is `petalinux` for reference.

We'll need to also build the device tree overlay like in the vitis_example_add.md tutorial.

```console
user@eceXXX:/scratch/<workspace>/kr260_vitis_platform$ xsct

xsct%: hsi::open_hw_design <path to platform xsa>
xsct%: createdts -hw <path to platform xsa> -zocl -platform-name kria_kr260 -git-branch xlnx_rel_v2022.2 -overlay -compile -out ./dtg_output
xsct% exit

user@eceXXX:/scratch/<workspace>/kr260_vitis_platform$ cd dtg_output/dtg_output/kria_kr260/psu_cortexa53_0/device_tree_domain/bsp
user@eceXXX:/scratch/<workspace>/kr260_vitis_platform/dtg_output/dtg_output/kria_kr260/psu_cortexa53_0/device_tree_domain/bsp$ dtc -@ -O dtb -o pl.dtbo pl.dtsi
```

We'll need three more things. In your petalinux folder, search for a `*.bit.bin` file. It should be `<block diagram name>_wrapper_bit.bin`. Copy that to your host. And like before, we'll need the pl.dtbo file, and then, create a file called `shell.json` with the contents
```
{
  "shell_type" : "XRT_FLAT",
  "num_slots": "1"
}
```


On your FPGA, create a folder called kr260_dpu in /lib/firmware/xilinx/. Copy over the three files mentioned above into that folder.

```console
sudo xmutil listapps
sudo xmutil unloadapp
sudo xmutil loadapp kr260_dpu
```

Finally, run 
```console
sudo show_dpu
```

```
"DPU Arch":"DPUCZDX8G_ISAX_BXXX_XXXXXXXXXXXXXXXX"
```

Make note of the arch/fingerprint it prints out, as we'll need this to compile models for our DPU configuration. We'll need the `DPUCZDX8G_ISAX_BXXX` and `XXXXXXXXXXXXXXXX` in the future. 
