| Device ID | Description                                       | USB version | Result             | Reason                                            |
|-----------|---------------------------------------------------|-------------|--------------------|---------------------------------------------------|
| 0403:6001 | Argolis Smartreader (FTDI)                        | 1.1         | :heavy_check_mark: |                                                   |
| 0718:0081 | Imation Flash Drive                               | 2.0         | :heavy_check_mark: |                                                   |
| 1845:0104 | Unknown Brand Flash Drive                         | 2.0         | :heavy_check_mark: |                                                   |
| 0951:1666 | Kingston DataTraveler 100                         | 3.0 / 3.1   | :heavy_check_mark: |                                                   |
| 050d:0012 | Belkin F8T012 Bluetooth Adapter                   | 2.0         | :heavy_check_mark: |                                                   |
| 1050:0407 | Yubikey 5                                         | 2.0         | :heavy_check_mark: |                                                   |
| 0529:0620 | SafeNet Aladdin Token                             | 2.0         | :heavy_check_mark: |                                                   |
| 2e04:c025 | HMD Global Nokia 9 PureView                       | 2.0         | :heavy_check_mark: |                                                   |
| 1bcf:28a6 | DELL XPS Integrated Webcam                        | 2.0         | :heavy_check_mark: |                                                   |
| 0c45:6366 | Microdia USB Camera                               | 2.0         | :heavy_check_mark: |                                                   |
| 0483:374b | STMicroelectronics ST-LINK/V2.1                   | 1.1 / 2.0   | :heavy_check_mark: |                                                   |
| 10c4:ea60 | Silicon Labs CP210x UART Bridge                   | 1.1         | :yellow_circle:    | WSL: Driver not present, requires building kernel |
| 1915:7777 | Nordic Semiconductor ASA Crazyradio PA USB Dongle | 1.1         | :yellow_circle:    | (possibly) fixed in master, not released yet      |
| 0781:5588 | SanDisk Corp. USB Extreme Pro                     | 3.0 / 3.1   | :heavy_check_mark: |                                                   |
| f055:9800 | MicroPython Pyboard Virtual Comm Port in FS Mode  | 1.1 / 2.0   | :heavy_check_mark: |                                                   |
| 2e8a:0005 | Pimoroni pico lipo MicroPython Board in FS mode   | 1.1 / 2.0   | :heavy_check_mark: |                                                   |

#  Device Test Procedure 

## Preparation 
Before testing new devices, please configure your machine and verify your setup with at least one other known good device.

### Verify WSL version
(from a Windows prompt

- [ ] `wsl --list --verbose`  
  Verify that you are running as WSL version 2
```
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

## Verify device

- [ ]  Plug the USB device into a USB port  
  Verify that the device is connected and recognized by Windows

### Share usb device
From a Windows **Elevated** prompt (run as Administrator), run the following 


- [ ] `usbipd list`  
  Verify that the device is visible and note the BUSID
- [ ] `usbipd wsl list`  
  Verify that the device is visible, and listed as **'Not attached'**
- [ ] `usbipd wsl attach --busid <BUSID>`  
  Verify that the device state is now **`Shared`**
- [ ] `usbipd wsl list`  
  Verify that the device is visible, and listed as **'Attached - Ubuntu'** (name of your distro may vary)


### Verify USB available in WSL
 
Below steps are based on Ubuntu, details may vary for other distros.
#### All USB device types 

- [ ] `lsusb`  
  Verify that the device is visible
    ```log
    jos@contoso:/mnt/c/Users/jos$ lsusb
    Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    Bus 001 Device 002: ID f055:9800 MicroPython Pyboard Virtual Comm Port in FS Mode
    Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
    ```
- [ ] `dmesg | tail`  
  There should be a recent message indicating that the the device has been discovered similar to the example below.
    ``` log
    jos@contoso:/mnt/c/Users/jos$ dmesg | tail
    [ 2401.168835] vhci_hcd: vhci_device speed not set
    [ 2401.238829] usb 1-1: new full-speed USB device number 2 using vhci_hcd
    [ 2401.318859] vhci_hcd: vhci_device speed not set 
    [ 2401.389087] usb 1-1: SetAddress Request (2) to port 0
    [ 2401.453006] usb 1-1: New USB device found, idVendor=f055, idProduct=9800, bcdDevice= 2.00
    [ 2401.453010] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
    [ 2401.453013] usb 1-1: Product: Pyboard Virtual Comm Port in FS Mode
    [ 2401.453014] usb 1-1: Manufacturer: MicroPython
    [ 2401.453015] usb 1-1: SerialNumber: 206437A1304E
    [ 2401.458834] cdc_acm 1-1:1.1: ttyACM0: USB ACM device
    ```

#### USB serial port devices
- [ ] `ls /dev/tty*`  
  Verify that the device is visible as `/tty/S<n>` or `/tty/ACM<n>`

For terminal like devices:
- [ ] `screen /dev/ttyACM0 115200`  
  Verify that you can connect to the device and interact or send / receive .

#### USB Composite Devices:
- [ ] run `lsusb --tree`  
  Verify that all expected USB interfaces are shown, such as mass storage and communication devices
    ```log
    jos@contoso:/mnt/c/Users/jos$ lsusb --tree
    /:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=vhci_hcd/8p, 5000M
        |__ Port 1: Dev 2, If 0, Class=Mass Storage, Driver=, 5000M
    /:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=vhci_hcd/8p, 480M
        |__ Port 1: Dev 2, If 0, Class=Mass Storage, Driver=, 12M
        |__ Port 1: Dev 2, If 1, Class=Communications, Driver=cdc_acm, 12M
        |__ Port 1: Dev 2, If 2, Class=CDC Data, Driver=cdc_acm, 12M
    ``` 
  