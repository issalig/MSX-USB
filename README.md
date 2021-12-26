# MSX-USB
## My fork and DIY guide
My fork provides a minified version of the pcb with PLC sockets and a header expansion as well as detailed building guide. All the other things are the same as original S0urceror version (https://github.com/S0urceror/MSX-USB) and I send him lots of thanks for sharing his knowledge.

As I built the cartridge from the scratch I will tell you some of the problems I faced and happily solved out.

### Step 0: Make a PCB
When I saw original S0urceror design I thought that maybe using PLC sockets could make the pcb smaller, then I tried to shrink everything into the minimum space with the maximum capabilities and that's why I added a expansion header. I am using https://www.kicad.org/ for the PCB layout and https://freerouting.org/ for the routing. I find freerouting extremely useful since there are a lot of connections and a small place to do the job. Sometines freerouting does not find the solution and you need to finish it manually. If this happens you can also lower clearance matrix values down to 125-150. Also using freerouting allows you to change the location of components without the need to re-route everything again.

Of course if you are satisfied with my design just can just use the files provided. Otherwise you can improve the design and share it.

### Step 1: Program CPLD
You will need the obsolete part EPM7064SLC44, the important part here is **SLC**, EPM7064SLC44-10 is also valid. You can get them from ali and other places but as they are refurbished/recovered from dumpsters expect to have only 50% working, so order double. Here is a possible link https://aliexpress.com/item/1005001512009032.html but there is a myriad of sellers.
Then, you will need a programmer such as USB blaster. I got this one https://aliexpress.com/item/1005002674193901.html
Next, you have to download Quartus software version **13.0sp1**, as newer versions do not support EPM7064. Go to https://fpgasoftware.intel.com/?edition=lite and proceed with the download and installation.

If you are in linux like me you will  need to add the this rules to file /etc/udev/rules.d/37-usbblaster.rules
```
SUBSYSTEM=="usb", ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6001", GROUP="plugdev", MODE="0666", SYMLINK+="usbblaster" 
```
and then restart the service
```
sudo service udev restart 
```

Connect the programmer to the board and also **power the board externally with 5V**, you can do it using the expansion header or CH3786 header. Find the pins VCC and GND and you are done. Failing to do so it will result in an JTAG read error.

Next, we check your connection and if everything went ok you will see this message.
```
$ ./quartus_pgm --auto
Info (213045): Using programming cable "USB-Blaster [3-1]"
1) USB-Blaster [3-1]
  070640DD   EPM7064S

Info: Quartus II 32-bit Programmer was successful. 0 errors, 0 warnings
    Info: Peak virtual memory: 127 megabytes
    Info: Processing ended: Sat Dec 18 10:16:56 2021
    Info: Elapsed time: 00:00:00
    Info: Total CPU time (on all processors): 00:00:00
```

And it has arrived the amazing moment where you are going to program it (you can do it with the command line *quartus_pgm* or with the GUI tool *quartus*)
with the following file https://github.com/S0urceror/MSX-USB/blob/master/hardware/quartus/output_files/MSXUSB.pof
```
./quartus_pgm -m jtag -o "p;./MSXUSB.pof"
```

In case the devices still shows JTAG communication error, keep calm and go to the next step.

### Step 1b. Recovering deaf JTAG units
It happens that some of the refurbished units come with programs that used JTAG pins for I/O and they are not able to be used for JTAG. I threw some of them to the bin until I discovered a recovery process (https://www.eevblog.com/forum/fpga/atmel-atf150x-cpld-and-wincupl/msg3162224/#msg3162224).
- Apply 12V to CPLD pin OE1 (44) with a 2k2 resistor for a minute or so. In my pcb you can access OE1 pin on header pin 15 (RST).
- Disconnect 12V, connect 5V and try to program it.
- Hopefully some CPLD's are working now! 

### Step 2. Flash memory
Recommended memory chips is SST39SF040 (512kb) but I also managed to make it work with SST39SF020 (256kb). Then, you will need a programmer such as TL866 II https://aliexpress.com/item/33000308958.html and the file we will write is located at https://github.com/S0urceror/MSX-USB/blob/master/drivers/NextorUsbHost/dist/nextor.rom

If you prefer to use a command line tool I suggest you minipro https://gitlab.com/DavidGriffith/minipro

```
$ ./minipro --device SST39SF040@PLCC32 --write nextor.rom -s
Found TL866II+ 04.2.126 (0x27e)
Warning: Firmware is newer than expected.
  Expected  04.2.124 (0x27c)
  Found     04.2.126 (0x27e)
Chip ID OK: 0xBFB7
Warning: Incorrect file size: 131072 (needed 524288)
Erasing... 0.40Sec OK
Writing Code...  26.38Sec  OK
Reading Code...  4.00Sec  OK
Verification OK

```

### Step 3. Select a valid USB drive
Not all USB drives will work :(, I have detected that the ones with the following EnpointAddress values usually work. It will probably work if you have the values shown below. If you find other values, please report.
```
lsusb -v | grep bEndpointAddress
...
bEndpointAddress 0x01 EP 1 OUT
bEndpointAddress 0x82 EP 2 IN 
```

### Step 4. Prepare a disk
Once we are here, all the hardware requirements are fulfilled, yes, you can go for a beer now!
It is only a matter of creating a disk image with the system files provided at https://github.com/S0urceror/MSX-USB/tree/master/software/dist

- If you have a MSX1 (i.e. VG8020, ...) you will be limited to 720kb disks and FAT12. Do not complain, do you still prefer a slow datacorder?
To create a MSX disk image I use dsktool from https://github.com/nataliapc/MSX_devs/tree/master/dsktool

I also provide linux and windows version of dsktool in order to make it easier, you can get them at https://github.com/issalig/MSX-USB/tree/master/software/utils
```
#create disk
dsktool c MYDISK.DSK
#add system files
dsktool a MYDISK.DSK COMMAND.COM MSXDOS.SYS
#add other files
dsktool a MYDISK.DSK MYGAME.COM
```

Next, flash it. **Warning: check if your device is /dev/sda** or you will destroy something. If you are a windows user maybe Rufus can write the image, if it works, please report it. 

```
sudo dd if=MYDISK.DSK of=/dev/sda
```

- If you are a lucky MSX2 owner you will need a FAT16 partition of 16Mb with COMMAND2.COM and NEXTOR.SYS. You can do it with gparted in linux, as usual report how is it done with windows.

Unplug the stick from the computer, plug it in the msxusb, switch on the MSX and you are in. **Enjoy!**

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

