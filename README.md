# NextJTAG

NextJTAG is a standalone command line utility used for accessing Xilinx FPGAs over USB.  It supports basic operations, such as checking the temperature and loading bitstreams.  Platform and FPGA support are fairly limited, but more are coming soon.  To gain access to all features, a [license](#Licensing) must be purchased.  Check the [releases](../../releases) to download the latest binaries.

## Supported Features

* Querying Device DNAs of attached FGPAs
* Loading bitstreams in parallel (not persistent across power cycling)
* Clearing the currently loaded bitstream
* Reloading the bitstream from flash
* Reading the min, max, and current temperature and voltage
* Reading/writing SYSMON registers
* Changing voltage controller settings (BCU1525 only, requires v2.0+)

## Supported Xilinx FPGAs

* XCVU9P (VCU1525, BCU1525, or BTU9P)
* XCVU13P (CVP13) - in BETA, requires v2.0.1+
* (More coming soon!)

## Supported Platforms

* Linux (x86-64)
* Windows (x64)

## Licensing

Licensing is done per FPGA, and is tied to the FPGA's Device DNA (a unique identifier fused into the chip by the manufacturer).  The majority of features require a license to use.  The only feature that does not require a license is to list the Device DNAs of all FPGAs connected to the system.

Licenses are stored in a text file, with each line specifying the Device DNA and associated license separated by whitespace.  Blank lines and lines starting with `#` are ignored.  The license file is searched for in three places:

1. If the `NEXTJTAG_LICENSE` environment variable is set, it will use the value as the (relative or absolute) path to the license file
2. If the environment variable is not set, then it looks for `nextjtag_license.txt` in the current directory
3. If that file does not exist, it looks for `nextjtag_license.txt` in the directory the NextJTAG tool is located in

Licenses must be purchased from Next Design Solutions directly or through an authorized third party vendor ([see purchasing options](#purchasing-options)).  Licenses are valid indefinitely, but will only work with the same major version of NextJTAG.  For example, a 1.x license will work with versions 1.0 and 1.1, but will not not with version 2.0.  In general, bug fixes and basic new features will not result in a major version bump.


### Purchasing options

* [fpga.guide](https://shop.fpga.guide/collections/all/products/nextjtag-license) (Paypal, Credit Card, or various crypto currencies)
* Contact `turekaj#2845` on discord (Ethereum only)

## Permissions (Linux only)

NextJTAG needs access to the FTDI chip on the FPGA over USB in order to work, which requires special permissions on Linux platforms.  There are two options:
1. Run as root user (not ideal, but convenient)
2. Install udev rules to allow non-root users access to read and write to USB FTDI devices (best practice)

To do option 2, simply copy the `.rules` files from the release (located in `<release>/lib/udev/rules.d`) into the `/etc/udev/rules.d` directory on your system.  However, the rule will only apply to new USB connections.  If you already have a FPGA connected over USB, you can manually trigger the rule by running `udevadm trigger --action=add` as root.

## Driver Install (Windows only)

NextJTAG supports two drivers: a proprietary FTDI driver from the manufacturer and an open source driver.  Only one driver needs to be used.

### Proprietary FTDI driver (default for Windows)

Most other tools and miners, including Vivado, use this driver, so using this with NextJTAG ensures maximum interoperability.  If the driver isn't already installed, it can be found [here](https://www.ftdichip.com/Drivers/D2XX.htm) (there is a Windows installer if you search for "setup executable" on that page).

Once the driver is installed, you should be able to run NextJTAG without any arguments, and see a list of your FPGAs.

### Open Source FTDI driver (advanced users only)

The open source driver is around for legacy reasons, and not recommended for most users.  The open source driver will need to be configured for the FTDI interfaces that NextJTAG uses (Interface 0 for JTAG, Interface 2 for BMC control).

1. (optional) Installed the Install the [proprietary driver](#proprietary-ftdi-driver-default-for-windows).  This will create a clean slate, and will use the proprietary driver as default the for all interfaces for the FTDI chip.  If you are already using Vivado, this is probably not necessary
2. Download the [zadig](https://zadig.akeo.ie/) tool and run it as administrator.  This is a convenient tool to for installing drivers compatible with `libusb`
3. In Zadig, select `Options > Show all devices`, and then use the dropdown to select Interface 0 for one of your FPGA cards.  On the line below the dropdown, select `libusbK` and then hit the `Replace Driver` button.  Repeat for Interface 0 for all of your FPGA cards.  Do not replace the driver for the other interfaces, since some miners require the proprietary driver for those to work.
4. (optional) If you require BMC control on the BCU1525, repeat step 3 for Interface 2 for all FPGA cards.  This may prevent USB based miners from working, so you might need to switch it back after the BMC is configured.

![alt text](zadig.png "Example to replace driver for VCU1525")

The changes made by Zadig will be persistent, even after reboots.  To remove the open source driver, just install the proprietary driver again from step 1.  **You will need to do this in order to use Vivado again after running these steps**

In order to tell NextJTAG to use the open source driver, the `--prefer_libftdi` must be passed on the commad line.

## Examples

Query Device DNAs from all attached devices (no license required):

```
$ nextjtag
List of available devices:
  0: Xilinx VCU1525 Dev Kit (DNA = 400200000117ab282d410505)
```

Read the temperature from device 0:
```
$ nextjtag -d0 -t
[2018-11-12 03:32:50] Device 0: Temperature: 43.5624°C (min), 58.4837°C (current), 71.4155°C (max)
```

Program bitstream into all FPGAs in parallel:
```
$ nextjtag -a -m -b vcu1525_keccak_21_600.bit
[2018-11-11 21:38:34] Device 0: Programming bitstream: STARTING...
[2018-11-11 21:38:34] Device 1: Programming bitstream: STARTING...
[2018-11-11 21:38:34] Device 2: Programming bitstream: STARTING...
[2018-11-11 21:38:34] Device 3: Programming bitstream: STARTING...
[2018-11-11 21:39:21] Device 1: Programming bitstream: SUCCESS
[2018-11-11 21:39:21] Device 0: Programming bitstream: SUCCESS
[2018-11-11 21:39:21] Device 2: Programming bitstream: SUCCESS
[2018-11-11 21:39:22] Device 3: Programming bitstream: SUCCESS
```

Set voltage for device 0 (BCU1525 only, requires v2.0+):
```
$ nextjtag -d0 --set_voltage=0.72
Using license file: /home/mbrase/nextjtag_license.txt
[2019-01-30 02:10:15] Device 0: Changing voltage: STARTING...
[2019-01-30 02:10:23] Device 0: Changing voltage: SUCCESS
```

## Troubleshooting

How to fix some common problems.

#### No devices avaiable or missing devices

```
$ nextjtag
List of available devices:
  (No devices found)
```

Check that your FPGAs are plugged in.  On Linux, you can do `lsusb` and search for FT4232H or "Future Technology Devices International" (FTDI).  On Windows, check device manager, and make sure you have gone through the [driver install](#driver-install-windows-only).

If it was previously working on Windows, but devices are not showing up now, then sometimes the FTDI driver gets stuck and prevents NextJTAG from opening deivces.  You can try running NextJTAG with the experimental `--force_ftd2xx_reload` option to attempt to get it unstuck.  If that doesn't work, you can try unplugging and plugging the device back in, or rebooting.

#### Unable to open device (no device name)

```
$ nextjtag
List of available devices:
  0:  (Unable to open device)
```

On Linux, this is probably a [permission](#permissions-linux-only) problem.  Try running as root or installing udev rules.

On Windows, this might be a [driver](#driver-install-windows-only) issue.  Sometimes a reboot will help.

#### Unable to open device (with device name)

```
$ nextjtag
List of available devices:
  0: Xilinx VCU1525 Dev Kit (Unable to open device)
```

The device is probably already open by another process.  Check for other instances of NextJTAG that are running.  Also check if Vivado is running, and if so, make sure it isn't connected to the FPGA.  If that doesn't work, you can try rebooting.

#### Unable to initialize device

```
$ nextjtag
List of available devices:
  0: Xilinx VCU1525 Dev Kit (Unable to initialize FPGA)
```

This means that it was able to open the device, but it received something unexpected from the FTDI chip.  For example, this can happen if you use NextJTAG with a non-supported FPGA.  It is also possible that the FTDI chip is in a bad state, but that is pretty rare.  You can try power cycling the card to see if it helps.

#### Bogus min/max temperatures

```
$ nextjtag -a -t
[2018-11-14 00:48:23] Device 0: Temperature: 228.5866°C (min), 53.5099°C (current), -280.2300°C (max)
```

This is a known issue with certain hardware/bitstream configurations, and something we can hopefully provide a workaround for in the future.

#### Weird characters being printed out for temperature command

This is probably an issue with your shell not rendering the degree symbol, which requires UTF-8 encoding.

On Windows, run `chcp 65001` in the command prompt or `[Console]::OutputEncoding = [System.Text.Encoding]::UTF8` in powershell.

On Linux, it depends on your terminal program.
