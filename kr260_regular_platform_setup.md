# KR260 Platform Setup

This guide serves as a starting point for setting up the hardware platform for the Kria KR260 Robotics Starter Kit

---

## Setup

Please use the tool versions specified in these guides to ensure maximum compatability. You might run into issues with different tool versions.

The various tools we will need to use ocassionally have issues being ran in your AFS workspace. To get around this, we do our work in the `/scratch` directory on the ECE machines.

Start by SSHing into your ECE machine 

``` console
ssh <username>@eceXXX.ece.local.cmu.edu
```

Navigate to the `/scratch` directory:

``` console
user@eceXXX:~$ cd /scratch
```

Create a workspace directory and set the permissions so only you can view the contents (or setup the permissions as you see fit).

``` console
user@eceXXX:/scratch$ mkdir ece500_kr260_workspace
user@eceXXX:/scratch$ chmod 700 ece500_kr260_workspace
```

Then source the Vivado setup file and start Vivado.
``` console
user@eceXXX:/scratch$ source /afs/ece/support/xilinx/xilinx.release/Vivado-2022.2/Vivado/2022.2/settings64.sh
user@eceXXX:/scratch$ vivado &
```

## Create Vivado Project

With Vivado 2022.2 open, under the `Quick Start` menu, click on `Create Project`

![Vivado Menu Page](images/vivado_menu_page.png)

Hit `Next` and go to the `Project Name` tab. Set a name for the project, specify the workspace created in the [Setup](##Setup) section, and leave the `Create Project Subdirectory` option checked.

![Vivado Project Name](images/vivado_project_name.png)

Hit `Next` and go to the `Project Type` tab. Check the `Project is an extensible Vitis platform`.

![Vivado Project Type](images/vivado_project_type.png)

Hit `Next` and go to the `Default Part` tab. Click the `Boards` header, and search for `KR260`. Then click on the `Connections` link that shows up under the `Kria KR260 Robotics Start Kit SOM`.

If no matches for the KR260 are found, click on the `Refresh` button near the bottom left corner of the window, and search again.

![Vivado Default Part](images/vivado_default_part.png)

Populate the connections as shown below:

![Vivado Manage Board Connections](images/vivado_manage_board_connections.png)

Hit `Okay` then `Next` and you should see this

![Vivado New Project Summary](images/vivado_new_project_summary.png)

Then hit `Finish`. You should now see this screen:

![Vivado Project Home](images/vivado_project_home.png)

## HW Block Diagram

Click on the `Create Block Design` button. Make sure to save your block design ofetn. If you need to exit Vivado, you can come back to your block design by clicking the `Open Block Design` button below the `Create Block Design` button.

![Vivado CBD Button](images/vivado_cbd_button.png)

In the popup, give it a design name. I named mine `Design name: kr260_bd` Leave the other fields to their default, `Directory: <Local to Project>` and `Specify source set: Design Sources`. 

In the `Diagram` window, click on the plus sign, and add the `Zynq Ultrascale+ MPSoC` component. A green banner will popup, click on the link that says `Run Block Automation`. Ensure that the option for `Apply Board Preset` is checked and hit `Ok`. 

You should expect to see something like this:

![BD Step 1](images/bd_step1.png)

Double click on the Zync UltraScale+ MPSoC IP block, then navigate to the `PS-PL Configuration` tab. Uncheck the two full power master AXI interfaces and only enable the low power master AXI interface. `PS` means `Processing System` which refers to the ARM core, while `PL` means `Programmable Logic` which refers to the FPGA portion of the die. 

![BD Step 2a](images/bd_step2a.png)

Now add a `Clocking Wizard` IP component. Double click it and navigate to the `Output Clocks` tab. Enable `clk_out1`, `clk_out2`, and `clk_out3` and set the frequencies to `100 MHz`, `200 MHz`, and `400 MHz`, respectively. 

![BD Step 3](images/bd_step3.png)

Scroll down, to the bottom of the `Output Clocks` tab and set the `Reset Type` to `Active Low` (recall the pl_reset*n*0) on the MPSoC.  

![BD Step 4](images/bd_step4.png)

Connect the `Clocking Wizard` to the `Zynq UltraScale+ MPSoC` by connecting `resetn` to `pl_resetn0` and `clk_in1` to `pl_clk0`. It should look like this:

![BD Step 5](images/bd_step5.png)

You can click on the `Optimize Routing` button to make the routing look a bit cleaner.

![BD Step 6](images/bd_step6.png)

For each of the clocks we've added, we need to add a `Processor System Reset`. Connect the `clk_outX` to each of the `slowest_sync_clk` of the reset blocks. Connect the `ext_reset_in` to the `pl_resetn0` from the MPSoC. Finally, connect the each of the `dcm_locked` to the `locked` of the `Clocking Wizard`. 

When you're done, it should look like this. 

![BD Step 7](images/bd_step7.png)

Next, we need to add an interrupt controller. We add an `AXI Interrupt Controller`. Update the `Interrupt Output Connection` to `Single` (instead of `Bus`). 

![BD Step 8](images/bd_step8.png)

In the green banner that pops up, click on `Run Connection Automation`. Set all of the `Clock Source for --` to the 200MHz clock created before. Double click on the `AXI Interconnect` that gets generated and take a look at its settings. Nothing has to be changed here. 

Finally, connect the `irq` of the Interrupt Controller to the `pl_ps_irq0[0:0]` pin of the MPSoC. When done, it will look like this.

![BD Step 9](images/bd_step9.png)

Now is a good time to save your design if you haven't done so already!

Navigate to the `Platform Setup` tab, then the `AXI Port` tab, and enable all of the full power AXI interfaces available on the MPSoC. In addition, set the `SP Tag` to some short string for each of the Slave AXI ports. I've just went with the name of the AXI port.

In addition, under the `AXI Interconnect` section, enable ports `M01_AXI` to `M08_AXI`. 

![BD Step 10](images/bd_step10.png)
![BD Step 11](images/bd_step11.png)

Then go to the `Clock` tab, enable the 3 clocks from the `Clocking Wizard`. Set the IDs of the clock `0`, `1`, and `2`. Set the default clock to 200MHz.

![BD Step 12](images/bd_step12.png)

Then, in the `Interrupt` tab, enable `intr` under the `AXI Interrupt Controller` section.

![BD Step 13](images/bd_step13.png)

Double click again on the MPSoC. Then navigate to the `I/O Configuration` tab and verify that the `Low Speed/Memory Interfaces/QSPI` settings look like below.

![BD Step 2b](images/bd_step2b.png)

Likewise for `Low Speed/Memory Interfaces/I2C`. `MIO` pins are pins that are directly connected from the `PS` to the package IO, while EMIO pins go between the `PS` and `PL`.  

![BD Step 2c](images/bd_step2c.png)

Verify these settings for the `PMU`(Platform Management Unit)

![BD Step 2d](images/bd_step2d.png)

For the sake of brevity, I'm going to just list out the rest of the settings.
- Under Low Speed
    - Under I/O Peripherals
        - Verify SPI 1 is enabled on MIO 6-11
        - Verify UART 1 is enabled on MIO 36-37
        - Verify GPIO 0 is enabled on MIO 0-25
        - Verify GPIO 1 is enabled on MIO 26-51
    - Under Processing Unit
        - Verify SWDT 0 is enabled
        - Verify SWDT 1 is enabled
        - Verify all 4 TTC0-3 is enabled
        - Connect TTC0 Waveout to EMIO (this will be used for fan PWM control)
- Under High Speed
    - Under GEM (these are the RJ45 connectors on the carrier board)
        - Verify GEM 0 is enabled on GT Lane 0
        - Verify GEM 1 is enabled on MIO 38-49 with its MDIO on 50-51
    - Under USB
        - Verify USB 0 is enabled with USB 0 set to MIO 52-63 and USB 3.0 to GT Lane2
        - Verify USB 1 is enabled with USB 1 set to MIO 64-75 and USB 3.0 to GT Lane3
        - Verify USB 0 and USB 1 reset is set to `Active Low` connected to MIO 76 and MIO 77, respectively
    - Under Display Port
        - Verify Display Port is enabled
        - Verify DPAUX is connected to MIO 27-30
        - Verify Lane Selection is Single Lower

Finally, navigate back to `PS-PL Configuration/General/Fabric Reset Enable/Number of Fabric Resets` and set this from 1 to 4. 

Exit out of the configuration. In the main block design view, click on the `Validate Design` button, after a few seconds, the design will be validated. It is expected to receive a critical warning about the `intr` pin, however this is something that will be automagically handled by the v++ linker later. 

![BD Step 14](images/bd_step14.png)

Finally, we have one more thing to add for the fan control. The output of the `TTC0` is 3 bits, we only need one of those bits to control the fan pwm. We use the MSB of the output. 

Add a `Slice` IP block and configure it so that:
- Din Width is 3
- Din From is 2
- Din Down To is 2
- Dout Width is 1

Connect the `Din` of the `Slice` to the `emio_ttc0` output on the MPSoC. 

It should look like this when you're done:

![BD Step 15](images/bd_step15.png)

Hit `Ctrl+K` (or under the `Design` tab, right-click on the top level and hit `Create Port`). Set the `Port name` as `fan_pwm_en` and set the `Direction` as `Output`.

![BD Step 16](images/bd_step16.png)

Finally, wire the port to the output of the slice.

![BD Step 17](images/bd_step17.png)

Validate the block design again and you can safely ignore the `intr` critical warning as mentioned before. Make sure to also save your block design.

Open up the `Settings` menu. This is in the same menu as the `Create Block Design` mentioend before, but a few items above. Change the `Bitstream` settings so that a `bin` file is generated.

![BD Step 18](images/bd_step18.png)

In the `Flow Navigator` menu, click on `Generate Block Design`. This is just a few items below the `Create Block Design` mentioned at the start. Set the `Synthesis Option` to `Global` and hit `Generate`. Ignore the `intr` warning again.

Then `Create HDL Wrapper` and choose the menu item that says `Let Vivado manage wrapper and auto-update`.

![BD Step 19](images/bd_step19.png)

Skim through the generated `*.v` and `.vhd` files in the `Design Sources` list. Note that `Vivado` supports mixing both VHDL and SystemVerilog/Verilog.

Finally, we need to setup the constraint for the fan. We've created a port in our block design for the fan PWM, but now we need to map that to a physical IO pin on the FPGA.

In the `Flow Navigator`, click `Add sources` and `Add or create constraints`. Click `Create Constraint` and add a file called `fan_pinout.xdc`. Make sure to check `Copy constraint files into project`.

![BD Step 20](images/bd_step20.png)

In the sources tab, navigate to the added constraint file and paste this in:

``` tcl
set_property BITSTREAM.GENERAL.COMPRESS TRUE [current_design]

set_property PACKAGE_PIN <X00> [get_ports {fan_pwm_en}]
set_property IOSTANDARD LVCMOS33 [get_ports {fan_pwm_en}]
set_property SLEW SLOW [get_ports {fan_pwm_en}]
set_property DRIVE 4 [get_ports {fan_pwm_en}]
```

**Please note that the `PACKAGE_PIN` is not set to a real value. This is left as an exercise to the reader.**

Here are some resources:
- Schematics for the KR260 SOM can be found here: [Schematics](https://www.xilinx.com/member/forms/download/design-license.html?cid=bad0ada6-9a32-427e-a793-c68fed567427&filename=xtp743-kr260-schematic.zip)
- XML for the FPGA pinout can be found `/afs/ece/support/xilinx/xilinx.release/Vivado-2022.2/Vivado/2022.2/data/xhub/boards/XilinxBoardStore/boards/Xilinx/kr260_som/1.1/part0_pins.xml`

<details>
    <summary>Answer</summary>
    Set the PACKAGE_PIN to A12

    From the schematic on page 3, we can see that the Fan PWM is controlled by HDA20.

    HDA20 is connected to C24 on page 6.

    We can see that C24 is mapped to A12 on the SOM from the XML: "./part0_pins.xml:56:            <pin index="39" name ="som240_1_c24"   loc="A12"  pcb_min_delay="0.33601" pcb_max_delay="0.41067"/>"
</details>
<br>

Now, can generate a bitstream. Click on `Generate Bitstream` in the `Flow Navigator`. A warning saying no implementation is found should pop up. This is just saying you haven't ran Synthesis and PnR yet. Hit Yes, and run the bitstream generation.

This will take a while! You can monitor the progress by clicking on the `Design Runs` tab in the bottom ribbon. View the implementation when it's done. Disregard the `Critical Warning` regarding the clock source pin.

Finally, in the `Flow Navigator` click on `Export Platform`. Select `Hardware and Hardware Emulation` in the `Platform Type`. Make sure to check the `Include Bitstream` option in the `Platform State` window. Give the platform whatever name you'd like in the two tabs, just make note of where the `.xsa` file ends up.

## Wrap Up

Let's save our project onto our AFS as the `/scratch` directory gets cleared after 28 days. 

In the `File` item in the top ribbon, navigate to `Project -> Write TCL`. 

![Wrap Up](images/wrap_up.png)

Check the relevant options, I would do:
- Write all properties
- Copy sources to new project
- Recreate block design using TCL
- Write object values

Then save the `.tcl` file to somwhere in your `private` directory. 

In the future, if you'd like to open up this project, run

```console
user@eceXXX:/scratch$ vivado -source <tcl file> &
```
A reference of this platform can be found at 
```console
/afs/ece.cmu.edu/class/ece500/fpga/kr260_hw_platform.tcl
```