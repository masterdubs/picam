# picam Compilation Guide

This document describes how to build picam. The whole process takes about 2-3 hours with a standard Linux box.

## Binary releases are also available

For those who don't want to build picam yourself, standalone binary is available at https://github.com/iizukanao/picam/releases/latest

Also, picam preinstalled Raspbian SD card image is available at <http://s.kyu-mu.net/raspbian-picam/>. Write it to an SD card larger than 4.5GB, boot it, then run `raspi-config` to expand the filesystem.

To build picam yourself, continue reading.


## Install crosstool-ng

First we need to install crosstool-ng on a powerful Linux machine (not Raspberry Pi) to be able to cross-compile ffmpeg.

    $ mkdir ~/pi
    $ cd ~/pi
    $ wget http://crosstool-ng.org/download/crosstool-ng/crosstool-ng-1.21.0.tar.xz
    $ tar xvf crosstool-ng-1.21.0.tar.xz
    $ cd crosstool-ng-1.21.0
    $ ./configure --prefix=/opt/cross
    $ make
    $ sudo make install
    $ export PATH=$PATH:/opt/cross/bin
    $ cd ..
    $ mkdir ctng
    $ cd ctng
    $ ct-ng menuconfig

Then a configuration screen appears. The followings are for crosstool-ng 1.21.0, and it may not be applicable to later versions.

- Paths and misc options
    - Check "Try features marked as EXPERIMENTAL"
    - Set "Prefix directory" to "/opt/cross/x-tools/${CT_TARGET}"
- Target options
    - Set "Target Architecture" to "arm"
    - Set "Endianness" to "Little endian"
    - Set "Bitness" to "32-bit"
    - Set "Architecture level" to "armv6zk"
    - Set "Emit assembly for CPU" to "arm1176jzf-s"
    - Set "Tune for CPU" to "arm1176jzf-s"
    - Set "Use specific FPU" to "vfp"
    - Set "Floating point" to "hardware (FPU)"
    - Set "Default instruction set mode" to "arm"
    - Check "Use EABI"
- Toolchain options
    - Set "Tuple's vendor string" to "rpi"
- Operating System
    - Set "Target OS" to "linux"
- Binary utilities
    - Set "Binary format" to "ELF"
    - Set "binutils version" to "2.22"
- C-library
    - Set "C library" to "glibc"
    - Set "glibc version" to "2.19"
- C compiler
    - Check "Show Linaro versions"
    - Set "gcc version" to "linaro-4.9-2015.03"
    - Check "C++" under "Additional supported languages"
    - Set "gcc extra config" to "--with-float=hard"
    - Check "Link libstdc++ statically into the gcc binary"
- Companion libraries
    - Set "ISL version" to "0.12.2"

Save the configuration and build.

    $ sudo chown -R $USER /opt/cross
    $ ct-ng build
    $ export CCPREFIX="/opt/cross/x-tools/arm-rpi-linux-gnueabihf/bin/arm-rpi-linux-gnueabihf-"


## Build fdk-aac

We will use $HOME/pi/ as a working directory and put compiled binaries and header files in $HOME/pi/build/. Set the environment variable $PIBUILD.

    $ export PIBUILD=$HOME/pi/build
    $ mkdir $PIBUILD
    $ cd ~/pi

Download fdk-aac-0.1.4.tar.gz (or the latest version) from http://sourceforge.net/projects/opencore-amr/.

    $ tar zxvf fdk-aac-0.1.4.tar.gz
    $ cd fdk-aac-0.1.4
    $ CC=${CCPREFIX}gcc CXX=${CCPREFIX}g++ ./configure --host=arm-rpi-linux-gnueabi --prefix=$PIBUILD
    $ make
    $ make install


## Build ffmpeg

You can't use ffmpeg that can be installed via `apt-get` or `pacman`.

### Copy ALSA headers and libs from Raspberry Pi (for Raspbian users)

Log in to Raspberry Pi and install **libasound2-dev** and **libssl-dev** via `apt-get`.

    $ sudo apt-get install libasound2-dev libssl-dev

Return to the compiling machine. Set an environment variable $PIUSR, and copy ALSA headers and libs from Raspberry Pi.

    $ export PIUSR=$HOME/pi/usr
    $ mkdir $PIUSR
    $ cd $PIUSR
    $ mkdir include lib
    $ rsync -rav pi@raspberrypi:/usr/include/alsa/ $PIUSR/include/alsa/
    $ rsync -rav pi@raspberrypi:/usr/lib/arm-linux-gnueabihf/libasound.so* $PIUSR/lib/
    $ tree --charset=ascii
    .
    |-- include
    |   `-- alsa
    |       |-- alisp.h
    :       :   :
    |       |-- sound
    |       |   |-- asound_fm.h
    :       :   :   :
    |       |   `-- type_compat.h
    |       |-- timer.h
    |       |-- use-case.h
    |       `-- version.h
    `-- lib
        |-- libasound.so -> libasound.so.2.0.0
        |-- libasound.so.2 -> libasound.so.2.0.0
        `-- libasound.so.2.0.0

    4 directories, 39 files


### Copy ALSA headers and libs from Raspberry Pi (for Arch Linux users)

Log in to Raspberry Pi and install **alsa-lib** and **alsa-utils** via `pacman`.

    $ sudo pacman -S alsa-lib alsa-utils

Return to the compiling machine. Set an environment variable $PIUSR, and copy ALSA headers and libs from Raspberry Pi.

    $ export PIUSR=$HOME/pi/usr
    $ mkdir $PIUSR
    $ cd $PIUSR
    $ mkdir include lib
    $ rsync -rav pi@raspberrypi:/usr/include/alsa/ $PIUSR/include/alsa/
    $ rsync -rav pi@raspberrypi:/usr/lib/libasound.so* $PIUSR/lib/
    $ tree --charset=ascii
    .
    |-- include
    |   `-- alsa
    |       |-- alisp.h
    :       :   :
    |       |-- sound
    |       |   |-- asound_fm.h
    :       :   :   :
    |       |   `-- type_compat.h
    |       |-- timer.h
    |       |-- use-case.h
    |       `-- version.h
    `-- lib
        |-- libasound.so -> libasound.so.2.0.0
        |-- libasound.so.2 -> libasound.so.2.0.0
        `-- libasound.so.2.0.0

    4 directories, 39 files


### Compile ffmpeg

Fetch ffmpeg source and configure it:

    $ cd ~/pi
    $ git clone git://source.ffmpeg.org/ffmpeg.git
    $ cd ffmpeg
    $ PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$PIBUILD/lib/pkgconfig CC=${CCPREFIX}gcc CXX=${CCPREFIX}g++ ./configure --enable-cross-compile --cross-prefix=${CCPREFIX} --arch=armel --target-os=linux --prefix=$PIBUILD --extra-cflags="-I$PIBUILD/include -I$PIUSR/include -fPIC" --extra-ldflags="-L$PIBUILD/lib -L$PIUSR/lib -fPIC" --enable-libfdk-aac --pkg-config=`which pkg-config`

In the output of `configure`, make sure that:

- "Enabled indevs" has "alsa"
- "Enabled encoders" has "libfdk_aac"

If all goes well, build ffmpeg.

    $ make
    $ make install


## Transfer the files to Raspberry Pi

Transfer the contents of $PIBUILD directory to Raspberry Pi.

    $ rsync -rav $PIBUILD/ pi@raspberrypi:build/

Then, put those files into /usr/local/ so that you have /usr/local/bin/ffmpeg.

    (on Raspberry Pi)
    $ cd ~/build
    $ ls
    bin  include  lib  share
    $ sudo rsync -rav ./ /usr/local/

We have done with the powerful machine which we used for cross compiling.

If you don't have /etc/ld.so.conf.d/libc.conf on Raspberry Pi, create that file with the following contents. On Raspbian, /etc/ld.so.conf.d/libc.conf is installed by default.

    /usr/local/lib

Run `ldconfig`.

    $ sudo ldconfig

Run `ffmpeg -codecs | grep fdk` and make sure that the output has this line:

     DEA.L. aac                  AAC (Advanced Audio Coding) (decoders: aac libfdk_aac ) (encoders: aac libfdk_aac )


## Build libilclient

From now on, all steps are performed on Raspberry Pi.

    $ cd /opt/vc/src/hello_pi/libs/ilclient
    $ make

If you do not have /opt/vc/src/ directory, download the firmware from https://github.com/raspberrypi/firmware and put all the contents under /opt/vc/src/. On Raspbian, /opt/vc/src/ is installed by default.


## Install dependencies

Install **libfontconfig1-dev** and **libharfbuzz-dev** via `apt-get`.

    $ sudo apt-get install libfontconfig1-dev libharfbuzz-dev


## Build picam

    $ cd (to anywhere you like)
    $ git clone https://github.com/iizukanao/picam.git
    $ cd picam
    $ make

If you want to save some disk space, strip the binary.

    $ strip picam

Check that picam runs without errors.

    $ ./picam --help


## Configuration for Arch Linux

If you are using Arch Linux, you need to have these two lines in /boot/config.txt.

    start_file=start_x.elf
    fixup_file=fixup_x.dat

Also, assign at least 128MB for GPU memory.

    gpu_mem_512=128

Reboot the Raspberry Pi for the changes to take effect.
