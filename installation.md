# Setting up the development environment
It is possible and more than recommendable to use openMSX for development and testing. Thus, you do not need to flash roms and you can try it on different machines. For this task you will also need and arduino UNO or similar which will be the bridge between PC and CH376s (ESP8266 will not work, because it has not enough pins but ESP32 should do, any volunteer? )


# Compiling rom
In this section I will focus on the MsxUsbNext rom but this is not a problem because requirements for other roms will be almost the same. Code is located at https://github.com/S0urceror/MSX-USB/tree/master/drivers/MsxUsbNext/macos

MsxUsbNext has several parts, macos which is a version to test from the computer (it is also valid for linux and maybe for windows with cygwin but not wsl2 which does not support usbs afaik), and msx which is the version that we will flash on the cartridge, ie., the rom.

First, you will need to install g++, clang, sjasm, and sdcc.
```
sudo apt-get install g++ clang sjasm sdcc
```
hex2bin is also necessary and you can get it from https://github.com/ericb59/Fusion-C-v1.2/tree/master/Working%20Folder/Tools/Hex2bin-2.5

Also you will need mknexrom. The one provided is for MacOS but if you are not in MacOS, just build or download mknexrom from https://github.com/Konamiman/Nextor/blob/v2.1/buildtools/linux/mknexrom and copy it to https://github.com/S0urceror/MSX-USB/tree/master/drivers/MsxUsbNext/msx/kernel 

Once here, you should have all the required tools in order to build the rom.

Noe, go to macos directory, create a build dir and make it!
```
cd macos
mkdir build
make
```
Take into account that you will probably need to edit ```hal.c``` file and set your usb device name. For example: ```char device[] = "/dev/ttyUSB0";```.

The built executable performs the same operations that will be done in rom but allows you to test it in a more confortable environment.



# Arduino
The skecth needed is found at https://github.com/S0urceror/MSX-USB/blob/master/hardware/arduino/noobtocol.parallel/noobtocol.parallel.ino and it is basically a bridge between the PC and the ch376s. Thus openMSX will open the serial port send some bytes and the arduino will redirect them to ch376s, get the answer if any and send it to the PC.

You may probably need to change the pins and below I show you the ones I used. I recommend you not to use pin 13 (LED pin) because it can cause problems. D0 is pin 5, D1 pin 6 and so on ...
```
//CH376 pins to Arduino digital connections mapping
const int CH_WR = 4;
const int CH_RD = 3;
const int CH_PCS = A0;
const int CH_A0 = A5;
const int CH_INT = 2;
const int CH_D0 = 5;
//CH_Dx = CH_D0 + x
```

Now flash the arduino and we are going to check the skecth with this tool https://github.com/S0urceror/MSX-USB/blob/master/test/usb_via_arduino/src/main.parallel.cpp

If you are in linux you need to add ```#include <assert.h>``` and remove ```-stdlib=libc++``` from the Makefile (https://stackoverflow.com/questions/19774778/when-is-it-necessary-to-use-the-flag-stdlib-libstdc)
Compile this tool ```make main_parallel``` and run it like ``` parallel /dev/ttyUSB0``` (of course change it according to your device)

If everything is ok you will see readData is getting values
```
$ ../dist/main_parallel /dev/ttyUSB0  | more
12:02:48:0001 writeCommand (CH_CMD_RESET_ALL) 0x05
12:02:48:0100 writeCommand (CH_CMD_CHECK_EXIST) 0x06
writeData (0xbe)
0x41 = readData ()
...
```

If this works openMSX will be able to talk to the ch376s, isn't that amazing?


# Compiling openMSX
In order to use ch376s you will need S0urceror's fork of OpenMSX which you can get at https://github.com/S0urceror/openMSX
```
git clone https://github.com/S0urceror/openMSX
```
Then ```./configure```
and if everything went well (you have all the libraries requiered) you should see something like this

```
$ ./configure 
Using Python: python3
Probing target system...
Up to date: derived/x86_64-linux-opt/config/probed_defs.mk
Up to date: derived/x86_64-linux-opt/config/systemfuncs.hh

Found libraries:
  ALSA:             version 1.2.2
  GLEW:             version unknown
  libogg:           version unknown
  libpng:           version 1.6.37
  libtheora:        version unknown
  libvorbis:        version unknown
  OpenGL:           version 4.6
  SDL2:             version 2.0.10
  SDL2_ttf:         version 2.0.15
  Tcl:              version 8.6.10
  zlib:             version 1.2.11

Components overview:
  Emulation core:   yes
  GL renderer:      yes
  Laserdisc:        yes
  ALSA MIDI:        yes

Customisable options:
  Install to        /opt/openMSX
  (you can edit these in build/custom.mk)

All required and optional components can be built.

If the detected libraries differ from what you think is installed on this system, please check the log file: derived/x86_64-linux-opt/config/probe.log
```

If some libraries are not found or installed they will be reported and you can also have a look at  derived/x86_64-linux-opt/config/probe.log for details.

In particular, under Ubuntu 20.04 I had to install these packages but you may need to install additional ones. In such a case, please report.

```
sudo apt-get install libasound2-dev libglew-dev libogg-dev libtheora-dev libvorbis-dev libsdl2-dev libsdl2-ttf-dev tcl-dev zlib1g-dev
```

Next step is to compile the sources ```make -j 8``` and after some minutes and if everything went file you will have a fresh compilation. Then ```sudo make install``` and you are done.

```
$ sudo make install
Using Python: python3
Build configuration:
  Platform: x86_64-linux
  Flavour:  opt
  Compiler: g++
  Subset:   full build
Up to date: derived/x86_64-linux-opt/config/Version.ii
Installing openMSX:
  Executable...
  Data files...
  Documentation...
  C-BIOS...
  Creating symlinks...
Installation complete... have fun!
Notice: if you want to emulate real MSX systems and not only the free C-BIOS machines, put the system ROMs in one of the following directories: /opt/openMSX/share/systemroms or ~/.openMSX/share/systemroms
If you want openMSX to find MSX software referred to from replays or savestates you get from your friends, copy that MSX software to /opt/openMSX/share/software or ~/.openMSX/share/software
```

Now it is time to set up your machine. You will need to add the system rom to /opt/openMSX/share/systemroms or ~/.openMSX/share/systemroms  
You can find get from Fusion-C project https://github.com/ericb59/Fusion-C-v1.2/tree/master/Working%20Folder/Tools/_For%20OpenMSX/systemroms and in particular I will use https://github.com/ericb59/Fusion-C-v1.2/blob/master/Working%20Folder/Tools/_For%20OpenMSX/systemroms/machines/philips/vg8020_basic-bios1.rom
You can also find other roms at https://www.planetemu.net/rom/msx-bios/ 

Now copy the .xml machine description to your your local configuration 
```
cp  /opt/openMSX/share/machines/Philips_VG_8020.xml ~/.openMSX/share/machines/Philips_VG_8020_MSXUSB.xml
```
Open the file .xml and include the ch376s device by addind the following lines in the ```<devices>...</devices>``` section.
```
<CH376s id="ch376s">
      <io base="0x10" num="1" type="IO"/>
      <io base="0x11" num="1" type="IO"/>
</CH376s>
```

**Warning:** If you modify the files in /opt/openMSX/share/machines/ and you recompile and install (make install) your modifications will be lost, that's why it is better to use ~/.openMSX/share/.

Also make sure the rom file is found, maybe you need to check the rom section. Here is how it looks for VG8020 and you should provide vg8020_basic-bios1.rom on r ~/.openMSX/share/systemroms
```
<primary slot="0">
      <ROM id="MSX BIOS with BASIC ROM">
        <rom>
          <filename>vg8020_basic-bios1.rom</filename>
          <sha1>829c00c3114f25b3dae5157c0a238b52a3ac37db</sha1>
        </rom>
        <mem base="0x0000" size="0x8000"/>
      </ROM>
    </primary>
```

## Go on
As a default, the serial device is located at /dev/tty.usbmodem123451 and you can of course change it in the source code but I prefer to do a symbolic link ``` sudo ln -s ttyUSB0 /dev/tty.usbmodem123451``` . If your arduino takes other port, like ttyACM0, just change it.
 
 If there are problems to open the port you will get this message, otherwise port will be found.
 ```
$ openmsx -machine Philips_VG_8020_MSXUSB -carta nextor.rom
failed to open port
```

If everything is ok you will see sth like
```
$ openmsx -machine Philips_VG_8020_MSXUSB -carta ../MSX-USB/drivers/MsxUsbNext/msx/dist/nextor.rom 
writeCommand (CH_CMD_RESET_ALL) 0x05
writeCommand (CH_CMD_CHECK_EXIST) 0x06
writeData (0xbe)
0x41 = readData ()
```

If it hangs in readData that usually means pin definition or connections in the Arduino are not ok. Also it can be some problems with the usb port.
