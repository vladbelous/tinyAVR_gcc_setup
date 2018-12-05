Notes to help with building/setting up GCC toolchain for the AVR target, and specifically making it work with the new tinyAVR 1-series microcontrollers.

Attiny1614 is used as an example. It is a tinyAVR 1-series MCU, classified in GCC under avrxmega3 architecture.

Recent GCC versions support it as a target, but current avr-libc (2.0.0) does not (yet?), which requires the workaround described here.

With obivous modifications (in AVR-Libc section), this should *probably* work with other tinyAVR 1-series, like:
* attiny212, attiny214,
* attiny412, attiny414, attiny416, attiny417,
* attiny814, attiny816, attiny817,
* attiny1614, attiny1616, attiny1617,
* attiny3214, attiny3216, attiny3217.

# AVR GCC cross compilation + avrdude toolchain:
Start by chosing the installation destination.

Used only during build/install:
```
export PREFIX=/usr/local/avr  # or some other dir
```

Needed for build/install and to invoke avr-gcc (obviously):
```
export PATH=$PATH:$PREFIX/bin
```

## Binutils:
Download (www.gnu.org/software/binutils/), unpack and enter into directory, then:
```
mkdir avr-obj
cd avr-obj
../configure --prefix=$PREFIX --target=avr --disable-nls

make
sudo make install
```
Note: here an onwards, `sudo` is only needed because of /usr/local/... destination (the `$PREFIX` variable). If it's user-writable, don't use `sudo`.

## GCC:
Download (www.gnu.org/software/gcc/), unpack and enter into directory, then:
```
mkdir avr-obj
cd avr-obj
../configure --prefix=$PREFIX --target=avr --enable-languages=c,c++ \
    --disable-nls --disable-libssp --with-dwarf2

make
sudo make install
```

## AVR-Libc
Download (http://download.savannah.gnu.org/releases/avr-libc/),
unpack and enter into directory, then:

```
./configure --prefix=$PREFIX --build=`./config.guess` --host=avr
make
sudo env "PATH=$PATH" make install
```

Notice the **extra 'env'** -- install script needs to run avr-ranlib, which is found
only under the new custom $PATH, which is (by default) not passed by sudo.

At this point you should be able to use avr-gcc toolchain for older MCUs like attiny85, attiny841, atmega328p, etc.
Next sections are for the newer tinyAVR 1-series.

### Atmel/Microchip ATtiny_DFP Pack
Microchip does not make it easy. Instead of them simply providing a full avr-libc to build, we have to hack things together.
Hopefully "upstream" avr-libc will soon support tinyAVR 1-series (like attiny1614), and this could be skipped.

Get Atmel/Microchip ATtiny_DFP Pack (try http://packs.download.atmel.com/)
It is called something like "Atmel.ATtiny_DFP.1.3.229.atpack". It's really just a zip archive with .atpack extention.

Unpack and enter into directory, then we must copy the following
files from Atmel pack:
gcc/dev/attiny1614/avrxmega3/libattiny1614.a,
gcc/dev/attiny1614/avrxmega3/crtattiny1614.o,
include/avr/iotn1614.h:
```
sudo mkdir $PREFIX/avr/lib/avrxmega3
sudo cp gcc/dev/attiny1614/avrxmega3/libattiny1614.a $PREFIX/avr/lib/avrxmega3/
sudo cp gcc/dev/attiny1614/avrxmega3/crtattiny1614.o $PREFIX/avr/lib/avrxmega3/
sudo cp include/avr/iotn1614.h $PREFIX/avr/include/avr/
```

You should see similar structure with other (supported) micros, e.g. compare:
```
find $PREFIX -iname '*841*'
find $PREFIX -iname '*1614*'
```

### Atmel/Microchip libm/libc
We also need avrxmega3-compatible libc.a and libm.a (avr-libc 2.0.0 does not have it).

Get Atmel/Microchip "avr8-gnu-toolchain-..."
(e.g. http://distribute.atmel.no/tools/opensource/Atmel-AVR-GNU-Toolchain/3.6.1/avr8-gnu-toolchain-osx-3.6.1.495-darwin.any.x86_64.tar.gz or later), unpack and enter directory, then:
```
sudo cp avr/lib/avrxmega3/libm.a $PREFIX/avr/lib/avrxmega3/
sudo cp avr/lib/avrxmega3/libc.a $PREFIX/avr/lib/avrxmega3/
```

### TODO
TODO: Check differences in device-specs/specs-attiny1614 between "vanilla" GCC
and Atmel/Microchip toolchains, specifically wrt
```
 --defsym=__RODATA_PM_OFFSET__=0x8000
```
being present in Atmel/Microchip's, but not "vanilla" GCC.
Test if memory-mapped flash actually works.


## avrdude (not tested)
Download, unpack and enter into directory, then:
```
mkdir avr-obj
cd avr-obj
../configure --prefix=$PREFIX

make
sudo make install
```

# Example of usage

Compile and link:
```
avr-gcc -c -Os -mmcu=attiny1614 main.cpp
avr-gcc -mmcu=attiny1641 -o main.elf main.o
```

If you want to check section sizes:
```
avr-size main.elf
```

If you want to check disassembled output:
```
avr-objdump -S main.elf > main.asm
```

Create hex for uploading (e.g. with avrdude):
```
avr-objcopy -j .text -j .data -O ihex main.elf main.hex
```

# TODO: firmware uploading
avrdude and jtag2updi stuff...
