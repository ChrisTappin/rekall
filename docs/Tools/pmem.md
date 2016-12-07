---
layout: docs
category: tools
title: The Pmem Memory acquisition suite (Legacy).
author: Michael Cohen <scudette@gmail.com>
---

NOTE: This document refers to the legacy pmem acquisition tools
(pre-2.0). Please check out the new [pmem 2 series](http://www.rekall-forensic.com/docs/Tools) of acquisition
tools.

Memory acquisition is the first step in memory analysis. Before any analysis can
be done, we need to acquire the memory in the first place. There are a number of
commercial solutions to acquire memory, but sadly open source solutions have
been abandoned or not maintained (For example win32dd has been a popular
solution many years ago but has now been commercialized and is no longer open
source).

We believe in open source forensic tools to make testing and transparency
easier. We also believe that the availability of open source solutions spurs
further development in the field and enables choices.

That is the reason we feel an open source, well tested and capable forensic
memory acquisition tool is essential - we call it the Pmem suite of tools. The
pmem acquisition tool aims to provide a complete imaging solution for Windows,
Linux and OSX.

The following is a quick overview of how to use the pmem tools. For detailed
information consult the source.

WinPmem
-------

The windows memory acquisition tool is called WinPmem.

These are the features it supports:

* Supports all windows versions from WinXP SP2 to Windows 8 in both i386 and
amd64 flavours.
* Output formats include:
 * Raw memory images.
 * ELF Core dump files for use in rekall.
 * Output to stdout (in both the above formats) for piping through other tools
   (e.g. ssh, ewfacquirestream etc).

* Memory acquisition using
 * MmMapIoSpace method.
 * \Device\PhysicalMemory and ZwMapViewOfSection method.
 * PTE Remapping technique (default)

* Direct analysis of the running kernel using Rekall (Live memory analysis).
* Optional Write support for manipulating kernel data structures from Rekall.

### Download

The latest version can be found [here](http://www.rekall-forensic.com/). You
will find the tool released in two versions:

- winpmem-1.6.0.exe: is the recommended binary for general use. This binary
  contains signed drivers so it can load on any windows system (even 64 bit
  ones). This binary does not include write support for memory.

- winpmem_write-1.6.0.exe: is the binary with write support enabled. It is not
  signed so it will only work on 32 bit windows or 64 bit windows with special
  preparation (see below).

IMPORTANT: The recommended version for regular use is the one without write
support. The version with write support can not be used on a regular system.

```
c:\..> winpmem_1.6.0.exe -h
Winpmem - A memory imager for windows.
Copyright Michael Cohen (scudette@gmail.com) 2012-2014.

Version 1.6.0 May 15 2014
Usage:
  winpmem_1.6.0.exe [option] [output path]

Option:
  -l    Load the driver and exit.
  -u    Unload the driver and exit.
  -d [filename]
        Extract driver to this file (Default use random name).
  -h    Display this help.
  -w    Turn on write mode.
  -0    Use MmMapIoSpace method.
  -1    Use \\Device\PhysicalMemory method (Default for 32bit OS).
  -2    Use PTE remapping (AMD64 only - Default for 64bit OS).
  -3    Use PTE remapping with PCI instrospection (AMD64 Only).
  -e    Produce an ELF core dump.

NOTE: an output filename of - will write the image to STDOUT.

Examples:
winpmem_1.6.0.exe physmem.raw
Writes an image to physmem.raw

winpmem_1.6.0.exe -e - | nc 192.168.1.1 80
Writes an elf coredump to netcat for network transport.
```

### Examples

Writes a raw image to physmem.raw

    winpmem_1.6.0.exe physmem.raw

Writes a crashdump file to netcat for network transport. Output is supressed
here because STDOUT is redirected.

    winpmem_1.5.2.exe -d - | nc 192.168.1.1 80

Normally the driver will be automatically unloaded after the image is
acquired. To allow Rekall to attach to the raw device for live analysis, we need
to load the driver and exit:

    c:\..> winpmem.exe -l
    Loaded Driver.
    c:\..> rekal -f \\.\pmem

NOTE: Rekall does not usually need a profile when running on a windows image
since it is autodetected.

To unload the driver and exit:

    c:\..> winpmem.exe -u
    Driver Unloaded.

To acquire a raw image using the MmMapIoSpace method:

    c:\..> winpmem_1.5.2.exe -1 myimage.raw


To acquire an image in crashdump format:

```
c:\..>winpmem_1.6.0.exe -d c:\temp\test.dmp
Driver Unloaded.
Loaded Driver C:\Users\mic\AppData\Local\Temp\win6C6.tmp.
Will write a crash dump file
CR3: 0x0000187000
 2 memory ranges:
Start 0x00001000 - Length 0x0009E000
Start 0x00100000 - Length 0x6F6FB000


00% 0x00001000 .

00% 0x00100000 ..................................................
02% 0x03300000 ..................................................
05% 0x06500000 ..................................................
*..
92% 0x67300000 ..................................................
95% 0x6A500000 ..................................................
98% 0x6D700000 .................................
Driver Unloaded.
```

### Experimental write support

As from Version 1.1, the winpmem drivers support writing to memory as well as
reading. This capability is a great learning tool since many rootkit hiding
techniques can be emulated by writing to memory directly. For example the
following Rekall session illustrates changing the name of the binary:


```
c:..> rekal -f \\.\pmem

WinXPSP2x86:pmem 03:10:40> task = session.profile._EPROCESS(0x82079c18)
WinXPSP2x86:pmem 03:10:57> task.ImageFileName
                    Out    [String:ImageFileName]: 'cmd.exe\x00'
WinXPSP2x86:pmem 03:11:15> task.ImageFileName = "foo.exe\x00"
WinXPSP2x86:pmem 03:11:21> task.ImageFileName
                    Out    [String:ImageFileName]: 'foo.exe\x00'
```

Since this is a rather dangerous capability, the signed binary drivers have
write support disabled. The unsigned binaries (really self signed with a test
certificate) can not load on a regular system due to them being test self
signed. You can allow the unsigned drivers to be loaded on a test system by
issuing (see
http://msdn.microsoft.com/en-us/library/windows/hardware/ff553484(v=vs.85).aspx):

    Bcdedit.exe -set TESTSIGNING ON

and reboot. You will see a small "Test Mode" text on the desktop to remind you
that this machine is configured for test signed drivers.

Alternatively you can test this on XP or Vista32 which have no driver signing
restrictions.

Once the correct driver is loaded, Write support must also be enabled at load
time using the -w switch:

    winpmem_write-1.5.2.exe -w -l

This will load the drivers and turn on write support. Then we can run rekall
interactively, as usual on the raw device:

    rekal --profile Win7SP1x64 --file \\.\pmem


OSXPMem - Mac OS X Physical Memory acquisition tool
---------------------------------------------------

The OSX Memory Imager was written by Johannes Stuettgen
(johannes.stuettgen@gmail.com) as an open source tool to acquire physical memory
on an Intel based Mac. It consists of 2 components:

* The usermode acquisition tool 'osxpmem', which parses the accessible sections
  of physical memory and writes them to disk in a specific format.
* A generic kernel extension 'pmem.kext', that provides read only access to
  physical memory. After loading it into the kernel it provides a device file
  ('/dev/pmem/'), from which physical memory can be read.

## Download

The binaries can be found [here](http://www.rekall-forensic.com) or from the
Rekall downloads page.


### Usage

* You need root access for this to work so first open a root shell ('sudo su').
* Now unpack the archive ('tar xvf OSXPMem.tar.gz'). This creates a new
  directory 'OSXPMem' containing the binary 'osxpmem', as well as the
  kernel extension 'pmem.kext'.
* Enter the directory you just created ('cd OSXPMem').
* Run the imager by passing it a file-name for the memory image.
  ('./osxpmem memory.dump' will create a file named 'memory.dump').

The imager supports multiple output formats, at the moment these are Mach-O, ELF
and zero-padded RAW. You can select which output format to use by passing the
'--format' option. For example to write a Mach-O image you would invoke
'./osxpmem --format mach memory.dump'. The default output format is ELF.

For more information on different command line switches run './osxpmem --help'.

### Common Pitfalls

* Mac OS X only allows kernel extension to load if they are owned by the user
  'root' and the group 'wheel'. The distribution package has this already set up
  for you. However, if you accidentally extract the archive as a normal user
  (eg.  omit 'sudo su' before unpacking the tarball), permissions might become
  corrupted and the loading of the driver will fail. In this case you can
  correct the problem by running 'sudo chown -R root:wheel ./pmem.kext' from
  within the 'OSXPMem' directory.

* If you try to run the imager from NFS or another networked file-system,
  permissions might also become corrupted. If the imager reports a failure to
  load the pmem driver, check the drivers permissions. If it is not owned by
  user 'root' and group 'wheel' and step 1 can't correct this, try copying it
  somewhere else and correct permissions there.

### Compatibility

Due to the nature of physical memory access many things are very platform
dependent. The tool is designed to work on 64 bit Intel Macs. It can probably be
compiled to work in 32 bit mode, the binary distribution however only contains
64 bit binaries.

Several low-level api's have changed in recent OS X versions. We have tested the
imager and driver on OS X 10.7 and 10.8, on which they work flawlessly. It
should also work on 10.6, but might encounter problems unloading the driver, as
the unloading api in IOKit is new in 10.7.

We have also successfully tested the tool in a VMWare Fusion OS X 10.7 machine,
so it should work in virtualized environments.
