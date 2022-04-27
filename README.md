# Booting a Zephyr binary from SD card on the Zedboard 

This tutorial explains how to set up and build a system development project for
the Zynq-7000 SoC on the Zedboard. These steps will result in the creation of
a matching FPGA bitstream and First Stage Boot Loader (FSBL), which performs
the I/O and clocking configuration of the SoC and loads the bitstream into the
FPGA Programmable Logic upon power-up.

Afterwards, u-boot will be built as the 2nd stage boot loader, which will
run after the FSBL has completed. U-boot will load and execute the Zephyr binary
(ELF format) stored on the SD card alongside the FSBL and u-boot.

## Prerequisites

A FAT-formatted SD card is required as boot media.

Download and install the following tools:
- [Xilinx Vivado ML Edition](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools.html)
- [Xilinx Vitis Unified Software Platform](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vitis.html)

Clone the following git repository:

- [u-boot-xlnx](https://github.com/Xilinx/u-boot-xlnx)

All of the following instructions and screenshots are based on Vivado and Vitis
release 2021.1. Use of Linux as host operating system is suggested.

## Step 1: Vivado: from system design to FPGA bitstream

### Setting up the Vivado project

*********************************

Start Vivado. On the startup screen, click 'Create Project'.

![Vivado startup: 'Create Project'](/img/01_create_vivado_project.png "Vivado startup: 'Create Project'")

Choose a project name and a project folder within Vivado's workspace directory.

![Vivado project name and folder](/img/02_project_name_path.png "Vivado project name and folder")

When asked to configure the project type, select 'RTL Project' and make sure that the 'Do not specify sources
at this time' checkbox is checked.

![Vivado project type](/img/03_project_type.png "Vivado project type")

The next dialog asks the user to specify the part identifier of the target SoC, or to alternatively
specify a pre-defined development board. A board definition for the Zedboard is available, which might
have to be downloaded into your local Vivado installation first. In the 'Default Part' dialog, perform
the following steps in order to select the Zedboard board definition:

* Select the 'Boards' tab
* Click the 'Refresh' button below the target boards list
* Search the list entry 'ZedBoard Zynq Evaluation and Development Kit'
* If the board definition is not yet downloaded, click the download symbol in the staus column
  of the 'ZedBoard' list entry
* Select the 'ZedBoard' list entry and click 'Next >'

![Vivado target board selection](/img/04_board_selection.png "Vivado target board selection")

Once the board selection has been confirmed, a summary of the project to be created will be displayed.
Click the 'Finish' button in order to create the new project. Vivado will then transition to the
'Project Manager' view.

### Setting up the block design

*******************************

In the 'Flow Navigator' menu, click 'Create Block Design'.

![Vivado Project Manager: Create Block Design](/img/05_create_block_design.png "Vivado Project Manager: Create Block Design")

You will be prompted to specify a name for the block design. The default value is 'design_1'.
Leave the other settings at their default values ('Local to Project', 'Design Sources').
Click 'OK', Vivado will then transition to the 'Block Design' view.

![Vivado Project Manager: Create Block Design parameters](/img/06_create_bd_parameters.png "Vivado Project Manager: Create Block Design parameters")

In the 'Diagram' pane, click the '+' symbol in the upper bar of icons, in the IP core
selection list which will then appear, select and double-click 'ZYNQ7 Processing System'.

![Vivado Block Design: add ZYNQ7 Processing System](/img/07_add_ps7.png "Vivado Block Design: add ZYNQ7 Processing System")

The Processor System block will appear in the 'Diagram' pane, below the icon bar, a green bar
offering block automation will Appear. Click 'Run Block Automation'.

![Vivado Block Design: Run Block Automation](/img/08_block_automation.png "Vivado Block Design: Run Block Automation")

Leave all settings in the 'Run Block Automation' dialog at their default values and click 'OK'.
Once the Block Automation process has completed, the 'processing_system7_0' block will have a
basic configuration and two additional ports, 'DDR' and 'FIXED_IO' will be configured in a way
matching the Zedboard.

![Vivado Block Design: PS7 post Block Automation](/img/09_ps7_post_automation.png "Vivado Block Design: PS7 post Block Automation")

Click the '+' icon once again, from the selection list, select the entry 'Processor System Reset'.
Once the block has been added to the block diagram, automation will once again be offered. In the
green bar below the menu icons, click 'Run Connection Automation'.

![Vivado Block Design: Processor System Reset Block Automation](/img/10_ps_reset_automation.png "Vivado Block Design: Processor System Reset Block Automation")

In the 'Run Connection Automation' dialog, check the 'All Automation' checkbox so that all entries
in the tree view are checked.

![Vivado Block Design: Processor System Reset Block Automation](/img/11_ps_reset_automation_check.png "Vivado Block Design: Processor System Reset Block Automation")

The block design now contains the 'Processor System Reset' and 'ZYNQ7 Processing System' blocks,
connected to each other. At any time, the layout of the block design can be optimized by clicking
on the 'Regenerate Layout' icon in the icon bar of the 'Diagram' pane.

![Vivado Block Design: Regenerate Layout](/img/12_regenerate_layout.png "Vivado Block Design: Regenerate Layout")

### Modifying the I/O mapping and clocking configuration

********************************************************

By now, the clocking and I/O configuration as well as the scope of the peripherals to be
accessible to Zephyr can be checked and modified if desired.

If you skip this step and continue with the default configuration, the Zedboard will eventually
run Zephyr with the following configuration:

- CPU clock frequency: 666.6 MHz, driven by the ARM PLL
- DDR3 RAM clock frequency: 533.3 MHz, driven by the DDR PLL

The following (supported) peripherals will be enabled, and the clock speeds to be specified in the
device tree are:

- UART1 for console and shell, 50 MHz driven by the IO PLL (Baud rate generator will apply pre-scalers
  in order to derive the specified baud rate from the input clock). The default baud rate in the
  Zedboard's device tree is 115200.
- GEM0 Ethernet, 1000 MHz driven by the IO PLL (depending on the link speed reported by the attached
  Ethernet PHY, pre-scalers will be applied in order to derive the matching TX clock for 10/100/1000
  MBit/s - 2.5, 25 or 125 MHz respectively).
- The ARM Architected Timer is always driven by the clock signal CPU_3x2x, which always has half the
  CPU's frequency, therefore, the Kconfig parameter SYS_CLOCK_HW_CYCLES_PER_SEC is set to 333.3 MHz
  by default in the Zedboard's Kconfig.defconfig file.
- The internal GPIO controller (PS GPIO) will be enabled, and the two pushbuttons BTN8 and BNT9 as well
  as the LED LD9 will be accessible. LD9 has been giving a matching alias so that the 'blinky' demo
  will work.

The unmodified zedboard.dts reflects exactly this default configuration.

If you want to change the clocking configuration, double-click the 'ZYNQ7 Processor System' block.
The 'Re-customize IP' dialog will open. Click 'Clock Configuration' in the 'Page Navigator'
column. There are two ways to modify the configuration:

1. Stay in the 'Basic Clocking' tab. For the enabled peripherals, the entry in the column 'Requested
   Frequency' can be modified. Vivado will re-calculate the dividers applied to the clock frequency
   of the PLL that drives the respective peripheral, attempting to match the requested clock frequency
   as best as possible. Make sure that the requested and actual frequencies eventually match.
   Notice that:
   - the UARTs are not present in this view.
   - for the two Ethernet controllers, the entry in the 'Requested Frequency' column is actually
     a drop-down box containing the supported link speeds, which map to the three TX clock frequencies
     mentioned above. **As the driver supports re-adjustment of the TX clock pre-scalers at run-time
     if the attached Ethernet PHY reports a change in link speed, the clock-frequency parameter in
     the device tree must instead specify the clock frequency of the PLL from which the TX clock is
     derived.**

![Vivado Re-Customize IP: Basic Clocking](/img/13_basic_clocking.png "Vivado Re-Customize IP: Basic Clocking")

2. Activate the 'Advanced Clocking' tab. If not for modifying the clocking configuration, this
   tab is useful for obtaining the clock frequency of the IO PLL (or whatever PLL is chosen to drive
   the Ethernet controllers) and the UARTs. In order to modify the clocking configuration, check
   the 'Override Clocks' checkbox. Modify the prescaler(s) values for each peripheral for which you
   wish to adjust the clock frequency. Observe the range of valid frequencies in the 'Range (MHz)'
   column. Notice that:
   - the ARM Architected Timer, used as the system timer for Zephyr, is absent from this view as it
     can not be parameterized. Its clock frequency is always fCPU / 2.
   - Certain peripherals of which multiple instances exist (e.g. UART, CAN) do not allow a per-instance
     configuration, unlike, for example, the Ethernet controllers. The same clock frequency is
     supplied to all instances of the respective peripheral. All device tree nodes referring to
     those instances must specify the same value for the 'clock-frequency' property.
   - By default, one clock signal, FCLK_CLK0 is supplied to the FPGA Programmable Logic (PL).
     IP cores / HDL code can be associated with this clock. Three more clocks for the PL,
     FCLK_CLK1 to FCLK_CLK3 can be enabled and configured in this view. In order to do so, expand
     the 'PL Fabric Clocks' category.

![Vivado Re-Customize IP: Advanced Clocking](/img/14_advanced_clocking.png "Vivado Re-Customize IP: Advanced Clocking")

<em>Modification of the I/O configuration will be explained in detail once peripherals which are not
enabled by default and which must be assigned manually to I/O pins connected to one of the PMOD
expansion connectors.</em>

Once all relevant clocking data has been obtained or modified, close the dialog by clicking 'OK'.
Save your project.

### Optional: adding the AXI GPIO IP cores for the slider switches and LEDs

***************************************************************************

In order to make the 8 slider switches SW0 to SW7 and/or the 8 LEDs above the slider switches,
LD0 to LD7, accessible to your Zephyr application, the addition of two Xilinx AXI GPIO IP cores
is necessary. If your application neither requires access to the slider switches nor the LEDs,
advance to the next section.

**If the switches and LEDs are not accessible, the device tree overlay provided in the
zedboard board directory that enables them MUST NOT BE INCLUDED. The system will crash with
a data abort exception upon trying to access the memory ranges of the GPIO IP cores, which
are non-existant unless the steps described in this section are performed.**

Adding the AXI GPIO IP cores is not required for building the blinky sample, as this uses
the single LED attached to the Zynq's internal GPIO controller, LD9.

As before when adding the Processor System and Processor System Reset blocks, click the
'+' icon in the 'Diagram' pane's icon bar. Select the entry 'AXI GPIO' from the list and
double-click it. An 'AXI GPIO' block will appear in the block design. BEFORE running the
offered Connection Automation, repeat this step so that there are two unconnected 'AXI
GPIO' blocks, labelled 'axi_gpio_0' and 'axi_gpio_1' in the block design before clicking
'Run Connection Automation'.

![Vivado Block Design: One of two AXI GPIO blocks before Block Automation](/img/15_axi_gpio.png "Vivado Block Design: One of two AXI GPIO blocks before Block Automation")

In the 'Run Connection Automation' dialog, check the 'All Automation' checkbox. Afterwards,
before closing the dialog window, click the 'GPIO' nodes located below the 'axi_gpio_0'
and 'axi_gpio_1' nodes in the tree view below the 'All Automation' root node. The 'GPIO'
nodes are used to assign each instance of the of the AXI GPIO core to one of the Zedboard's
pre-defined interfaces, which can be selected from the 'Select Board Part Interface'
drop-down list.

![Vivado Block Design: AXI GPIO Block Automation](/img/16_axi_gpio_config.png "Vivado Block Design: AXI GPIO Block Automation")

Configure the following mapping:

- 'axi_gpio_0' mapped to 'sws_8bits'
- 'axi_gpio_1' mapped to 'leds_8bits'

Click 'OK' to perform the Connection Automation. Re-generate the block design layout in
order to clean up the block design.

Double-click the block 'axi_gpio_0'. The 'Re-customize IP' dialog will open. In the
'IP Configuration' tab, make sure that the 'Enable Dual Channel' checkbox is **unchecked**.
Also make sure that below the 'IP Configuration' tab, the 'Enable Interrupt' checkbox
is also **unchecked** before closing the dialog box by clicking 'OK'. Repeat this step
for the 'axi_gpio_1' block.

![Vivado Block Design: AXI GPIO IP Re-Customization](/img/16a_axi_gpio_config.png "Vivado Block Design: AXI GPIO IP Re-Customization")

The final block design should now contain 5 blocks:

- 'ZYNQ7 Processing System' / 'processing_system_7_0'
- 'Processor System Reset' / 'proc_sys_reset_0'
- 'AXI Interconnect' / 'ps7_0_axi_periph' (added automatically)
- 'AXI GPIO' / 'axi_gpio_0' connected to external port 'sws_8bits'
- 'AXI GPIO' / 'axi_gpio_1' connected to external port 'leds_8bits'

![Vivado Block Design: final block design](/img/17_final_block_design.png "Vivado Block Design: final block design")

Prior to starting systhesis, implementation and bitstream generation, the memory
addresses and memory range sizes for each of the AXI GPIO IP core instances shall
be modified, as by default, an unnecessarily large memory area is reserved per
instance, resulting in the mapping of 4k memory pages by the MMU which aren't
required for anything. In the 'Block Design' view, activate the 'Address Editor'
tab. Under the '/processing_system7_0/Data' node, a node for each AXI GPIO IP
core instance is present: '/axi_gpio_0/S_AXI' and '/axi_gpio_1/S_AXI'.
Change the 'Master Base Address' and 'Range' values to the following configuration:

- '/axi_gpio_0/S_AXI' Base Address 0x4000_0000, range 4K.
- '/axi_gpio_1/S_AXI' Base Address 0x4000_1000, range 4K.

![Vivado Address Editor: AXI GPIO address configuration](/img/17a_address_editor.png "Vivado Address Editor: AXI GPIO address configuration")

Save your project.

### Synthesis, Implementation and bitstream generation

******************************************************

In the 'Block Design' view, activate the 'Sources' tab. Expand the node 'Design Sources'.
Right-click the 'design_1' node (actual name depends on the design name assigned upon
the creation of the Vivado project) and select 'Create HDL Wrapper':

![Vivado Block Design: Create HDL Wrapper](/img/18_hdl_wrapper.png "Vivado Block Design: Create HDL Wrapper")

In the 'Create HDL Wrapper' dialog, leave the default choice 'Let Vivado manage wrapper
and auto-update' unchanged and click 'OK'. Once the wrapper creation process has completed,
under the 'Design Sources' node, there will now be an entry 'design_1_wrapper(STRUCTURE)'.

In the 'Flow Navigator' menu, click the entry 'Generate Bitstream' below the 'PROGRAM
AND DEBUG' node. A dialog box prompting 'There are no implementation results available.
OK to launch synthesis and implementation?' Click 'Yes' to confirm. In the 'Launch Runs'
dialog, keep the default choice 'Launch runs on local host'. The number of parallel jobs
by default equals the number of CPUs present in the host machine. This number can be
reduced in order to reduce the CPU load of the host machine. Click 'OK' to confirm your
choice.

Synthesis and implementation of the block design will now be performed and the FPGA
bitstream will be generated afterwards.

**This process may take a long time and is not complete before the dialog 'Bitstream
Generation Completed' appears.**

None of the options offered by the 'Bitstream Generation Completed' dialog is required,
therefore, click 'Cancel' in order to close this dialog box.

### Project data export to Vitis

********************************

Open the menu 'File', select 'Export' followed by 'Export Hardware...':

![Vivado: Export Hardware](/img/18_hdl_wrapper.png "Vivado Block Design: Export Hardware")

The 'Export Hardware Platform' wizard dialog will appear. Click 'Next' to move on from the
introductory text to the 'Output' options choice. Select the option 'Include bitstream', then
click 'Next'. The next step of the wizard is the 'Files' options. Leave those options at
their default values. The implementation results plus the bitstream will be stored in an
XSA archive file named "design_1_wrapper" (unless a different design name was chosen upon
project creation) in the project's directory within Vivado's workspace directory. Click
'Next' to continue, when shown the summary of the wizard, click 'Finish'.

Once the data has been exported for import into Vitis, Vivado can be closed.

**Hint:** If you navigate to the project directory within Vivado's workspace directory,
you will find the exported XSA file next to a number of sub-directories. XSA files are
actually ZIP archives, and under Linux, this type of file should usually be associated
with the respective distribution's archive manager. If you open the file with a ZIP-
capable archive manager, within the archive you will find a file called 'ps7_init.html'.
This HTML file contains a summary of the implemented clocking configuration for each of
the SoC's components as well as of the implemented I/O pin mapping and how the respective
configurations map to the contents of the SLCR configuration registers. This sequence
of configuration register writes is also contained in the file 'ps7_init.tcl'. Any JTAG
debugger frontend capable of script-based parsing of this file's contents can make use
of this file to configure the system prior to uploading a Zephyr binary if the board is
not configured by running the FSBL. This is for example the case with Lauterbach's
TRACE32 frontend for their PowerDebug series of JTAG debuggers.

## Step 2: u-boot: configuring and building the 2nd stage bootloader

### Setting up the build environment

************************************

Open a shell and change the working directory to the cloned u-boot-xlnx repository.
In order to build u-boot for the Zedboard, set the following environment variables:

```
export ARCH=arm
export CROSS_COMPILE=arm-none-eabi-
export DEVICE_TREE=zynq-zed
```

The cross compiler required for building u-boot, arm-none-eabi-gcc and its associated
binutils, are provided in the Vitis installation. A shell script in the Vitis installation
directory adds the compiler's installation directory to the PATH:

```
source /<your Xilinx root install directory>/Vitis/2021.1/settings64.sh
```

Check that arm-none-eabi-gcc is available after sourcing the settings64 script.
Afterwards, create the default build configuration for the Zedboard:

```
make xilinx_zynq_virt_defconfig
```

### Customizing the boot configuration

**************************************

In order to hand the system over to Zephyr in a state it can work with when coming out
of u-boot, certain parameters in the u-boot configuration must be modified. Enter the
configuration menu:

```
make menuconfig
```

In the configuration menu, modify the following configuration parameters:

- ARM architecture / Do not enable icache : check
- ARM architecture / Do not enable icache in SPL : check
- ARM architecture / Do not enable dcache : check
- ARM architecture / MMU-based Paged Memory Management Support : uncheck
- ARM architecture / L2cache off : check
- Boot options / bootcmd value : ```fatload mmc 0 0x2000000 zephyr.elf;bootelf 0x2000000```

Exit the configuration menu and save the modified settings. Afterwards, build the
u-boot bootloader by invoking

```
make
```

Once the build process has completed, the boot loader binary which will be used for
booting the Zedboard, u-boot.elf, will have been created.

## Step 3: Vitis: configuring and creating BOOT.bin

The Zynq's boot ROM requires a file named BOOT.bin to be present on the SD card.
BOOT.bin is a container file which contains the first stage bootloader (FSBL),
the FPGA bitstream and the u-boot binary as second stage bootloader to which execution
will be handed over once the Processor System has been configured and the FPGA
bitstream has been programmed. Whenever a configuration parameter of the Processor
System is changed in the Vivado project, the contents of the Programmable Logic are
modified or a new u-boot binary with a modified configuration is built, this file
will have to be re-built. As long as none of these items are modified, it is possible
to just replace the zephyr.elf file on the SD card whenever Zephyr is re-built.

### Setting up the Vitis project

********************************

Start Vitis. On the startup screen, open the 'File' menu, then navigate to 'New' and
'Application Project'.

![Vitis startup: 'Create Application Project'](/img/20_create_vitis_project.png "Vitis startup: 'Create Application Project'")

The 'New application project' wizard will open, click 'Next >' to advance past the
introduction. The next step is the 'Platform' dialog. Switch to the 'Create a new
platform from hardware (XSA)' tab, then click the 'Browse' button. In the file
selection dialog which will then appear, navigate to your Vivado workspace, then on
to the directory containing the project generated in step 1. Within that project
directory, select the .xsa file, which has the same name as that chosen for the
design wrapper in step 1 at the 'Synthesis, Implementation and bitstream generation'
step.

Once you have confirmed your selection, the wrapper name will appear in the 'Platform
name:' text box. Make sure that the 'Generate boot components' checkbox is checked
before clicking 'Next >':

![Vitis: 'Create Application Project: Platform'](/img/21_platform_add.png "Vitis: 'Create Application Project: Platform'")

The 'Application Project Details' dialog is next, enter a name for the Vitis project
in the 'Application Project name' text box. This name is automatically applied to the
'System project name' text box and the 'Associated applications' column in the 'Target
processor' grid view. Make sure that the application is associated with the 'Processor'
entry 'ps7_cortexa9_0', this is the CPU core which will be used for the boot process
and eventually by Zephyr. Once all entries are present, click 'Next >':

![Vitis: 'Create Application Project: Application Project Details'](/img/22_project_name.png "Vitis: 'Create Application Project: Application Project Details'")

The 'Domain' dialog is next. Its default configuration does not require any modifications:

- Name / Display Name: standalone_ps7_cortexa9_0
- Operating System: standalone
- Processor: ps7_cortexa9_0
- Architecture: 32-bit.

Click 'Next >' to move on to the 'Templates' dialog.

In the 'Templates' dialog, select the 'Zynq FSBL' template before clicking 'Finish':

![Vitis: 'Create Application Project: Templates'](/img/23_templates.png "Vitis: 'Create Application Project: Templates'")

The 'Explorer' tab will now contain two entries at the top-level: one for the imported
design wrapper, and one for the project based on it:

![Vitis: 'Explorer'](/img/24_explorer.png "Vitis: 'Explorer'")

The Vitis project setup is now complete and BOOT.bin for the Zedboard can now be created.

### Building the project

************************

Open the 'Project' menu and click 'Build All'. The FSBL binary including the configuration
settings of the Processor System specified in Vivado will be built.

### Creating BOOT.bin

*********************

Open the 'Xilinx' menu, select 'Create Boot Image', 'Zynq and Zynq Ultrascale':

![Vitis: 'Xilinx' menu](/img/25_create_boot_image.png "Vitis: 'Xilinx' menu")

In the 'Create Boot Image' dialog, make sure that in the 'Architecture' drop-down
box, 'Zynq' is selected. Set 'Create new BIF file' as the active mode. The output
format must be set to 'BIN'.

- If no output path for the BIF file is suggested, select a convenient location
  for the generated file using the corresponding 'Browse...' button. The BIF file
  will contain the configuration of the boot image as specified in this dialog.
- Choose a convenient output path for the resulting BOOT.bin file using the 'Browse...'
  button in the 'Output path:' row
- Add three entries to the 'Boot image partitions' grid view by clicking the 'Add'
  button once for every file to be added:

![Vitis: 'Create Boot Image' dialog](/img/26_boot_image_basic.png "Vitis: 'Create Boot Image' dialog")

The first file to be added to BOOT.bin is the FSBL binary. Using the 'Browse...'
button in the 'File path:' row, navigate to the 'Debug' sub-directory of the project
directory within the Vitis workspace belonging to the nested build project which
is labelled 'standalone_ps7_cortexa9_0'. Select the ELF binary within the 'Debug'
directory. Make sure the 'Partition type:' drop-down box is set to 'bootloader':

![Vitis: 'Add partition' dialog, FSBL](/img/27_bootloader.png "Vitis: 'Add partition' dialog, FSBL")

The second file to be added is the FPGA bitstream. Withih the Vitis workspace,
navigate to the sub-directory named after the design wrapper imported from Vivado
using the 'Browse...' button in the 'Add partition' dialog. Within the design
wrapper's project sub-directory, navigate to the 'hw' sub-directory. The file to
be added is the '.bit' file contained in the 'hw' sub-directory. The 'Partition
type' drop-down box must be set to 'datafile' for the bitstream.

The third file to be added is the u-boot binary. Select 'u-boot.elf' in the
directory in which u-boot was built. For this file, the 'Partition type' must
also be set to 'datafile'.

**Notice:** the order of the files to be added to BOOT.bin can be modified using
the 'Up' and 'Down' buttons. The following order **must** be observed:

1. FSBL
2. Bitstream
3. u-boot:

![Vitis: correct BOOT.bin partition order](/img/28_boot_order.png "Vitis: correct BOOT.bin partition order")

The output file is now configured correctly. Create the boot image by clicking
'Create Image'. The resulting file will appear in the 'Explorer' tab:

![Vitis: BOOT.bin in 'Explorer' tab](/img/29_boot_bin_explorer.png "Vitis: BOOT.bin in 'Explorer' tab")

Copy the generated BOOT.bin to the root directory of the SD card to be used for
booting Zephyr.

## Step 4: Zephyr: building and running Zephyr

### Building the Zephyr binary

******************************

Use the target board identifier 'zedboard' when building a Zephyr binary.
In order to build the blinky sample, run the following command:

```
west build -b zedboard samples/basic/blinky
```

If your system design contains the two optional instances of the AXI GPIO
controller IP core described in step 1, add the provided device tree overlay
in order to make them accessible in Zephyr:

```
west build -b zedboard <application> -- -DDTC_OVERLAY_FILE=axi_gpio_switches_leds.overlay
```

The overlay file defines the aliases 'ld0' to 'ld7' for the corresponding
LEDs above the slider switches, and 'sw0' to 'sw7' for the slider switches.
For out-of-the-box compatibility with the blinky sample, the single LED
attached to the Processor System's GPIO controller, LD9, which is always
available without having to include a device tree overlay, has the alias
'led0'.

Once the Zephyr binary has been built, copy ```zephyr.elf``` from the ```build/zephyr```
sub-directory of your Zephyr installation to the SD card's root directory, which
should now contain both BOOT.bin and zepyhr.elf.

### Setting up the Zedboard for booting from SD card

****************************************************

While the Zedboard is off, re-configure the boot mode jumpers JP7 to JP11 to the
following setting:

![Zedboard: jumpers JP7 to JP11](/img/30_jumper.png "Zedboard: jumpers JP7 to JP11")

### Console UART

****************

The device used for the serial console / shell is 'uart1', which by default
is configured to 115200/8N1. Once the board is powered on, on a Linux host,
the corresponding device ```/dev/ttyACM<n>``` will appear.

### Booting Zephyr

******************

Power on the Zedboard with the SD card inserted. If the FSBL works as intended,
the blue LED LD12/DONE between the red pushbuttons (BTN6/PROG, BTN7/PS-RST) and
the OLED display comes on shortly after the board is powered on. The u-boot console
should now appear. If the automatic boot sequence is not interrupt, u-boot will
boot straight into Zephyr.

![Zedboard: Zephyr booting](/img/31_putty.png "Zephyr booting")

## Further documentation

[Avnet Zedboard product website](https://www.avnet.com/wps/portal/us/products/avnet-boards/avnet-board-families/zedboard/)
[Xilinx Zynq-7000 All Programmable SoC Technical Reference Manual ](https://docs.xilinx.com/v/u/en-US/ug585-Zynq-7000-TRM)
