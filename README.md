# MSX-USB
## My fork and DIY guide
I provide a minified version of the pcb with PLC44 sockets and a header expansion. All the other things are the same as original S0urceror version and I send him a great thanks for sharing.

As I built the cartridge from the scratch I will tell you some of the problems I faced and solved.

### Step 1: Program CPLD
You will need the obsolete part EPM7064SLC44, the important part is SLC, EPM7064SLC44-10 is also valid. You can get them from ali and other places but as they are refurbished expect to have only 50% working, so order double.
Then, you will need a programmer such as USB blaster. I got this one https://es.aliexpress.com/item/1005002674193901.html
Also you will need to download Quartus software, newer version do not support EPM7064, so you need version

If you are in linux like me you will  need to add the needed rules to file /etc/udev/rules.d/37-usbblaster.rules
```
SUBSYSTEM=="usb", ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6001", GROUP="plugdev", MODE="0666", SYMLINK+="usbblaster" 
```
and then
```
sudo service udev restart 
```

If everything went ok 
```
./quartus_pgm --auto
```

Now it is time to program it (you can do it with the command line or with the GUI)
BUT, wait a moment, in order to program it you need to power it externally with 5V (yes, you can use expansion header and connect 5V and GND there)

```
./quartus_pgm -m jtag -o "p;./MSXUSB.pof"
```

### Step 1b. Recovering deaf JTAG units
It happens that some of the refurbished units came with programs that used JTAG pins for I/O and they are not able to communicate. I threw some of them to the bin until I discovered this process (https://www.eevblog.com/forum/fpga/atmel-atf150x-cpld-and-wincupl/msg3162224/#msg3162224).
By applying 12V to CPLD pin OE1 (44) with a 2k2 resistor for a minute I was able to restore JTAG communication. It did not work for all but some CPLD's are working now! In my pcb you can access OE1 pin on header pin 15 (RST)

### Step 2. Flash memory
You will need a programmer such as TL866 II 
Also it seem to be posible to program it from the MSX itself but I have not done this yet. I will let explore this option (please read original S0urceror version).
https://github.com/S0urceror/MSX-USB/blob/master/drivers/NextorUsbHost/dist/nextor.rom

### Step 3. Select a valid USB drive
Not all USB drives will work, I have detected that the ones with the following EnpointAddress values will work. It will probably work if you have the values shown below
```
lsusb -v | grep bEndpointAddress
...
bEndpointAddress 0x01 EP 1 OUT
bEndpointAddress 0x82 EP 2 IN 
```

### Step 4. Prepare a disk
If you have a MSX1 (i.e. VG8020, ...) you will be limited to 720kb disks and FAT12.
To create a MSX disk image I use dsktool from https://github.com/nataliapc/MSX_devs/tree/master/dsktool
```
#create disk
dsktool c MYDISK.DSK
#add files
dsktool a MYDISK.DSK COMMAND.COM MSXDOS.SYS
```
Next, flash it. **Warning: check if your device is /dev/sda**
```
dd if=MYDISK.DSK of=/dev/sda
```

If you are a lucky MSX2 owner you will need COMMAND2.COM and NEXTOR.SYS instead and you will not need for FAT12 partition.


## Some history
In the 80's the MSX standard was created by Kazuhiko Nishi. He envisioned a simple yet powerful home computer based on standard hardware, software (BIOS/MSX DOS) and connectivity. From 1983 to 1990 manufacturers, like Philips, Sony, Yamaha, Sharp, Canon and many more, made computers according to this standard.

Because of the fact that extensibility was part of the standard people could design cartridges that would add synthesizer sound, memory, modems, rom-based games, or combinations of the above.

Cartridges were connected via a standard 50pin connector that remained the same accross 4 generations of MSX (1/2/2+/TurboR). BIOS and MSX-DOS were designed in a way to discover these via rom-based drivers, software hooks, extended bios.

Standardisation offers a big advantage because everything that adheres to the standard can connect to it.

## USB
USB was designed in the 90's to do something similar. To standardize the connection of peripherals to personal computers, both to communicate with and to supply electric power. It has now largely replaced interfaces such as serial ports and parallel ports, and has become commonplace on a wide range of devices. Like USB flash drives, keyboards, mice, wifi, serial, camera's, etc.

What if the MSX would have an USB connection? And if the BIOS/MSX DOS would automatically recognize devices connected to it?

## Meet MSX-USB
Inspired by the work of Xavirompe and his [RookieDrive](http://rookiedrive.com/en/) I set out to do exactly this. And since I'm a big believer of open standards and open source I'm opening up my work to everyone.

On this GitHub page you can find:
* Schematics (KiCAD), that interfaces a cheap USB CH376s board to the MSX cartridge port.
* Drivers, currently Flash drives on Nextor and HID keyboards are supported.
* Debugging tools, to connect a test board to your PC/MAC and assist in debugging.

On my GitHub I also have forks of OpenMSX and CocoaMSX that either emulate or connect to a test board via serial to assist in development and debugging.

## Schematics
This connects the CH376s board you can find on eBay or AliExpress for around 4 Euro to the MSX. 

Circuitry is added to correctly handle the _BUSDIR signal. Make sure you select parallel by setting the jumper on the CH376s board correctly.

The CH376s board will be visible on port 10h and 11h on the MSX. Port 11h is the command port and 10h is the data port. Specifics on how to program for the CH376s are scattered around the Internet:
* https://www.mpja.com/download/ch376ds1.pdf
* https://arduinobasics.blogspot.com/2015/05/ch376s-usb-readwrite-module.html

Most information is available on how to use the higher-order API for flash drives. 

If you want to use other USB devices you have to go low-level. As it turns out Konamiman already did some work there and after some researching I finally got USB HID keyboards working as well. Check out my source-code or these great pages for more information:
* http://www.usbmadesimple.co.uk/index.html
* https://www.beyondlogic.org/usbnutshell/usb1.shtml

## Flash Drive
For the flash drive I created a NEXTOR driver that looks for a file called NEXTOR.DSK on the root of the flash drive at startup. I initially used 720Kb images but I discovered that I can use bigger FAT16 images as well. Now I'm using a 128Mb image and it works good. 

I created some routines in MSX-Basic to show the flash directory itself and to eject and insert images. Since the CH376s is capable of using FAT32 you can have multiple FAT16 images on one flash drive. I settled for 128Mb FAT16 because of cluster size and optimum storage use.

## UNAPI USB
I wrote a UNAPI USB specification and implemented the Usb Host driver according to this. The next version of this Host driver will also implement the Usb Hub specification and enumerate and initialise all devices connected.

## USB HID Keyboard
The Usb Keyboard driver connects to Unapi Usb driver and hooks itself to H.CHGE. From that moment on it replaces your trusted MSX keyboard by a shiny new USB Keyboard. Or a wireless one if you have inserted the appropriate Logitech receiver.

## USB Ethernet
The USB Ethernet driver is finished. It uses the Unapi USB and conforms to the Unapi Ethernet standard. Internestor Lite can now connect and use your USB Ethernet device. Please note that currently we only support USB CDC ECM. Make sure your Ethernet device supports this.

All USB Ethernet devices built around the **RTL8153** chipset support USB CDC ECM. They usually cost around 20 euro.

# Installation instructions / downloads
Check [this page](INSTALL.md) for installation instructions and links to the various binaries that have been developed.

# Collaboration
Do you want to help with the development of MSX USB? Write drivers for other devices? Or contribute in other ways?

Please drop me line on sourceror at neximus.nl or meet me on the msx.org forum.

