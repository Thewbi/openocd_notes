# openocd_notes
Notes on openocd

This document explains how to compile openocd on windows using MINGW 64 bit.

Firstly, the latest commit of openocd will not compile with MINGW.
It is necessary to use an older commit such as checkout commit SHA 420f637 - 23.Dez 2024
The other versions have automate m4 errors

If is necessary to compile without the option to convert warnings to errors:
```
../configure --enable-ftdi --enable-internal-jimtcl --disable-werror
```

It is possible to get MINGW into a situation where you have installed so many incompatible packages
that the entire thing is not able to compile openocd any more. If that happens, deinstall MINGW
and reinstall it from scratch.

A working gcc compiler is installed with
```
pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-toolchain
```

Do not use
```
pacman -S gcc
```
because the plain gcc package will not be able to find the libusb library even when it is installed correctly.

The compiled binary is located in the build/src folder.
To run it on a stm8 devboard (First replace the default driver used for the STLink by the WinUSB driver using Zadig. 
Only then will libusb/openocd be able to talk to the board:

```
cd /c/Users/lapto/dev/c/openocd
./build/src/openocd.exe -d3 -f ./tcl/interface/stlink.cfg -f ./tcl/target/stm8s105.cfg -c "init" -c "reset halt"
```

If you made changed, recompile using:

```
cd /c/Users/lapto/dev/c/openocd
make -j2
```




# Building openocd with MSYS2

Open the MSYS2 console


cd C:\aaa_se\sdk_toolchain\MSYS2Portable\App\msys32 double click mingw64.exe

```
cd /c/aaa_se/c/openocd-master
```

```
pacman -Syu
pacman -S --needed base-devel mingw-w64-ucrt-x86_64-toolchain

pacman -Rns $(pacman -Qqe)
paccache -r

pacman -S mingw-w64-x86_64-toolchain
pacman -S msys/libtool
pacman -S msys/autoconf
pacman -S msys/automake-wrapper
pacman -S mingw-w64-x86_64-autotools
//pacman -S gcc  // this is not the package that you want to install since this gcc does not find the libusb
pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-toolchain
pacman -S mingw-w64-x86_64-libusb
```

checkout commit SHA 420f637 - 23.Dez 2024
The other versions have automate m4 errors

```
git submodule update --init --recursive
```

```
./bootstrap
git submodule init
git submodule update
```

from the file configure, remove enable_werror on line 933

```
./configure
make -j2

mkdir build
cd build
```

// https://sourceforge.net/p/openocd/mailman/message/59200248/

```
../configure --enable-ftdi --enable-internal-jimtcl --disable-werror
```

configure: error: jimtcl is required but not found via pkg-config and system includes
https://sourceforge.net/p/openocd/mailman/message/59200248/

```
configure: error: in `/c/aaa_se/c/openocd-master/build':
configure: error: no acceptable C compiler found in $PATH
See `config.log' for more details
```




# STLink Protocol

I am trying to understand the protocol that openocd speaks to the STLink over USB.
Sadly I have not found any official documentation about that protocol yet.

## GetVersion

openocd uses libsub transfers instead of calling bulk-functions!
It prepares two transfers. The first transfer first contians the ASCII characters ("USBC", 0x55 0x53 0x42 0x43) followed by some other bytes and
finally followed by 16 bytes for the GET_VERSION command which is started using the code 0xF1 for GetVersion, followed by zeroes to fill up to 16 bytes.

TODO: Next steps
-- learn how to use transfers with libusb
-- learn why "USBC" is used before the GetVersion command and what each individual byte means




## Compiling

SWIM

https://www.st.com/resource/en/user_manual/cd00173911.pdf



https://github.com/openocd-org/openocd/blob/master/src/jtag/swim.c

With Zadig.exe list all devices and make sure it has discovered STM32 STLink.
Replace the driver to WinUSB since (https://stackoverflow.com/questions/74523773/why-can-i-open-stmicroelectronics-stlink-v2-with-libusb-but-not-with-stmicroele)
WinUSB is the driver that allows low level USB access to the STLink that libusb needs.

VID: 0483
PID: 3744



https://github.com/stlink-org/stlink


Get Version seems to be 0xF4 0xF1




Action / CommandHex SequencePurposeGet 
Version 0xF1 0x00 ...Retrieves ST-LINK hardware/firmware version
Enter SWD Mode 0xF2 0x20 0xA3 ...Shifts the probe state machine into SWD mode
Enter SWIM Mode 0xF2 0x10 ...Shifts state machine into STM8 SWIM mode
Read Memory 0xF3 ...Fetches data from a specific target RAM/Flash address
Write Memory 0xF4 ...Flashes or writes to target RAM/Flash address
Current Status 0xF5 ...Returns halted/running state of the target MCU




https://gitlab.cba.mit.edu/calischs/openocd_nrf52_patch/-/blob/f60d42b0e2d7c1bde4f75cf5fcaeddf11d21f7ed/src/jtag/drivers/stlink_usb.c



/** */
static inline int stlink_usb_xfer_noerrcheck(void *handle, const uint8_t *buf, int size)
{
	struct stlink_usb_handle *h = handle;
	return h->backend->xfer_noerrcheck(handle, buf, size);
}



In the ST-Link USB protocol, Endpoint 0x81 (IN) and Endpoint 0x02 (OUT) 
are the primary bulk endpoints used for transferring debugger commands and receiving target responses.





# openocd

/** */
static void stlink_usb_init_buffer(void *handle, uint8_t direction, uint32_t size)
{
	struct stlink_usb_handle *h = handle;

	h->direction = direction;

	h->cmdidx = 0;

	memset(h->cmdbuf, 0, STLINK_SG_SIZE);
	memset(h->databuf, 0, STLINK_DATA_SIZE);

	if (h->version.stlink == 1)
		stlink_usb_xfer_v1_create_cmd(handle, direction, size);
}





openocd -f interface/stlink.cfg -c "transport select swim" -f target/stm8s103.cfg -c "init" -c "reset halt"

Open On-Chip Debugger 0.10.0 (2020-03-10) [https://github.com/sysprogs/openocd]
Licensed under GNU GPL v2
libusb1 09e75e98b4d9ea7909e8837b7a3f00dda4589dc3
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Error: Debug adapter doesn't support 'swim' transport


THIS WORKS:

openocd -d3 -f interface/stlink.cfg -f target/stm8s105.cfg -c "init" -c "reset halt"




U5353@DEM1272 /cygdrive/c/aaa_se/c/libusb_stlink
$ openocd -d3 -f interface/stlink.cfg -f target/stm8s105.cfg -c "init" -c "reset halt"
Open On-Chip Debugger 0.10.0 (2020-03-10) [https://github.com/sysprogs/openocd]
Licensed under GNU GPL v2
libusb1 09e75e98b4d9ea7909e8837b7a3f00dda4589dc3
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
User : 14 1 options.c:63 configuration_output_handler(): debug_level: 3
User : 15 4 options.c:63 configuration_output_handler():
Debug: 16 5 options.c:187 add_default_dirs(): bindir=../../bin
Debug: 17 7 options.c:188 add_default_dirs(): pkgdatadir=../../share/openocd
Debug: 18 10 options.c:189 add_default_dirs(): exepath=C:/aaa_se/sdk_toolchain/openocd/bin
Debug: 19 13 options.c:190 add_default_dirs(): bin2data=../share/openocd
Debug: 20 15 configuration.c:42 add_script_search_dir(): adding C:\cygwin64\home\U5353/.openocd
Debug: 21 18 configuration.c:42 add_script_search_dir(): adding C:\Users\U5353\AppData\Roaming/OpenOCD
Debug: 22 21 configuration.c:42 add_script_search_dir(): adding C:/aaa_se/sdk_toolchain/openocd/bin/../share/openocd/site
Debug: 23 26 configuration.c:42 add_script_search_dir(): adding C:/aaa_se/sdk_toolchain/openocd/bin/../share/openocd/scripts
Debug: 24 32 configuration.c:97 find_file(): found C:/aaa_se/sdk_toolchain/openocd/bin/../share/openocd/scripts/interface/stlink.cfg
Debug: 25 37 command.c:143 script_debug(): command - adapter adapter driver hla
Debug: 27 40 command.c:354 register_command_handler(): registering 'hla_device_desc'...
Debug: 28 43 command.c:354 register_command_handler(): registering 'hla_serial'...
Debug: 29 45 command.c:354 register_command_handler(): registering 'hla_port'...
Debug: 30 48 command.c:354 register_command_handler(): registering 'hla_layout'...
Debug: 31 51 command.c:354 register_command_handler(): registering 'hla_vid_pid'...
Debug: 32 54 command.c:354 register_command_handler(): registering 'init_core'...
Debug: 33 56 command.c:354 register_command_handler(): registering 'close_core'...
Debug: 34 59 command.c:354 register_command_handler(): registering 'hla_command'...
Debug: 35 62 command.c:143 script_debug(): command - hla_layout hla_layout stlink
Debug: 37 64 hla_interface.c:290 hl_interface_handle_layout_command(): hl_interface_handle_layout_command
Debug: 38 69 command.c:143 script_debug(): command - hla_device_desc hla_device_desc ST-LINK
Debug: 40 73 hla_interface.c:216 hl_interface_handle_device_desc_command(): hl_interface_handle_device_desc_command
Debug: 41 77 command.c:143 script_debug(): command - hla_vid_pid hla_vid_pid 0x0483 0x3744 0x0483 0x3748 0x0483 0x374b 0x0483 0x374d 0x0483 0x374e 0x0483 0x374f 0x0483 0x3752 0x0483 0x3753
Debug: 43 84 configuration.c:97 find_file(): found C:/aaa_se/sdk_toolchain/openocd/bin/../share/openocd/scripts/target/stm8s105.cfg
Debug: 44 90 configuration.c:97 find_file(): found C:/aaa_se/sdk_toolchain/openocd/bin/../share/openocd/scripts/target/stm8s.cfg
Debug: 45 94 command.c:143 script_debug(): command - transport transport select stlink_swim
Debug: 46 98 hla_transport.c:191 hl_transport_select(): hl_transport_select
Debug: 47 101 command.c:354 register_command_handler(): registering 'hla'...
Debug: 48 104 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 49 106 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 50 109 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 51 112 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 52 115 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 53 117 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 54 120 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 55 122 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 56 124 command.c:354 register_command_handler(): registering 'jtag'...
Debug: 57 128 command.c:354 register_command_handler(): registering 'jtag_ntrst_delay'...
Debug: 58 131 command.c:143 script_debug(): command - hla hla newtap stm8s cpu -irlen 4 -ircapture 0x1 -irmask 0xf -expected-id 0
Debug: 59 135 hla_tcl.c:110 jim_hl_newtap_cmd(): Creating New Tap, Chip: stm8s, Tap: cpu, Dotted: stm8s.cpu, 8 params
Debug: 60 140 hla_tcl.c:121 jim_hl_newtap_cmd(): Processing option: -irlen
Debug: 61 142 hla_tcl.c:121 jim_hl_newtap_cmd(): Processing option: -ircapture
Debug: 62 144 hla_tcl.c:121 jim_hl_newtap_cmd(): Processing option: -irmask
Debug: 63 147 hla_tcl.c:121 jim_hl_newtap_cmd(): Processing option: -expected-id
Debug: 64 150 core.c:1484 jtag_tap_init(): Created Tap: stm8s.cpu @ abs position 0, irlen 0, capture: 0x0 mask: 0x0
Debug: 65 154 command.c:143 script_debug(): command - target target create stm8s.cpu stm8 -chain-position stm8s.cpu
Debug: 66 157 target.c:1973 target_free_all_working_areas_restore(): freeing all working areas
Debug: 67 161 stm8.c:467 stm8_configure_break_unit(): hw breakpoints: numinst 2 numdata 2
Debug: 68 163 command.c:354 register_command_handler(): registering 'stm8'...
Debug: 69 165 command.c:354 register_command_handler(): registering 'stm8'...
Debug: 70 168 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 71 171 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 72 173 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 73 176 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 74 178 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 75 180 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 76 183 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 77 185 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 78 188 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 79 190 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 80 193 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 81 195 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 82 198 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 83 200 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 84 203 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 85 205 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 86 207 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 87 210 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 88 213 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 89 215 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 90 218 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 91 221 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 92 224 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 93 226 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 94 230 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 95 232 command.c:354 register_command_handler(): registering 'stm8s.cpu'...
Debug: 96 235 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu configure -work-area-phys 0x0 -work-area-size 0x400 -work-area-backup 1
Debug: 97 240 target.c:1973 target_free_all_working_areas_restore(): freeing all working areas
Debug: 98 243 target.c:1973 target_free_all_working_areas_restore(): freeing all working areas
Debug: 99 247 target.c:1973 target_free_all_working_areas_restore(): freeing all working areas
Debug: 100 250 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu configure -flashstart 0x8000 -flashend 0xffff -eepromstart 0x4000 -eepromend 0x43ff
Debug: 101 257 stm8.c:2010 stm8_jim_configure(): flashstart=00008000
Debug: 102 260 stm8.c:2029 stm8_jim_configure(): flashend=0000ffff
Debug: 103 264 stm8.c:2048 stm8_jim_configure(): eepromstart=00004000
Debug: 104 266 stm8.c:2067 stm8_jim_configure(): eepromend=000043ff
Debug: 105 270 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu configure -optionstart 0x4800 -optionend 0x487f -blocksize 0x80
Debug: 106 276 stm8.c:2086 stm8_jim_configure(): optionstart=00004800
Debug: 107 278 stm8.c:2105 stm8_jim_configure(): optionend=0000487f
Debug: 108 280 stm8.c:1991 stm8_jim_configure(): blocksize=00000080
Debug: 109 282 command.c:143 script_debug(): command - adapter adapter speed 1
Debug: 111 286 core.c:1822 jtag_config_khz(): handle jtag khz
Debug: 112 288 core.c:1785 adapter_khz_to_speed(): convert khz to interface specific speed value
Debug: 113 291 core.c:1785 adapter_khz_to_speed(): convert khz to interface specific speed value
Debug: 114 295 command.c:143 script_debug(): command - reset_config reset_config srst_only
User : 116 298 options.c:63 configuration_output_handler(): srst_only separate srst_gates_jtag srst_open_drain connect_deassert_srst
User : 117 303 options.c:63 configuration_output_handler():
Debug: 118 305 command.c:143 script_debug(): command - init init
Debug: 120 307 command.c:143 script_debug(): command - target target init
Debug: 122 311 command.c:143 script_debug(): command - target target names
Debug: 123 314 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu cget -event gdb-flash-erase-start
Debug: 124 318 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu configure -event gdb-flash-erase-start reset init
Debug: 125 323 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu cget -event gdb-flash-write-end
Debug: 126 327 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu configure -event gdb-flash-write-end reset halt
Debug: 127 332 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu cget -event gdb-attach
Debug: 128 335 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu configure -event gdb-attach halt
Debug: 129 339 target.c:1435 handle_target_init_command(): Initializing targets...
Debug: 130 341 command.c:354 register_command_handler(): registering 'target_request'...
Debug: 131 344 command.c:354 register_command_handler(): registering 'trace'...
Debug: 132 348 command.c:354 register_command_handler(): registering 'trace'...
Debug: 133 351 command.c:354 register_command_handler(): registering 'fast_load_image'...
Debug: 134 354 command.c:354 register_command_handler(): registering 'fast_load'...
Debug: 135 357 command.c:354 register_command_handler(): registering 'profile'...
Debug: 136 359 command.c:354 register_command_handler(): registering 'virt2phys'...
Debug: 137 362 command.c:354 register_command_handler(): registering 'reg'...
Debug: 138 365 command.c:354 register_command_handler(): registering 'poll'...
Debug: 139 368 command.c:354 register_command_handler(): registering 'wait_halt'...
Debug: 140 371 command.c:354 register_command_handler(): registering 'halt'...
Debug: 141 374 command.c:354 register_command_handler(): registering 'resume'...
Debug: 142 378 command.c:354 register_command_handler(): registering 'reset'...
Debug: 143 381 command.c:354 register_command_handler(): registering 'soft_reset_halt'...
Debug: 144 384 command.c:354 register_command_handler(): registering 'step'...
Debug: 145 387 command.c:354 register_command_handler(): registering 'mdd'...
Debug: 146 389 command.c:354 register_command_handler(): registering 'mdw'...
Debug: 147 391 command.c:354 register_command_handler(): registering 'mdh'...
Debug: 148 395 command.c:354 register_command_handler(): registering 'mdb'...
Debug: 149 398 command.c:354 register_command_handler(): registering 'mwd'...
Debug: 150 400 command.c:354 register_command_handler(): registering 'mww'...
Debug: 151 403 command.c:354 register_command_handler(): registering 'mwh'...
Debug: 152 406 command.c:354 register_command_handler(): registering 'mwb'...
Debug: 153 409 command.c:354 register_command_handler(): registering 'bp'...
Debug: 154 412 command.c:354 register_command_handler(): registering 'rbp'...
Debug: 155 415 command.c:354 register_command_handler(): registering 'wp'...
Debug: 156 418 command.c:354 register_command_handler(): registering 'rwp'...
Debug: 157 421 command.c:354 register_command_handler(): registering 'load_image'...
Debug: 158 423 command.c:354 register_command_handler(): registering 'dump_image'...
Debug: 159 428 command.c:354 register_command_handler(): registering 'verify_image_checksum'...
Debug: 160 431 command.c:354 register_command_handler(): registering 'verify_image'...
Debug: 161 434 command.c:354 register_command_handler(): registering 'test_image'...
Debug: 162 437 command.c:354 register_command_handler(): registering 'reset_nag'...
Debug: 163 440 command.c:354 register_command_handler(): registering 'ps'...
Debug: 164 443 command.c:354 register_command_handler(): registering 'test_mem_access'...
Debug: 165 447 command.c:354 register_command_handler(): registering 'report_flash_progress'...
Debug: 166 450 command.c:354 register_command_handler(): registering 'run_until_stop_fast'...
Debug: 167 453 command.c:354 register_command_handler(): registering 'wait_for_stop'...
Debug: 168 457 hla_interface.c:109 hl_interface_init(): hl_interface_init
Debug: 169 459 hla_layout.c:89 hl_layout_init(): hl_layout_init
Debug: 170 461 core.c:1785 adapter_khz_to_speed(): convert khz to interface specific speed value
Debug: 171 464 core.c:1789 adapter_khz_to_speed(): have interface set up
Debug: 172 467 core.c:1785 adapter_khz_to_speed(): convert khz to interface specific speed value
Debug: 173 472 core.c:1789 adapter_khz_to_speed(): have interface set up
Info : 174 474 core.c:1565 adapter_init(): clock speed 1 kHz
Debug: 175 476 openocd.c:141 handle_init_command(): Debug Adapter init complete
Debug: 176 478 command.c:143 script_debug(): command - transport transport init
Debug: 178 481 transport.c:239 handle_transport_init(): handle_transport_init
Debug: 179 483 hla_transport.c:152 hl_transport_init(): hl_transport_init
Debug: 180 487 hla_transport.c:169 hl_transport_init(): current transport stlink_swim
Debug: 181 490 hla_interface.c:42 hl_interface_open(): hl_interface_open
Debug: 182 493 hla_layout.c:40 hl_layout_open(): hl_layout_open
Debug: 183 494 stlink_usb.c:2565 stlink_usb_open(): stlink_usb_open
Debug: 184 497 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x3744 serial:
Debug: 185 501 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x3748 serial:
Debug: 186 504 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x374b serial:
Debug: 187 507 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x374d serial:
Debug: 188 511 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x374e serial:
Debug: 189 514 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x374f serial:
Debug: 190 518 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x3752 serial:
Debug: 191 522 stlink_usb.c:2580 stlink_usb_open(): transport: 3 vid: 0x0483 pid: 0x3753 serial:
Debug: 192 538 libusb_helper.c:151 jtag_libusb_open(): libusb reported 9 devices
Debug: 193 541 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #0 (04f2/b71f)
Debug: 194 544 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #1 (27c6/5503)
Debug: 195 549 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #2 (04f2/b71f)
Debug: 196 553 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #4 (0b05/195c)
Debug: 197 556 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #5 (8086/9a17)
Debug: 198 560 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #6 (8086/43ed)
Debug: 199 564 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #7 (0b05/195c)
Debug: 200 568 libusb_helper.c:167 jtag_libusb_open(): USB descriptor mismatch for device #8 (8087/0026)
Info : 201 572 stlink_usb.c:892 stlink_usb_version(): STLINK v1 JTAG v0 API v1 SWIM v3 VID 0x0483 PID 0x3744
Info : 202 576 stlink_usb.c:2703 stlink_usb_open(): using stlink api v1
Debug: 203 582 stlink_usb.c:1127 stlink_exit_mode(): MODE: 0x00
Debug: 204 584 stlink_usb.c:1148 stlink_exit_mode(): E-MODE: 0x01
Debug: 205 588 stlink_usb.c:1177 stlink_usb_init_mode(): MODE: 0x01
Debug: 206 591 stlink_usb.c:1238 stlink_usb_init_mode(): MODE: 0x03
Debug: 207 611 core.c:640 adapter_system_reset(): SRST line released
Debug: 208 614 hla_interface.c:67 hl_interface_init_target(): hl_interface_init_target
Debug: 209 616 command.c:143 script_debug(): command - dap dap init
Debug: 212 620 arm_dap.c:106 dap_init_all(): Initializing all DAPs ...
Debug: 213 623 openocd.c:158 handle_init_command(): Examining targets...
Debug: 214 627 target.c:1621 target_call_event_callbacks(): target event 17 (examine-start) for core stm8s.cpu
Debug: 215 631 stm8.c:1704 stm8_examine(): writing A0 to SWIM_CSR (SAFE_MASK + SWIM_DM)
Debug: 216 658 stm8.c:1709 stm8_examine(): writing B0 to SWIM_CSR (SAFE_MASK + SWIM_DM + HS)
Debug: 217 663 target.c:1621 target_call_event_callbacks(): target event 18 (examine-end) for core stm8s.cpu
Debug: 218 668 command.c:143 script_debug(): command - flash flash init
Debug: 220 673 tcl.c:1240 handle_flash_init_command(): Initializing flash devices...
Debug: 221 676 command.c:143 script_debug(): command - nand nand init
Debug: 223 680 tcl.c:498 handle_nand_init_command(): Initializing NAND devices...
Debug: 224 683 command.c:143 script_debug(): command - pld pld init
Debug: 226 687 pld.c:206 handle_pld_init_command(): Initializing PLDs...
Debug: 227 689 gdb_server.c:3512 gdb_target_start(): starting gdb server for stm8s.cpu on 3333
Info : 228 694 server.c:310 add_service(): Listening on port 3333 for gdb connections
Debug: 229 697 command.c:143 script_debug(): command - reset reset halt
Debug: 231 700 target.c:1640 target_call_reset_callbacks(): target reset 2 (halt)
Debug: 232 703 command.c:143 script_debug(): command - target target names
Debug: 233 706 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event reset-start
Debug: 234 710 command.c:143 script_debug(): command - transport transport select
Debug: 235 713 command.c:143 script_debug(): command - transport transport select
Debug: 236 715 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event examine-start
Debug: 237 718 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu arp_examine allow-defer
Debug: 238 722 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event examine-end
Debug: 239 724 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event reset-assert-pre
Debug: 240 728 command.c:143 script_debug(): command - transport transport select
Debug: 241 731 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu arp_reset assert 1
Debug: 242 734 target.c:1973 target_free_all_working_areas_restore(): freeing all working areas
Debug: 243 739 core.c:636 adapter_system_reset(): SRST line asserted
Debug: 244 741 stm8.c:896 stm8_halt(): target->state: reset
Debug: 245 743 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event reset-assert-post
Debug: 246 746 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event reset-deassert-pre
Debug: 247 750 command.c:143 script_debug(): command - transport transport select
Debug: 248 753 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu arp_reset deassert 1
Debug: 249 756 target.c:1973 target_free_all_working_areas_restore(): freeing all working areas
Debug: 250 761 core.c:640 adapter_system_reset(): SRST line released
Debug: 251 763 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event reset-deassert-post
Debug: 252 766 command.c:143 script_debug(): command - transport transport select
Debug: 253 769 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu was_examined
Debug: 254 772 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu arp_waitstate halted 1000
Debug: 255 784 stm8.c:1157 stm8_read_core_reg(): read core reg 0 value 0x8000
Debug: 256 786 stm8.c:1157 stm8_read_core_reg(): read core reg 1 value 0x0
Debug: 257 790 stm8.c:1157 stm8_read_core_reg(): read core reg 2 value 0x0
Debug: 258 792 stm8.c:1157 stm8_read_core_reg(): read core reg 3 value 0x0
Debug: 259 794 stm8.c:1157 stm8_read_core_reg(): read core reg 4 value 0x3ff
Debug: 260 797 stm8.c:1157 stm8_read_core_reg(): read core reg 5 value 0x28
Debug: 261 804 stm8.c:482 stm8_examine_debug_reason(): csr1 = 0x10 csr2 = 0x08
Debug: 262 806 stm8.c:522 stm8_debug_entry(): entered debug state at PC 0x8000, target->state: reset
Debug: 263 810 target.c:1621 target_call_event_callbacks(): target event 0 (gdb-halt) for core stm8s.cpu
Debug: 264 815 target.c:1621 target_call_event_callbacks(): target event 1 (halted) for core stm8s.cpu
User : 265 817 stm8.c:1312 stm8_arch_state(): target halted due to debug-request, pc: 0x00008000
Debug: 266 821 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu curstate
Debug: 267 824 command.c:143 script_debug(): command - stm8s.cpu stm8s.cpu invoke-event reset-end
Info : 268 829 server.c:310 add_service(): Listening on port 6666 for tcl connections
Info : 269 832 server.c:310 add_service(): Listening on port 4444 for telnet connections
Debug: 270 836 command.c:143 script_debug(): command - init init






# Building openocd in cygwin on windows 11


https://privateisland.tech/dev/openocd-darsena-windows

To install new software, use downloads/setup-x86_64.exe

Cygwin Devel packages to install:

cygwin-devel

autobuild
autoconf
autoconf-archive
automake
dos2unix
git
gcc-core
libtool
libusb1.0
libusb1.0-devel
make
pkg-config
usbutils



open cygwin shell

```
cd /cygdrive/c/aaa_se/c/openocd-master
mkdir build
./bootstrap
```

bootstrap causes errors that I cannot solve. Switching to the MSys2 console

-I<search path to include files>
-L<search path to the lib file>
-l<libname>

```
gcc -o my_program.exe main.c -I/usr/include/libusb-1.0 -L/usr/lib -llibusb-1.0

./my_program.exe
```









