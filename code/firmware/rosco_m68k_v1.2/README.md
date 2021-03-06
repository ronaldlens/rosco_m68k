# rosco_m68k Firmware v1.2
## For all Revision 1.2 rosco_m68k main boards

This is the official firmware for the revision 1.2 rosco_m68k. It provides
a means of loading code at boot time via Kermit or from SD card. 

This firmware performs basic initialization of the machine, including
the MFP and V9958 video board (where fitted). Additionally, it provides a 
variety of basic input/output routines and runtime support via standard 
TRAP function interfaces (See InterfaceReference_v1.2.md for details).

## Building

The build supports building 16KB ROMs with **either** Kermit or SD Card
support - there isn't space to support both. The basic configuration
includes Kermit only (for backward compatibility).

Regardless of ROM size, the basic (TRAP 14) runtime support and Easy68k
compatibility will always be available. VDP support will be available
by default, but can be omitted.

### 16KB ROMs

To build the basic firmware:

```
make clean all
```

This will build a 16K-targeted ROM (plus odd/even pair) with VDP support,
Kermit loading and runtime support.

To enable SD Card support (experimental), pass `WITH_SDFAT=true`. In a 
16KB ROM, this will fail unless you also omit either Kermit or VDP 
support as there is insufficient space in the ROM.

For example:

```
WITH_SDFAT=true WITH_KERMIT=false make clean all
```

Will build a 16KB ROM with SD Card load and VDP support, but no kermit, or:

```
WITH_SDFAT=true WITH_VDP=false make clean all
```

Will build a 16KB ROM with SD Card and Kermit load, but no VDP support.

### 64KB ROMs

To build 64KB ROMs with all options included, set `BIGROM=true`, e.g:

```
BIGROM=true make clean all
```

When building a 64KB ROM you can still optionally include and exclude 
things with the options listed above.

### Burning

If you are using a TL866II+ programmer, you can burn your ROMs 
directly from the Makefile with:

```
make clean burn
```

This will automatically set the device based on the BIGROM setting to
either AT28C64B or AT28C256. This can be overriden by passing 
`ROMDEVICE` on the command line, e.g:

```
ROMDEVICE=<SOMEDEVICE> make clean burn
``` 

Where `<SOMEDEVICE>` is a Minipro-recognised device string.

## SD Card 

Documentation TODO

## Serial Loader (Kermit)

### Protocols

Currently, only Kermit is supported, and only in the `robust` mode. To support
cautious or fast mode would probably need hardware flow control, for which
most FTDI adapters implement inadequate support.

This is mildly annoying as it requires one to install additional software 
to get Kermit support from the command line, or in minicom. On most Unices
you should be able to install from your package manager (try `ckermit` or 
`c-kermit`). On OSX you can install from homebrew (`brew install c-kermit`) 
and ports probably has it too. On Windows YMMV, but if you're successfully
building this on Windows then you've probably already done a lot of work to
get a cross-compiler and so on working, so I'm sure you'll figure it out.

### Kermit Settings

It is **strongly** recommended you use the following settings for your
c-kermit. These should go in your `~/.kermrc` file (or whatever the 
Windows equivalent is):

```
set carrier-watch off ;
set flow xon/xoff ;
robust
```

This instructs c-kermit not to look for a carrier detect signal, to use
software flow-control (because all FTDI chips I've seen get RTS/CTS 
just wrong enough to make them useless without a FIFO on the receiving 
end), and to use the `robust` kermit protocol variant, which is the only 
one I've been able to get to work with the embedded Kermit implementation 
in the bootloader - YMMV of course, feel free to experiment...

You'll need this (at least the carrier detect disable) whether you're
using c-kermit from the command-line or from minicom (or similar).

Of course if your FTDI adapter has a CD pin and you tie this low, you
can skip the carrier-detect setting...
 
## Code

TODO There will eventually be some documentation here on how to write and
build code that is compatible with this loader. For now, these notes
will have to suffice:

* **There is a very simple POC "kernel" at ../poc-kernel** - This shows how to
  e.g. relocate your code (down to $1000) and how to link for that etc.
* Code is loaded at $40000 (somewhat arbitrarily). The loader will jump
  directly to that location after the code is received.
* This means you are limited to ~860KB with the standard memory configuration.
* (It's actually slightly more, but the stack is at the top of RAM!)
* Once your code is loaded, all of RAM is yours.
* Depending how much setup you want to do, you might leave the lowest 1KB 
  alone (exception vectors), other than setting up any vectors your code
  actually wants to handle.
* If you want to use the standard runtime support stuff (see below), leave the
  bottom 4KB alone (i.e. $1000-$FFFFF are free for your use). 
* The _recommended_ thing to do is to relocate your code after loading,
  so you don't have it stuck in the middle of RAM. This will make your
  life easier later.
* Obviously your link script will need to take this into account!
* On entry to your code, you are free to (and probably should) reset the
  stack, and can trash any registers you wish. 
* The loader **does not** expect to be returned to. It _will_ handle
  such a condition gracefully, however (it will print a message and halt
  the machine).
  
On entry to the loaded code, the system will be in the following state:

* CPU will be in supervisor mode
* PC will be at $40000
* VBR will point to $0
* Supervisor stack will be at $100000, SSP could be anywhere and can be reset
* Registers will be undefined, and can all be trashed.
* Exception table will be set up (with mostly no-op handlers) from $0 - $3FF 
* Some (very) basic system data will exist between $400-$4FF. You can trash this if you don't need it
  * Some of the default exception handlers **do** write to this area, however!
  * So replace them if you're going to use this area for your own purposes!
* Interrupts will be enabled 
* Bus error, address error and illegal instruction will have default handlers (that flash I1 1, 2 or 3 times in a loop)
* MFP Timer C will be driving a 100Hz system tick
* System tick will be vectored to CPU vector 0x45, default handler flashes I0.
* MFP Interrupts other than Timer C will be disabled
* TRAP#14 will be hooked by the firmware to provide some basic IO (see next section)
* UART TX and RX will be enabled, but their interrupts won't be 
  * you'll either have to enable (and handle) them or use polling
  * If you _do_ enable them, don't use the TRAP#14 IO routines any more or sadness is likely to ensue.
* CTS (MFP GPIO #7) **will be low** (i.e. asserted). 

## Runtime Support

See InterfaceReference_v1.2.md for full details of the runtime interfaces,
TRAPs and memory layout supported by this firmware, including the 
Easy68k compatibility layer.

