I'm not writing a full guide for this, but leaning heavily on pre-published guides on Vitis-AI.

https://xilinx.github.io/Vitis-AI/3.0/html/docs/quickstart/mpsoc.html

Here is what I would do:

Do the `Prerequisites` section

Do the `Quickstart` section. When you clone the Vitis-AI repo, make sure to checkout the `3.0` branch. You can also ignore the steps for setting up the target. Just do what you've been doing with setting up the IP and stuff.

Do the `PyTorch Tutorial` section with these modifications:

When you get to the Quantizing the Model section, and it has you run

```console
python resnet18_quant.py --quant_mode float --inspect --target DPUCZDX8G_ISA1_B4096 --model_dir model
```

Replace the thing after --target with what you noted down from when you ran `sudo show_dpu`. For example `DPUCZDX8G_ISA2_B512`

When you get to the `Compile the model` step, and it has you run

```console
vai_c_xir -x quantize_result/ResNet_int.xmodel -a /opt/vitis_ai/compiler/arch/DPUCZDX8G/<Target ex:KV260>/arch.json -o resnet18_pt -n resnet18_pt
```

We're going to change the file after the -a. Create your own file called arch.json with the contents:

```console
{"fingerprint":"0x101000056010400"}
```

where the fingerprint field has the fingerprint you saw when you ran `sudo show_dpu` on your FPGA. 

Another change is that a lot of the scp commands have you running them with `root@<ip address>` but this is no longer supported. So just use `petalinux@<ip address>` as before.

The image inferencing test should just work out of the box. One thing that was not entirely clear is that you need to run `sudo startx` and then `sudo ./test_video_classification <usb device> -t <threads>` for the USB camera. 

Here is me and a toilet plunger with resnet18
![Working Resnet](images/toilet.png)

Another just general piece of advice, just run everything with sudo. It'll save you some headeache if some commands aren't working.
