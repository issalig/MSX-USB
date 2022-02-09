# Installing OpenMSX
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
You can also find roms at https://www.planetemu.net/rom/msx-bios/ 

Then, you should include the ch376s device in the .xml which resides in /opt/openMSX/share/software or ~/.openMSX/share/software
In particular, add the following code in the ```<devices>...</devices>``` section.
```
<CH376s id="ch376s">
      <io base="0x10" num="1" type="IO"/>
      <io base="0x11" num="1" type="IO"/>
</CH376s>
```

Also make sure the rom file is found, maybe you need to check the rom section. Here is how it looks for VG8020.
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
$ openmsx -machine Philips_VG_8020 -carta nextor.rom
failed to open port
```

If everything is ok you will see sth like
```
$ openmsx -machine Philips_VG_8020 -carta ../MSX-USB/drivers/MsxUsbNext/msx/dist/nextor.rom 
writeCommand (CH_CMD_RESET_ALL) 0x05
writeCommand (CH_CMD_CHECK_EXIST) 0x06
writeData (0xbe)
0x41 = readData ()
```

If it hangs in readData that usually means pin definition or connections in the Arduino are not ok.
