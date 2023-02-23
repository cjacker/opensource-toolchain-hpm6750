# Opensource toolchain for hpm6750 from HPMicro Semiconductor

HPMicro Semiconductor is a leading manufacturer of high-performance embedded solutions, and Andes Technology, a leading supplier of 32/64-bit RISC-V embedded processors.

HPM6750, adopts dual RISC-V AndesCore™ D45 cores, is equipped with innovative bus architecture, efficient level-1 caches and Local Memory, and has set a new performance record of over 9000 CoreMark and 4500 DMIPS, with a main frequency of up to 800 MHz. It provides robust computing power for edge computing and other applications.

The whole series of HPM6000 MCUs, including dual-core HPM6750, single-core HPM6450, and entry-level HPM6120, are all equipped with double-precision floating-point operations and powerful DSP extension instructions, built-in 2 MB SRAM, rich multimedia functions, motor control modules, communication interface and security encryption. The HPM6000 series can be used widely in popular applications such as industry 4.0, smart home appliances, payment terminal, edge computing, and IoT.

In short, up to now, hpm6750 (816 MHz Dual Core) is **the most powerful real-time RISC-V mcu in the world**.

<img src="https://github.com/cjacker/opensource-toolchain-hpm6750/raw/main/evkmini.jpg" />

# Hardware prerequiest
- hpm6750evk / hpm6750evkmini board
  + these boards can be aquired from [HPM official website](http://www.hpmicro.com)
  + hpm6750 do **NOT have on-chip flash**, but evk/evkmini has 8M external FLASH and 16M DRAM integrated  on board.
- CMSIS-DAP / JTAG adapters
  - if no one integreated on board
  - official evk board has ft2232 integrated.

# Toolchain overview
- Compiler: riscv 32bit gnu toolchain
- SDK: [hpm_sdk](https://github.com/hpmicro/hpm_sdk)
- Programming: OpenOCD
- Debugging: OpenOCD / gdb

# RISC-V GNU Toolchain

[xpack-dev-tools](https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/) provde a prebuilt toolchain for riscv. you can download it from https://github.com/xpack-dev-tools/riscv-none-elf-gcc-xpack/. The lastest version is '12.2.0', Download and extract it:

```
sudo mkdir -p /opt/xpack-riscv-toolchain
sudo tar xf xpack-riscv-none-elf-gcc-12.2.0-3-linux-x64.tar.gz -C /opt/xpack-riscv-toolchain --strip-components=1
```

And add `/opt/xpack-riscv-toolchain/bin` to PATH env according to your shell.

**NOTE**, the target triplet of xpack riscv toolchain is **`riscv-none-elf`**.

# SDK

HPMacro proide a well organized, cmake based SDK, the sdk is release under 'BSD-3' license, and no doubt, it is completely open source.

## Get the SDK:
```
git clone https://github.com/hpmicro/hpm_sdk
```

## Change toolchain settings of the SDK

Since we use xpack riscv toolchain here, the settings of toolchain of the sdk should be modified, open `cmake/toolchain.cmake` in your favorite editor, and find:
```
elseif("${TOOLCHAIN_VARIANT}" STREQUAL "gcc")
  set(COMPILER gcc)
  set(LINKER ld)
  set(BINTOOLS gnu)
  set(C++ g++)
  set(CROSS_COMPILE_TARGET riscv32-unknown-elf)
  set(SYSROOT_TARGET       riscv32-unknown-elf)
  set(TOOLCHAIN_CMAKE gcc.cmake)
```
Note the `riscv32-unknown-elf`, it should change to the triplet of xpack riscv toolchain, aka, change it to `riscv-none-elf`.

## Setup some env variables:

hpm_sdk need:
- declare a env variable "HPM_SDK_BASE" point to the path of SDK root
- declare a env variable "GNURISCV_TOOLCHAIN_PATH" point to the toolchain top dir.

For example, for bash:
- append `export HPM_SDK_BASE=<path to sdk>` to `~/.bashrc` 
- append `export GNURISCV_TOOLCHAIN_PATH=/opt/xpack-riscv-toolchain` to `~/.bashrc`


If you do not use bash, please setup these 2 variables according to your shell.

## Build rgb_led sample:
After all setup, enter 'samples/rgb_led' dir:

```
mkdir build
cd build
cmake .. -DBOARD=hpm6750evkmini -DCMAKE_BUILD_TYPE=flash_xip
make
```

As mentioned above, hpm6750 do not have on-chip flash, but evkmini board has 8M external flash and 16m dram integrated on board. 'hpm_sdk' use `-DCMAKE_BUILD_TYPE` to use these resources. 

- without `-DCMAKE_BUILD_TYPE`, code will load to internal ram
- with `-DCMAKE_BUILD_TYPE=flash_xip`, code will be load to 8M flash
- with `-DCMAKE_BUILD_TYPE=flash_sdram_xip`, it is suitable for code need to allocate large memory space, such as deal with 'framebuffer'.

**NOTE 1 :** If you use hpm6750evk board, change the '-DBOARD=hpm6750evkmini' to '-DBOARD=hpm6750evk'.


After build successfully, the target file "demo.elf/bin" will be generated at `output` dir.

# Programming

HPM6750 is not supported by upstream OpenOCD up to now due to lack of support for 'hpm_xpi' flash driver. The forked OpenOCD by HPMacro should be used to program HPM6750.

## HPM OpenOCD installation
```
git clone https://github.com/hpmicro/riscv-openocd
cd riscv-openocd
./bootstrap
./configure --prefix=/usr --datadir=/usr/share/hpm-openocd --disable-werror --program-prefix=hpm-
make 
sudo make install
```

After install successfully, the `hpm-openocd` command should already in your PATH.

## Programming with hpm-openocd

As mentioned aove, you should use `-DCMAKE_BUILD_TYPE=flash_xip` when build the sample codes. After build successfully, program the target device as:

```
 hpm-openocd -f $HPM_SDK_BASE/boards/openocd/probes/ft2232.cfg \
  -f $HPM_SDK_BASE/boards/openocd/soc/hpm6750-single-core.cfg \
  -f $HPM_SDK_BASE/boards/openocd/boards/hpm6750evkmini.cfg \
  -c "program output/demo.elf verify reset exit"
```

If no error happened, the output looks like:
```
...
Info : JTAG tap: hpm6750.cpu tap/device found: 0x1000563d (mfg: 0x31e (Andes Technology Corporation), part: 0x0005, ver: 0x1)
clocks has been enabled!
SDRAM has been initialized
** Programming Started **
Info : Flash write discontinued at 0x80001090, next section at 0x80003000
Warn : Adding extra erase range, 0x80000000 .. 0x800003ff
Warn : Adding extra erase range, 0x80001090 .. 0x80001fff
Warn : Adding extra erase range, 0x8000fe48 .. 0x8000ffff
** Programming Finished **
** Verify Started **
** Verified OK **
** Resetting Target **
...
```

# Debugging

It is very slow to load code into flash for debugging. please rebuild rgb_led sample without `-DCMAKE_BUILD_TYPE` and let code load into ram.

## Re-build rgb_led sample:
Enter 'samples/rgb_led' dir:

```
rm -rf build
mkdir build
cd build
cmake .. -DBOARD=hpm6750evkmini
make
```
After build successfully, the target file 'demo.elf' will be generated in 'output' dir.

## Launch hpm-openocd
For single core debugging:
```
 hpm-openocd -f $HPM_SDK_BASE/boards/openocd/probes/ft2232.cfg \
  -f $HPM_SDK_BASE/boards/openocd/soc/hpm6750-single-core.cfg \
  -f $HPM_SDK_BASE/boards/openocd/boards/hpm6750evkmini.cfg
```
The debugserver is listen on port 3333:
```
Info : starting gdb server for hpm6750.cpu0 on 3333
Info : Listening on port 3333 for gdb connections
```

For dual core debugging:
```
 hpm-openocd -f $HPM_SDK_BASE/boards/openocd/probes/ft2232.cfg \
  -f $HPM_SDK_BASE/boards/openocd/soc/hpm6750-dual-core.cfg \
  -f $HPM_SDK_BASE/boards/openocd/boards/hpm6750evkmini.cfg
```

The debugserver is listen on port 3333 for CPU0, and port 3334 for CPU1:
```
Info : starting gdb server for hpm6750.cpu0 on 3333
Info : Listening on port 3333 for gdb connections
Info : starting gdb server for hpm6750.cpu1 on 3334
Info : Listening on port 3334 for gdb connections
```

## Debugging with gdb
Start another terminal window, run:
```
riscv-none-elf-gdb output/demo.elf
```
And input below commands after `(gdb)` promt show up:

```
(gdb) target remote :3333
Remote debugging using :3333
0x80003000 in ?? ()
(gdb) load
Loading section .start, size 0x36 lma 0x0
Loading section .vectors, size 0x600 lma 0x200
Loading section .text, size 0xc658 lma 0x800
Loading section .data, size 0x1d0 lma 0xce58
Start address 0x00000000, load size 52830
Transfer rate: 259 KB/sec, 7547 bytes/write.
(gdb) b main
Breakpoint 1 at 0x3634: file /home/cjacker/hpm_sdk-1.0.0/samples/rgb_led/src/rgb_led.c, line 190.
(gdb) c
Continuing.

Breakpoint 1, main () at /home/cjacker/hpm_sdk-1.0.0/samples/rgb_led/src/rgb_led.c:190
190         board_init();
(gdb)
```
