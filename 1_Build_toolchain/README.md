# Toolchain
Toolchain is the first thing that you need to start your embedded project.
It includes tools to compile, link your code, and libraries,...

A standard GNU toolchain consists of:
- Binutils
- GCC
- C library

There are 2 types of toolchains:
- Native
- Cross

# CPU architectures
The toolchain has to be built according to the capabilities of the target CPU:
- CPU architecture
- Big- or little-endian operation
- Floating point support
- Application Binary Interface (ABI)

# Choosing the C library
**glibc**
musl libc
uClibc-ng
eglibc

# Finding a toolchain

If you truly want to do the whole thing yourself, take a look at Cross Linux From Scratch (http://trac.clfs.org)
Or easier, use crosstool-NG


# Building a toolchain using crosstool-NG
## Installing crosstool-NG

Ubuntu 12.04 

$ sudo apt-get install automake bison chrpath flex g++ git gperf \
gawk libexpat1-dev libncurses5-dev libsdl1.2-dev libtool \
python2.7-dev texinfo

$ git clone https://github.com/crosstool-ng/crosstool-ng.git

$ cd crosstool-ng

$ git checkout crosstool-ng-1.22.0

$ ./bootstrap

$ ./configure --enable-local

$ make

$ make install

-> output is something and a file named *ct-ng*

## Building a toolchain for BeagleBone Black
The BeagleBone Black has a TI AM335x SoC, which contains an ARM Cortex A8 core and a VFPv3 floating point unit
./ct-ng list-samples

./ct-ng show-arm-cortex_a8-linux-gnueabi

./ct-ng arm-cortex_a8-linux-gnueabi

./ct-ng menuconfig

In Paths and misc options, disable Render the toolchain read-only (CT_INSTALL_DIR_RO)

In Target options | Floating point, select hardware (FPU) (CT_ARCH_FLOAT_HW)

Save and 

./ct-ng build

outout -> ~/x-tools/arm-cortex_a8-linux-gnueabihf

## Building a toolchain for QEMU

On the QEMU target, you will be emulating an ARM-versatile PB evaluation board that has
an ARM926EJ-S processor core, which implements the ARMv5TE instruction set

./ct-ng list-samples

./ct-ng distclean

./ct-ng arm-unknown-linux-gnueabi


./ct-ng menuconfig

In Paths and misc options, disable Render the toolchain read-only (CT_INSTALL_DIR_RO)

./ct-ng build

output -> ~/x-tools/arm-unknown-linux-gnueabi

**Note**
Some tarballs are not available when you build the toolchain

So you need to download them manually to [path-to/]crosstool-ng/.build/tarballs

You may never find isl-0.14 on internet, so I put in this git repo under the folder crosstool-ng/tarballs.

When building, isl-0.14.tar.gz may not be able to be extracted. You need to extract manually to  [path/to/]crosstool-ng/.build/src/isl-0.14 and comment out 
the command which is used to extract that tarball (int [/path/to/]crosstool-ng/scripts/build/companion_libs/121-isl.sh)

[INFO ]  Build completed at 20220405.180701

[INFO ]  (elapsed: 23:05.08)

[INFO ]  Finishing installation (may take a few seconds)...


Somethings you need to discover...

PATH=~/x-tools/arm-unknown-linux-gnueabi/bin:$PATH

arm-unknown-linux-gnueabi-gcc helloworld.c -o helloworld

arm-unknown-linux-gnueabi-gcc --version

arm-unknown-linux-gnueabi-gcc -v

- --with-sysroot=/home/chris/x-tools/arm-unknown-linux-gnueabi/arm-unknown-linux-gnueabi/sysroot:
- --enable-languages=c,c++
- --with-cpu=cortex-a8
- --with-float=hard
- --enable-threads=posix

 –-with-cpu
arm-unknown-linux-gnueabi-gcc -mcpu=cortex-a5 helloworld.c -o helloworld

arm-unknown-linux-gnueabi-gcc --target-help

arm-unknown-linux-gnueabi-gcc -print-sysroot

- lib
- usr/lib
- usr/include
- usr/bin
- use/share
- sbin

**ld-linux in lib - are needed on the target at runtime when using dynamic libraries**

## Static libraries

export SYSROOT=$(arm-unknown-linux-gnueabi-gcc -print-sysroot)

arm-unknown-linux-gnueabi-gcc -c test1.c

arm-unknown-linux-gnueabi-gcc -c test2.c

arm-unknown-linux-gnueabi-ar rc libtest.a test1.o test2.o

-> arm-unknown-linux-gnueabi-gcc helloworld.c -ltest -L../libs -I../libs -o helloworld

## Shared libraries
arm-unknown-linux-gnueabi-gcc -fPIC -c test1.c

arm-unknown-linux-gnueabi-gcc -fPIC -c test2.c

arm-unknown-linux-gnueabi-gcc -shared -o libtest.so test1.o test2.o

-> arm-unknown-linux-gnueabi-gcc helloworld.c -ltest -L../libs -I../libs -o helloworld
**Note: The runtime linker for this program is /lib/ld-linux-armhf.so.3, which must be present in the target's filesystem**
The linker will look for libtest.so in the default search
path: /lib and /usr/lib. If you want it to look for libraries in other directories as well,
you can place a colon-separated list of paths in the shell variable LD_LIBRARY_PATH:

export LD_LIBRARY_PATH=/opt/lib:/opt/usr/lib


You will usually see some lib likes:
- libjpeg.a: This is the library archive used for static linking
- libjpeg.so -> libjpeg.so.8.0.2: This is a symbolic link, used for dynamic linking
- libjpeg.so.8 -> libjpeg.so.8.0.2: This is a symbolic link, used when loading the library at runtime
- libjpeg.so.8.0.2: This is the actual shared library, used at both compile time and runtime

The first two are needed on the host for building and the last two are needed on the target

# The art of cross compiling
There are some common build systems, including:
- Pure makefiles, where the toolchain is usually controlled by the make variable CROSS_COMPILE
- The GNU build system known as Autotools
- CMake (https://cmake.org/)


## Simple makefiles

Some important packages are very simple to cross compile, including the Linux kernel, the U-Boot bootloader, and BusyBox you only need to put the toolchain prefix in the make variable CROSS_COMPILE, for example arm-cortex_a8-linux-gnueabi-. **Note the trailing dash -**.

make CROSS_COMPILE=arm-unknown-linux-gnueabi-

or 

export CROSS_COMPILE=arm-unknown-linux-gnueabi-

make

## Autotools

Packages that use Autotools come with a script named **configure** that checks dependencies and generates makefiles according to what it finds.

The configure script may also give you the opportunity to enable or disable certain features. You can find the options on offer by running ./configure --help.

./configure

make

sudo make install

Autotools is able to handle cross development as well. You can influence the behavior of the configure script by setting these shell variables:
- CC
- CFLAGS
- LDFLAGS
- LIBS
- CPPFLAGS
- CPP
ex: CC=arm-unknown-linux-gnueabi-gcc ./configure

Autotools understands three different types of machines that may be involved when compiling a package
- build: the computer that build the package
- host: the computer that the program will run on, default: current host
- target: the computer the program will generate code for, you would set this **when building a cross compiler**

to cross compile, you just need to override the host

CC=arm-unknown-linux-gnueabi-gcc ./configure --host=arm-unknown-linux-gnueabi


The default install directory is <sysroot>/usr/local/*

You would usually install it in <sysroot>/usr/*, so that the header files and libraries would be picked up from their default locations

The complete command to configure a typical Autotools package is as follows:

CC=arm-unknown-linux-gnueabi-gcc \

./configure --host=arm-unknown-linux-gnueabi --prefix=/usr

make

make DESTDIR=$(arm-unknown-linux-gnueabi-gcc -print-sysroot) install

it will be installed in DESTDIR/usr/...

## Package configuration

pkg-config (https://www.freedesktop.org/wiki/Software/pkg-config/) helps track which
packages are installed and which compile flags each needs by keeping a database of
Autotools packages in [sysroot]/usr/lib/pkgconfig

cat $(arm-unknown-linux-gnueabi-gcc -print-sysroot)/usr/lib/pkgconfig/sqlite3.pc

\# Package Information for pkg-config

prefix=/usr

exec_prefix=${prefix}

libdir=${exec_prefix}/lib

includedir=${prefix}/include

Name: SQLite

Description: SQL database engine

Version: 3.8.11.1

Libs: -L${libdir} -lsqlite3

Libs.private: -ldl -lpthread 

Cflags: -I${includedir}


export PKG_CONFIG_LIBDIR=$(arm-unknown-linux-gnueabi-gcc -print-sysroot)/usr/lib/pkgconfig

pkg-config sqlite3 --libs --cflags

->  -lsqlite3

-> now we can build:

arm-unknown-linux-gnueabi-gcc $(pkg-config sqlite3 --cflags --libs) \

sqlite-test.c -o sqlite-test

# Problems with cross compiling
- Home-grown build systems has a configure script, but it does not behave like the Autotools configure described in the previous section
- Configure scripts that read pkg-config information, headers, and other files from the host, disregarding the --host override
- Scripts that insist on trying to run cross compiled code



References:
- Mastering embedded linux programming



