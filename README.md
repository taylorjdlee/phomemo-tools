# Phomemo-tools

This package aims to provide Linux drivers for Phomemo thermal printers.

All the information here has been reverse-engineered sniffing
the packets emitted by each Phomemo printer.

python3-pybluez and phomemo-tools CUPS driver can be found at
[phomemo-tools RPM](http://vivier.eu/phomemo-tools/)

Phomemo printers that are currently supported are listed below:
* M02
* M02S
* M110
* M120

If you don't see your printer on this list feel free to contribute a USB or Bluetooth dump of your printer!

## 1. Usage

### 1.1. Bluetooth
* Use the commands to below to establish a Bluetooth connection with the device

```
$ hcitool scan
Scanning ...
  DC:0D:30:90:23:C7	Mr.in_M02
$ sudo rfcomm connect 0 DC:0D:30:90:23:C7
  Connected /dev/rfcomm0 to DC:0D:30:90:23:C7 on channel 1
  Press CTRL-C for hangup
```

### 1.2. USB

* Plug the printer into the computer via USB cable

* Check the printer is present:

```
  $ lsusb
  ...
  Bus 003 Device 013: ID 0493:b002 MAG Technology Co., Ltd 
  ...
```
* You can find the serial port via dmesg or inside /dev/usb, in this case it is /dev/usb/lp0:

```
  $ dmesg
  ...
  usb 3-3.7.2: new full-speed USB device number 13 using xhci_hcd
  usb 3-3.7.2: New USB device found, idVendor=0493, idProduct=b002, bcdDevice= 3.00
  usb 3-3.7.2: New USB device strings: Mfr=1, Product=2, SerialNumber=3
  usb 3-3.7.2: Product: USB Virtual COM
  usb 3-3.7.2: Manufacturer: Nuvoton
  usb 3-3.7.2: SerialNumber: A02014090305
  cdc_acm 3-3.7.2:1.0: ttyACM0: USB ACM device
  usblp 3-3.7.2:1.2: usblp0: USB Bidirectional printer dev 13 if 2 alt 0 proto 2 vid 0x0493 pid 0xB002
  $ ls -lrt /dev
  ...
  drwxr-xr-x.  2 root    root         100 Dec  5 17:44 usb
  crw-rw----.  1 root    dialout 166,   0 Dec  5 17:44 ttyACM0
  ...
  $ ls -lrt /dev/usb
  total 0
  crw-------. 1 root root 180, 96 Dec  5 16:46 hiddev0
  crw-------. 1 root root 180, 97 Dec  5 16:46 hiddev1
  crw-rw----. 1 root lp   180,  0 Dec  5 17:44 lp0
```

## 2. CUPS

### 2.1. Installation

```
  $ sudo apt-get update
  $ sudo apt-get -y install cups
  $ cd cups
  $ make
  $ sudo make install
```

### 2.2. Configuration
#### 2.2.1. CLI
##### 2.2.1.1. Bluetooth

A Bluetooth connection use the "phomemo" backend to connect to a Phomemo printer, the example below
shows a connection to an M02 printer via Bluetooth:

```
  $ sudo lpadmin -p M02 -E -v phomemo://DC0D309023C7 \
                           -P /usr/share/cups/model/Phomemo/Phomemo-M02.ppd.gz
```

##### 2.2.1.2. USB

A USB connection will use the /dev/usb/lp0 device to connect to a Phomemo printer, the example below
shows a connection to an M02 printer via USB:

```
  $ sudo lpadmin -p M02 -E -v serial:/dev/usb/lp0 \
                           -P /usr/share/cups/model/Phomemo/Phomemo-M02.ppd.gz
```

##### 2.2.1.3. Check printer options
You can use the following command to check the options for your printer which will list the printer defaults with a "*":

```
  $ lpoptions -d M02 -l
```
##### 2.2.1.4. Printing
You can use the following command to print text using CUPS:

```
  $ echo "This is test"  | lp -d M02 -o media=w50h60 -
```
You can use the following command to print an image using CUPS:

```
  $ lp -d M02 -o media=w50h60 my_picture.png
```
The M110 & M120 printers have support for LabelWithGaps, Continuous and LabelWithMarks media types which can be specified as follows:

```
  $ echo "This is test"  | lp -d M110 -o media=w30h20 -o MediaType=Continuous
```
#### 2.2.2. GUI
##### 2.2.2.1. Pre-requisite

The CUPS backend (phomemo) uses the Python pyBluez to connect to the printer, so
be sure to install the required dependencies, for instance with Fedora:

```
   $ sudo dnf install python3-devel
   $ sudo pip install pybluez
```

SELinux seems to prevent the backend to create a bluetooth socket.
If you have such error message in your syslog:

```
localhost.localdomain cupsd[2659]: Can\'t open Bluetooth connection: [Errno 13] Permission denied
```
You might need to disable SELinux enforcement to allow the backend to run correctly:

```
  $ sudo semanage permissive -a cupsd_t
```

I didn't find a way to define correctly the SELinux rules to allow the backend
to use bluetooth socket without to change the enforcement mode 
(the couple ausearch/audit2allow doesn't fix the problem).

##### 2.2.2.2. Pair the printer

1. Switch on the printer
2. Open the "Settings" window:

![Settings Menu](Pictures/Menu.png)

3. Select the "Bluetooth" Panel:

![Bluetooth Panel](Pictures/Bluetooth-1.png)

4. Select your bluetooth printer (here "Mr.in_M02"):

![Bluetooth Printer](Pictures/Bluetooth-2.png)

5. Your printer must be paired but not connected ("Disconnected"):

![Bluetooth Printer](Pictures/Bluetooth-3.png)

6. Select the "Printers" Panel:

![Printers Panel](Pictures/Printers-1.png)

You'll probably need to unlock it to be able to add a new printer.

Click on "Add a Printer...".

8. Select your printer and click on "Add":

![Printers Panel](Pictures/Printers-2.png)

9. Your printer will appear in the printers list:

![Printers Panel](Pictures/Printers-3.png)

10. Click on the settings menu of the printer and select "Printing Options":

![Printers Panel](Pictures/Printers-4.png)

11. Select "Media Size Label 50mmx70mm" and click on "Test Page":

![Printers Panel](Pictures/Printers-5.png)

12. Check the result:

![Printers Panel](Pictures/Printers-6.jpg)

## 3. Printer Protocols

After dumping packets, it appears Phomemo printers use EPSON ESC/POS Commands.

### 3.1. Protocol for M02

Below is a Bluetooth dump from an M02 printer.

#### 3.1.1. HEADER

```
  0x1b 0x40      -> command ESC @: initialize printer
  0x1b 0x61      -> command ESC a: select justification
  0x01           range: 0 (left-justification), 1 centered,
                        2 (right justification)
  0x1f 0x11 0x02 0x04           
```

#### 3.1.2. BLOCK MARKER
```
  0x1d 0x76 0x30 -> command GS v 0 : print raster bit image
  0x00              mode: 0 (normal), 1 (double width),
                          2 (double-height), 3 (quadruple)
  0x30 0x00         16bit, little-endian: number of bytes / line (48)
  0xff 0x00         16bit, little-endian: number of lines in the image (255)
```
  
  Values seem to be 16bit little-endian

  If the picture is not finished, a new block marker must be sent with
  the remaining number of line (max is 255).

#### 3.1.3. FOOTER
```
  0x1b 0x64      -> command ESC d : print and feed n lines
  0x02           number of line to feed
  0x1b 0x64      -> command ESC d : print and feed n lines
  0x02           number of line to feed
  0x1f 0x11 0x08
  0x1f 0x11 0x0e
  0x1f 0x11 0x07
  0x1f 0x11 0x09
```
#### 3.1.4. IMAGE

  Each line is 48 bytes long, each bit is a point (384 pt/line).
  size of a line is 48 mm (80 pt/cm or 203,2 dpi, as announced by Phomemo).
  ratio between height and width is 1.

#### 3.1.5. Printer message

```
1a 04 5a
1a 09 0c
1a 07 01 00 00
1a 08 
51 30 30 31 45 30 XX XX XX XX XX XX XX XX XX -> Serial Numer: E05C0XXXXXX
```

### 3.2. Protocol for M110/M120

Below is a USB packet dump from a M110 printer, the M120 printer also uses the same protocol. 

#### 3.2.1. HEADER

```
  0x1b 0x4e 0x0d  -> Print Speed
  0x05            range: 0x01 (Slow) -  0x05 (Fast)
  0x1b 0x4e 0x04  -> Print Density 
  0x0f            range: 01 - 0f
  0x1f  0x11      -> Media Type
  0x0a            Mode: 0a="Label With Gaps" 0b="Continuas" 26="Label With Marks"
```

#### 3.2.2. BLOCK MARKER

```
  0x1d 0x76 0x30 -> command GS v 0 : print raster bit image
  0x00              mode: 0 (normal), 1 (double width),
                          2 (double-height), 3 (quadruple)
  0x2b 0x00         16bit, little-endian: number of bytes / line (43)
  0xf0 0x00         16bit, little-endian: number of lines in the image (240)
```

#### 3.2.3. FOOTER

```
  0x1f 0xf0 0x05 0x00 
  0x1f 0xf0 0x03 0x00
```
