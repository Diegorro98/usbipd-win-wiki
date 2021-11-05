<!--
SPDX-FileCopyrightText: Microsoft Corporation

SPDX-License-Identifier: GPL-2.0-only
-->

# WSL setup

Running uname -a from within WSL should report a kernel version of 5.10.60.1 or later. You’ll need to be running a WSL 2 distro.

From within WSL, install the user space tools for USB/IP and a database of USB hardware identifiers. On Ubuntu, run this command.

```bash
sudo apt install linux-tools-5.4.0-77-generic hwdata
```

Edit /etc/sudoers so that root can find the usbip command. On Ubuntu, run this command.
```bash
sudo visudo
```
Add /usr/lib/linux-tools/5.4.0-77-generic to the beginning of secure_path. After editing, the line should look similar to this.
```bash
Defaults secure_path="/usr/lib/linux-tools/5.4.0-77-generic:/usr/local/sbin:..."
```
Note that depending on your application, you may need to configure udev rules to allow non-root users to access the device. Rules to enable a device must be in place before connecting the device. As a common example for using embedded devices with openocd copy share/openocd/contrib60-openocd.rules to the /etc/udev/rules.d folder. 

After updating your rules run udevadm control --reload. If you get an error that "Failed to send reload request: No such file or directory", run sudo service udev restart then run it again.

At this point the `usbip` command should be available to root, and running `sudo usbip` in WSL should successfully find the command. For non-root access, edit your `.bashrc` to add an alias.

```bash
alias usbip=/usr/lib/linux-tools/5.4.0-77-generic/usbip
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

# Building your own USBIP enabled WSL 2 kernel

_Recent versions of Windows running WSL kernel 5.10.60.1 or later already include support for common scenarios like USB-to-serial adapters and flashing embedded development boards. If you're trying to do one of these tasks on Ubuntu, you can avoid recompiling the kernel by following the WSL Setup instructions at the top of this page. If you require special drivers, you'll need to build your own kernel for WSL 2._

Update WSL:

```pwsh
wsl --update
```

List your distributions.

```pwsh
wsl --list --vebose
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

Build USBIP tools.

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