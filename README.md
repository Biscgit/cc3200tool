# CC3200 Tool v1.2.2

A small tool to read and write files in TI's CC3200 SimpleLink (TM) filesystem.
Partial support for the CC32XX (CC3230/CC3235) series (via flash image)

Copyright (C) 2016-2020 Allterco Robotics

Copyright (C) 2020- Team RevvoX (0xbadbee, g3gg0)

![](https://img.shields.io/badge/license-GPL_2-green.svg "License")

## Fork by Biscgit

This fork is for fixing an issue with the loading of DDL files when running directly with `python3 cc.py` without having
to install the programm.
On the current version running the tool with the above command on Linux fails to load the DDL files.
If you have encountered the following error (with traceback):

```
File ".../cc3200tool/cc.py", line 846, in _raw_write
    while sent < len(data):
    
TypeError: object of type 'NoneType' has no len()
```

This fork should fix the issue.

### The encountered (possible) issue

The package's root path defined in the function `dll_data(fname)`
(`cc3200tool/cc.py`, line 312) for loading the DDL modules is not correct when running the `cc.py` file directly.
On not finding the file, the function `pkgutil.get_data(package, resource)` returns `None` which cannot be handled
correctly in the `_raw_write` function (`cc3200tool/cc.py`, line 832) and raises an Exception.
Since the other method of running did not work, we tried to run it directly and encountered this error (tested with
python 3.12 on Linux 6.9.1).
It is likely that the issue is not present when running with the shown method (which did not work for us).

### Use as Library
You can now import this and use as library. 
Simply pass the command to the main() function and it will do the rest.

```
main(args: list[str] = None, console: typing.Any = None, p_path: str = None)
```

Pass your normal command as a list of arguments in `args`, your custom logger as `console` and the file's path 
relative to where the module gets imported in python syntax as `p_path`.

On failing a command it will raise an Exception of type `ExitException` with the corresponding error code instead of exiting the program.
You can now define the location from where the ddl files get loaded.
You can also pass custom loggers for example to use with rich.

## Rationale

The only other tool which can officially do this is Uniflash, but good luck
using that on a modern Linux system.

There is also `cc3200prog` which [Energia](http://energia.nu) sneak in their toolchain tarball,
but it's not open source and can only write to `/sys/mcuimg.bin`.

Finally, there's the great guys at [Cesanta](https://www.cesanta.com/)
who reversed the CC3200 bootloader
protocol just enough to make day-to-day development on the platform possible.
However, their tool is specific to [smart.js](https://www.cesanta.com/products/smart-js)
and feeds on specially-crafted zip archives.

This tool is based on the work done by Cesanta but is written in Python and
exposes a generic cli interface for uploading user files and application binaries
on a CC3200-attached serial flash.

`cc3200tool` can upload NWP/MAC/PHY firmwares (`/sys/servicepack.ucf`), but it seems
this only works on a clean FS. The tool also implements the functionality
described in TI's Application Note [CC3100/CC3200 Embedded Programming](http://www.ti.com/tool/embedded-programming).

## Installation

This runs on Python >=3.6 with recent [pySerial](https://github.com/pyserial/pyserial).

To install, if you have pip and want system-wide:

    pip install git+git://github.com/toniebox-reverse-engineering/cc3200tool.git

or clone this repǫ

    git clone http://github.com/toniebox-reverse-engineering/cc3200tool.git
    cd cc3200tool
    pip3 install .

then it's just like any other python package:

    python setup.py install # as root, system-wide

    # or in a virtualenv with pip
    virtualenv env && ./env/bin/activate
    pip install -e .
    # then get updates with
    git pull

## Usage

You need a serial port connected to the target's UART interface. For
programming to work, SOP2 needs to be asserted (i.e. tied to VCC) and a reset
has to be peformed to switch the chip in bootloader mode. `cc3200tool` can
optionally use the RTS and DTR lines of the serial port controller to
automatically perform these actions via the `--sop2` and `--reset` options.

See `cc3200tool -h` and `cc3200tool <subcommand> -h` for complete description
of arguments. Some examples:

    # upload an application
    cc3200tool -p /dev/ttyUSB2 --sop2 ~dtr --reset prompt \
        write_file ./exe/myapp.bin /sys/mcuimg.bin

    # format and upload an application binary
    cc3200tool -p /dev/ttyUSB2 \
        format_flash --size 1M \
        write_file exe/program.bin /sys/mcuimg.bin

    # dump a file on stdout
    cc3200tool read_file /sys/mcuimg.bin -

    # format the flash, upload a servciepack and two files
    cc3200tool -p /dev/ttyUSB2 --sop2 ~rts --reset dtr \
        format_flash --size=1M \
        write_file --file-size=0x20000 \
            --signature ../servicepack-ota/ota_1.0.1.6-2.6.0.5.ucf.signed.bin \
            ../servicepack-ota/ota_1.0.1.6-2.6.0.5.ucf.ucf /sys/servicepack.ucf \
        write_file ../application_bootloader/gcc/exe/application_bootloader.bin /sys/mcuimg.bin \
        write_file yourapp.bin /sys/mcuimg1.bin

    # list file and filesystem statistics (occupied and free block sequences)
    cc3200tool -p /dev/ttyUSB2 list_filesystem

    # Reads all files to a directory and creates subdirecty structure
    cc3200tool -p /dev/ttyUSB2 read_all_files extract/

    # Writes all files from a directory and its subdirectories (add --simulate to skip writing)
    cc3200tool -p /dev/ttyUSB2 write_all_files extract/

    # list filesystem from a flashdump
    cc3200tool -if cc3200-flash.bin list_filesystem

    # list filesystem from a cc3230/cc3235 flashdump
    cc3200tool -if cc32xx-flash.bin -d cc32xx list_filesystem

    # dump a file from a flashdump
    cc3200tool -if cc3200-flash.bin read_file /sys/mcuimg.bin

    # Reads all files to a directory and creates subdirecty structure from a flashdump
    cc3200tool -if cc3200-flash.bin read_all_files extract/

    # Replace the existing ca.der on the flash (only works if the new file isn't bigger than the old one, experimental!)
    cc3200tool -if cc32xx-flash.bin -of cc32xx-flash.custom.bin -d cc32xx write_file customca.der /cert/ca.der

