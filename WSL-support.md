<!--
SPDX-FileCopyrightText: Microsoft Corporation

SPDX-License-Identifier: GPL-2.0-only
-->

# WSL setup

## WSL version

Running uname -a from within WSL should report a kernel version of 5.10.60.1 or later. You’ll need to be running a WSL 2 distro.

## USB/IP client tools

From within WSL, install the user space tools for USB/IP and a database of USB hardware identifiers. On Ubuntu 20.04 LTS, run these commands:

```bash
sudo apt install linux-tools-virtual hwdata
sudo update-alternatives --install /usr/local/bin/usbip usbip `ls /usr/lib/linux-tools/*/usbip | tail -n1` 20
```

⚠️ **These instructions have changed.**\
Updating to `usbipd-win` version 2.0.0 or higher will require these new instructions, even if the old instructions were followed before.

ℹ️ **Installing package updates.**\
After installing package updates (e.g. with `apt upgrade`), you may have to run `update-alternatives` again to re-enable the `usbip` command.

ℹ️ **Other distributions.**\
For other distributions a different `usbip` client package may be required. In any case, make sure that the resulting `usbip` command is in the PATH for user root; for example by adjusting the above `update-alternatives`. Please search the (possibly closed) [issues](https://github.com/dorssel/usbipd-win/issues?q=is%3Aissue) to see if instructions for your distribution are already known.

## udev

Note that depending on your application, you may need to configure udev rules to allow non-root users to access the device. Rules to enable a device must be in place before connecting the device. As a common example for using embedded devices with openocd copy share/openocd/contrib60-openocd.rules to the /etc/udev/rules.d folder. 

After updating your rules run `udevadm control --reload`. If you get an error that "Failed to send reload request: No such file or directory", run `sudo service udev restart` then run it again.

If you plan to allow non-root users to access the device frequently and udev service is stopped every time the distribution is booted, you can add this to `/etc/wsl.conf`:

```
[boot]
command = "service udev restart"
```

# WSL convenience commands

After following the setup instructions above and installing usbipd in Windows, you can use the usbipd WSL convenience commands to easily attach devices to a WSL instance and view which distributions devices are attached to.

First ensure a WSL command prompt is open. This will keep the WSL 2 lightweight VM active.

From an administrator command prompt on Windows, run this command. It will list all the USB devices connected to Windows.

```pwsh
> usbipd wsl list
BUSID  DEVICE                                      STATE
1-7    USB Input Device                            Not attached
4-4    STMicroelectronics STLink dongle, STMic...  Not attached
5-2    Surface Ethernet Adapter                    Not attached
```

Select the bus ID of the device you’d like to attach to WSL and run this command. You’ll be prompted by WSL for a password to run a sudo command.
```pwsh
> usbipd wsl attach --busid 4-4
[sudo] password for user:
```

Now we can run list again and see the device is shared with WSL.
```pwsh
> usbipd wsl list
BUSID  DEVICE                                      STATE
1-7    USB Input Device                            Not attached
4-4    STMicroelectronics STLink dongle, STMic...  Attached - Ubuntu
5-2    Surface Ethernet Adapter                    Not attached
```

From within WSL, run lsusb to list the attached USB devices. You should see the device you just attached and be able to interact with it using normal Linux tools. 

```bash
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 002: ID 0483:374b STMicroelectronics ST-LINK/V2.1
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

`wsl detach` can be used to stop sharing the device. The device will also
automatically stop sharing if it is unplugged or the computer is restarted.

```pwsh
> usbipd wsl detach --busid 4-4

> usbipd wsl list
BUSID  DEVICE                                      STATE
1-7    USB Input Device                            Not attached
4-4    STMicroelectronics STLink dongle, STMic...  Not attached
5-2    Surface Ethernet Adapter                    Not attached
```

Use the `--help` to learn more about these convenience commands. In particular,
the `--distribution` and `--usbippath` options can be useful to customize how
the WSL commands are invoked.

# Building your own USB/IP enabled WSL 2 kernel

_Recent versions of Windows running WSL kernel 5.10.60.1 or later already include support for common scenarios like USB-to-serial adapters and flashing embedded development boards. If you're trying to do one of these tasks on Ubuntu, you can avoid recompiling the kernel by following the WSL Setup instructions at the top of this page. If you require special drivers, you'll need to build your own kernel for WSL 2._

Update WSL:

```pwsh
wsl --update
```

List your distributions.

```pwsh
wsl --list --verbose
```

Verify that your target distribution is version 2;
see [WSL documentation](https://docs.microsoft.com/en-us/windows/wsl/install-win10#set-your-distribution-version-to-wsl-1-or-wsl-2)
for instructions on how to set the WSL version.

Export current distribution to be able to fall back if something goes wrong.

```pwsh
wsl --export <current-distro> <temporary-path>\wsl2-usbip.tar
```

Import new distribution with current distribution as base.

```pwsh
wsl --import wsl2-usbip <install-path> <temporary-path>\wsl2-usbip.tar
```

Run new distribution.

```pwsh
wsl --distribution wsl2-usbip --user <user>
```

Update resources (assuming `apt`, you may need to use `yum` or another package manager).

```bash
sudo apt update
sudo apt upgrade
```

Install prerequisites.

```bash
sudo apt install build-essential flex bison libssl-dev libelf-dev libncurses-dev autoconf libudev-dev libtool
```

Clone kernel that matches wsl version. To find the version you can run.

```bash
uname -r
```

The kernel can be found at: <https://github.com/microsoft/WSL2-Linux-Kernel>

Clone the kernel repo, then checkout the branch/tag that matches your kernel version; run `uname -r` to find the kernel version.

```bash
git clone https://github.com/microsoft/WSL2-Linux-Kernel.git
cd WSL2-Linux-Kernel
git checkout linux-msft-wsl-5.10.43.3
```

Copy current configuration file.

```bash
cp /proc/config.gz config.gz
gunzip config.gz
mv config .config
```

You may need to set CONFIG_USB=y in .config prior to running menuconfig to get all options enabled for selection.

Run menuconfig to select kernel features to add.

```bash
sudo make menuconfig
```

These are the necessary additional features in menuconfig.\
Device Drivers -> USB Support\
Device Drivers -> USB Support -> USB announce new devices\
Device Drivers -> USB Support -> USB Modem (CDC ACM) support\
Device Drivers -> USB Support -> USB/IP\
Device Drivers -> USB Support -> USB/IP -> VHCI HCD\
Device Drivers -> USB Support -> USB/IP -> Debug messages for USB/IP\
Device Drivers -> USB Serial Converter Support\
Device Drivers -> USB Serial Converter Support -> USB FTDI Single port Serial Driver

In the following command the number '8' is the number of cores to use; run `getconf _NPROCESSORS_ONLN` to find the number of cores.

```bash
sudo make -j 8 && sudo make modules_install -j 8 && sudo make install -j 8
```

Build USB/IP tools.

```bash
cd tools/usb/usbip
sudo ./autogen.sh
sudo ./configure
sudo make install -j 8
```

Copy tools libraries location so usbip tools can get them.

```bash
sudo cp libsrc/.libs/libusbip.so.0 /lib/libusbip.so.0
```

Install usb.ids so you have names displayed for usb devices.

```bash
sudo apt-get install hwdata
```

From the root of the repo, copy the image.

```bash
cp arch/x86/boot/bzImage /mnt/c/Users/<user>/usbip-bzImage
```

Create a `.wslconfig` file on `/mnt/c/Users/<user>/` and add a reference to the created image with the following.

```ini
[wsl2]
kernel=c:\\users\\<user>\\usbip-bzImage
```

Your WSL distro is now ready to use!

# Bluetooth on WSL2

If you have a Bluetooth adapter available to attach using usbipd (you can check it by checking if is present in `usbipd list`), you can make it work in WSL2 by following these steps:

First you should check if it there is kernel drivers for the device, you can check it in [Hardware for Linux](https://linux-hardware.org/?view=search) with the id of the Bluetooth adapter.
You can obtain it by executing `usbipd list`:
```
BUSID  VID:PID		DEVICE				STATE
1-1    xxxx:0000  	Bluetooth device	Not attached
```
\
After knowing that there is kernel drivers, you should follow [Building your own USB/IP enabled WSL 2 kernel](/WSL-support.md#building-your-own-usbip-enabled-wsl-2-kernel) instructions but enable the following Bluetooth features and those on which they depend in menuconfig:\
Networking support -> Bluetooth subsystem support\
Networking support -> Bluetooth subsystem support -> Bluetooth device drivers -> Bluetooth HCI USB driver\

Install bluez:
```
sudo apt-get install bluez
```

Enable Bluetooth:
```
echo 'export BLUETOOTH_ENABLED=1' | sudo tee /etc/default/bluetooth
```

Start dbus and Bluetooth services:
```
sudo service dbus start
sudo service bluetooth start
```

Alternatively, if you are going to use frequently the Bluetooth device on the distro, you can start the services every time the distro is booted by setting in `/etc/wsl.conf` the following content: 
```
[boot]
command = "service dbus start && service bluetooth start"
```

Attach the Bluetooth device to the WSL2 distro using usbipd and test if it works by running `bluetoothctl list`. you should get an output similar to this:
```
Controller XX:XX:XX:XX:XX:XX BlueZ 5.53 [default]
```
 
If it doesn't work, it might be due to one of the following reasons:
- There isn't any kernel drivers. Check it at Hardware linux.
- The device is too new. Microsoft wsl2 kernel might not support yet the device. Check https://github.com/dorssel/usbipd-win/discussions/310#discussioncomment-2483246.
- Some more features are required at the kernel's menuconfig. Check Hardware for linux to see if there is additional features required at Config column.
