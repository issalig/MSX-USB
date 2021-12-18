# MSX-USB
## Mi fork y guía de construcción
Mi fork proporciona una versión reducida de la pcb con zócalos PLC, una cabecera de expansión, así como una guía de construcción detallada. Todas las demás cosas son iguales a la versión original de S0urceror y desde aquí le mando muchas gracias por compartir sus conocimientos.

Ya que he construido el cartucho desde cero, os contaré algunos de los problemas que encontré y que felizmente he resuelto.
Si ya tienes el hardware funcionando puede saltar directamente al paso 4.

### Paso 1: Programar CPLD
Se nececista el componente obsoleto EPM7064SLC44, la parte importante aquí es **SLC**, EPM7064SLC44-10 también es válido. Se pueden obtener de ali y otros lugares, pero como se reciclan/recuperan de los contenedores de basura, es normal que sólo funcionen el 50%, así que pide el doble. Aquí dejo un enlace posible https://aliexpress.com/item/1005001512009032.html pero hay una gran cantidad de vendedores.
A continuación, se necesita un programador USB Blaster como este https://aliexpress.com/item/1005002674193901.html y se debe descargar la versión **13.0sp1** del software Quartus, ya que las versiones más recientes no son compatibles con EPM7064. Vamos a https://fpgasoftware.intel.com/?edition=lite descargamos e instalamos.

Si estás en Linux como yo, es necesario añadir estas reglas al archivo /etc/udev/rules.d/37-usbblaster.rules
```
SUBSYSTEM=="usb", ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6001", GROUP="plugdev", MODE="0666", SYMLINK+="usbblaster" 
```
y después hay que reiniciar el servicio
```
sudo service udev restart 
```

Conecta el programador a la placa msxusb y también **alimenta la placa externamente con 5V**, puedes hacerlo usando el cabezal de expansión o el cabezal CH376. Encuentra los pines VCC y GND y listo. Si no haces esto se producirá un error de lectura de JTAG.

A continuación, verificamos la conexión y si todo está correcto, aparecerá este mensaje.
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

Y ha llegado el apasionante momento donde lo vamos a programar (se puede hacer con la línea de comando *quartus_pgm* o con la herramienta GUI *quartus*)
con el siguiente archivo https://github.com/S0urceror/MSX-USB/blob/master/hardware/quartus/output_files/MSXUSB.pof
```
./quartus_pgm -m jtag -o "p;./MSXUSB.pof"
```

En caso de que los dispositivos sigan mostrando un error de comunicación JTAG, mantén la calma y continúa con el siguiente paso.

### Paso 1b. Recuperación de unidades JTAG sordas
Puede ocurrir que algunas de las CPLDs vengan con programas que usaban pines JTAG para E/S y por lo tanto no se pueden usar para JTAG. Tiré algunos de ellos a la asura hasta que descubrí un proceso de recuperación (https://www.eevblog.com/forum/fpga/atmel-atf150x-cpld-and-wincupl/msg3162224/#msg3162224).
- Aplicar 12V al pin CPLD OE1 (44) con una resistencia de 2k2 durante aproximadamente un minuto. En mi pcb puede acceder al pin OE1 en el pin 15 del encabezado (RST).
- Desconectar 12V, conectar 5V e intentar programarlo.
- ¡Posiblemente algunos CPLD funcionarán ahora!

### Paso 2. Memoria flash
El chip de memoria recomendado es SST39SF040 (512kb) pero también puede funcionar con SST39SF020 (256kb). A conituanicón, se necesita un programador TL866 II https://aliexpress.com/item/33000308958.html y el archivo que escribiremos se encuentra en  https://github.com/S0urceror/MSX-USB/blob/master/drivers/NextorUsbHost/dist/nextor.rom

Si prefieres usar una herramienta de línea de comandos, sugiero minipro https://gitlab.com/DavidGriffith/minipro

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

### Paso 3. Selecciona una unidad USB válida
No todas las unidades USB funcionarán :(, he detectado que las que tienen los siguientes valores de EnpointAddress generalmente funcionan. Probablemente funcionará si tiene los valores que se muestran a continuación. Si encuentras otros valores que funcionan me alegraré de que me lo comuniques.
```
lsusb -v | grep bEndpointAddress
...
bEndpointAddress 0x01 EP 1 OUT
bEndpointAddress 0x82 EP 2 IN 
```

### Paso 4. Prepara un disco
Una vez que hemos llegado aquí, ya se cumplen todos los requisitos de hardware, sí, **¡puedes ir a tomar una birra ahora!**
Lo que queda es solo cuestión de crear una imagen de disco con los archivos del sistema proporcionados en https://github.com/S0urceror/MSX-USB/tree/master/software/dist

- Si tienes un MSX1 (p. ej., VG8020, ...) está limitado a discos de 720kb y FAT12. No te quejes, ¿sigues prefiriendo el lento lector de cassettes?
Para crear una imagen de disco MSX utilizo dsktool de https://github.com/nataliapc/MSX_devs/tree/master/dsktool
```
#create disk
dsktool c MYDISK.DSK
#add system files
dsktool a MYDISK.DSK COMMAND.COM MSXDOS.SYS
#add other files
dsktool a MYDISK.DSK MYGAME.COM
```

Y ahora vamos a flashearlo. **Advertencia: comprueba si tu dispositivo es /dev/sda** o vendrán los lloros. Si eres un usuario de Windows, tal vez Rufus pueda escribir la imagen, si funciona, cuéntanoslo.


```
sudo dd if=MYDISK.DSK of=/dev/sda
```

- Si eres un afortunado propietario de MSX2, necesitarás una partición FAT16 de 16Mb con COMMAND2.COM y NEXTOR.SYS. Puedes hacerlo con gparted en linux, como siempre informa cómo se hace con windows.

Desconecta el pendrive del pc, conéctalo al msxusb, enciende el MSX y listo. 
**¡Disfruta!** 
