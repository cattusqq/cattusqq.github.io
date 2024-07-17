---
title: "SWD using Glasgow Interface Explorer PART 1"
date: 2024-06-17 11:05:00
categories: [Hardware, SWD]
tags: [hardware, swd, glasgow]     # TAG names should always be lowercase
published: true
---
# Summary
A guide for using the Glasgow Interface Explorer to program and debug a Raspberry Pi Pico using SWD (Serial Wire Debug) PART 1/2.

# Context
I've wanted for many years to learn how to use SWD (Serial Wire Debug) for hardware hacking ARM devices. Given it's built into ARM processors, I figured it could be a way to gain control of embedded devices that don't expose JTAG or where kernels do not have UART enabled or exposed. There are of course other benefits over hardware debugging than just serial console access - the ability to directly control the CPU, means controlling register values and thus control flow among other abilities. 

But SWD has always stumped me. Theoretically the technology seems simple enough, one wire for clock and a single bidirectional wire for IO. And while I've done my fair share of JTAG and UART projects, actually attempting to use SWD has always left me frustrated and confused. 

It seems like most multiprotocol devices these days, like the JTAGULATOR and the TIGARD, all support SWD. And there are a multitude of specific SWD dongles for specific target boards. I found it all a bit overwhelming. 

A few months back I bought the [Glasgow Interface Explorer](https://glasgow-embedded.org/latest/intro.html) through Crowd Supply, and decided to give SWD another try.

I have some interesting embedded devices in mind I'd love to use it on in the future, but I decided to go with something very simple and easy to troubleshoot given my history of mental blocks with SWD. So the target device is a simple Raspberry Pi Pico. 

Here's my journey recount.

Note that I am using MacOS 13.3 as my primary environment, so unless otherwise stated below, assume all actions are taken on MacOS. 

# Lets Do This!
## Step 1 - Set up the Glasgow
I followed the instructions on the [Official Glasgow Documentation](https://glasgow-embedded.org/latest/install.html) to clone the git repo down and install it with pipx.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> Don't forget to check your version, and choose an applet to deploy to the FPGA. You need to deploy the right applet for the task you're going to do! Note that I'm starting with the UART applet, for reasons explained below.
{: .prompt-warning }
<!-- markdownlint-restore -->

```bash
$ glasgow --version
$ glasgow build --rev C3 uart
```
At this stage I'm not ready to develop for it, I just want to use it, so after that I skipped to [Using the Device](https://glasgow-embedded.org/latest/use/basic.html).

## Step 2 - Using the Glasgow

Since I am familiar with UART, I read about how to use the Glasgow for UART. Since I'd never used SWD before, and thus didn't know how to tell if I was on the right track, or notice red flags to tell me I'm doing something very wrong, I figured I'd start with something easily verifiable first. 

```bash
$ glasgow run uart -V 3.3 --pin-tx 0 --pin-rx 1 tty
```

Seems simple enough. `glasgow run (protocol) (protocol specific options)`

Ok, let's try SWD.

I couldn't see any documentation on the site specifically for SWD, but the help built into the Glasgow cli was pretty decent and I was able to figure out what to do. 

```bash
$ glasgow --help
$ glasgow run --help
$ glasgow run uart --help
```

```bash
$ glasgow run --help
usage: glasgow run [-h] [--override-required-revision] [--reload | --prebuilt | --prebuilt-at BITSTREAM-FILE] [--trace VCD-FILE] APPLET ...

positional arguments:
  APPLET
    analyzer                capture logic waveforms (revC0+)
    audio-dac               play sound using a ΣΔ-DAC
    audio-yamaha-opx        /!\ unavailable due to unmet requirements: aiohttp-remotes<2,>=1.2, aiohttp>=3.9.0
    benchmark               evaluate communication performance
    control-servo           control RC servomotors and ESCs (revC0+)
    control-tps6598x        configure TPS6598x USB PD controllers (revC0+)
    debug-arc               debug ARC processors via JTAG
    debug-arm               debug ARM processors via JTAG (PREVIEW QUALITY APPLET)
    debug-mips              debug MIPS processors via EJTAG (PREVIEW QUALITY APPLET)
    display-hd44780         display characters on HD44780-compatible LCDs (PREVIEW QUALITY APPLET) (revC0+)
    display-pdi             display images on Pervasive Display Inc EPD panels
    i2c-initiator           initiate I²C transactions (revC0+)
    i2c-target              accept I²C transactions (revC0+)
    jtag-openocd            expose JTAG via OpenOCD remote bitbang interface
    jtag-pinout             automatically determine JTAG pinout
    jtag-probe              test integrated circuits via IEEE 1149.1 JTAG
    jtag-svf                play SVF test vectors via JTAG
    memory-24x              read and write 24-series I²C EEPROM memories (revC0+)
    memory-25x              read and write 25-series SPI Flash memories
    memory-floppy           read and write disks using IBM/Shugart floppy drives (PREVIEW QUALITY APPLET) (revC0+)
    memory-onfi             read and write ONFI-like NAND Flash memories (PREVIEW QUALITY APPLET)
    memory-prom             read and rescue parallel EPROMs, EEPROMs, and Flash memories
    program-avr-spi         program Microchip (Atmel) AVR microcontrollers via SPI
    program-ice40-flash     program 25-series Flash memories used with iCE40 FPGAs
    program-ice40-sram      program SRAM of iCE40 FPGAs
    program-m16c            program Renesas M16C microcomputers via UART
    program-mec16xx         program Microchip MEC16xx embedded controller via JTAG
    program-nrf24lx1        program nRF24LE1 and nRF24LU1+ RF microcontrollers
    program-stusb4500-nvm   read and write STUSB4500 NVM (revC0+)
    program-xc6s            program Xilinx Spartan-6 FPGAs via JTAG (PREVIEW QUALITY APPLET)
    program-xc9500          program Xilinx XC9500 CPLDs via JTAG
    program-xc9500xl        program Xilinx XC9500XL and XC9500XV CPLDs via JTAG
    program-xpla3           program Xilinx XPLA3 CPLDs via JTAG
    ps2-host                communicate with IBM PS/2 peripherals (revC0+)
    radio-nrf24l01          transmit and receive using nRF24L01(+) RF PHY
    sbw-probe               probe microcontrollers via TI Spy-Bi-Wire (revC0+)
    selftest                diagnose hardware faults
    sensor-bmx280           measure temperature, pressure, and humidity with Bosch BMx280 sensors (revC0+)
    sensor-hx711            measure voltage with AVIA Semiconductor HX711
    sensor-ina260           measure voltage, current and power with TI INA260 sensors (revC0+)
    sensor-mouse-ps2        receive axis and button information from PS/2 mice (revC0+)
    sensor-pmsx003          measure air quality with Plantower PMx003 sensors
    sensor-scd30            measure CO₂, humidity, and temperature with Sensirion SCD30 sensors (revC0+)
    sensor-sen5x            measure PM, NOx, VOC, humidity, and temperature with Sensirion SEN5x sensors (revC0+)
    spi-controller          initiate SPI transactions
    spi-flashrom            expose SPI via flashrom serprog interface
    swd-openocd             expose SWD via OpenOCD remote bitbang interface
    uart                    communicate via UART
    video-hub75-output      display a test pattern on HUB75 panel
    video-rgb-input         capture video stream from RGB555 LCD bus (PREVIEW QUALITY APPLET)
    video-vga-output        display video via VGA
    video-ws2812-output     display video via WS2812 LEDs

options:
  -h, --help                show this help message and exit
  --override-required-revision
                            (advanced) override applet revision requirement
  --reload                  (advanced) reload bitstream even if an identical one is already loaded
  --prebuilt                (advanced) load prebuilt applet bitstream from ./<applet-name.bin>
  --prebuilt-at BITSTREAM-FILE
                            (advanced) load prebuilt applet bitstream from BITSTREAM-FILE
  --trace VCD-FILE          trace applet I/O to VCD-FILE
  $

```

The one that seemed most like what I wanted was `swd-openocd             expose SWD via OpenOCD remote bitbang interface`, even though I had no idea what 'remote bitbang' or OpenOCD were. 

```bash
$ glasgow run swd-openocd --help
usage: glasgow run swd-openocd [-h] [--port SPEC] [--pin-swclk NUM] [--pin-swdio NUM] [--pin-srst NUM] [-f FREQ] (-V [VOLTS] | -M | --keep-voltage) ENDPOINT

Expose SWD via a socket using the OpenOCD remote bitbang protocol.

Usage with TCP sockets:

    glasgow run swd-openocd tcp:localhost:2222
    openocd -c 'adapter driver remote_bitbang; transport select swd' \
        -c 'remote_bitbang port 2222'

Usage with Unix domain sockets:

    glasgow run swd-openocd unix:/tmp/swd.sock
    openocd -c 'adapter driver remote_bitbang; transport select swd' \
        -c 'remote_bitbang host /tmp/swd.sock'

positional arguments:
  ENDPOINT                  listen at ENDPOINT, either unix:PATH or tcp:HOST:PORT

options:
  -h, --help                show this help message and exit

build arguments:
  --port SPEC               bind the applet to port SPEC (default: AB)
  --pin-swclk NUM           bind the applet I/O line 'swclk' to pin NUM (default: 0)
  --pin-swdio NUM           bind the applet I/O line 'swdio' to pin NUM (default: 1)
  --pin-srst NUM            bind the applet I/O line 'srst' to pin NUM
  -f FREQ, --frequency FREQ
                            set SWCLK frequency to FREQ kHz (default: 100)

run arguments:
  -V [VOLTS], --voltage [VOLTS]
                            set I/O port voltage explicitly
  -M, --mirror-voltage      sense and mirror I/O port voltage
  --keep-voltage            do not change I/O port voltage
  $
```

It's pretty clear from this example that I have to use OpenOCD to connect over a socket to the Glasgow. I still don't know what a remote bitbang is, but we're making progress. 

Unfortunately, just using the commands in the help text verbatim did not work. Neither did any of the various permutations I made. I kept getting errors like `Debug adapter doesn't support any transports?`, exits, and even a segfault on the OpenOCD side. The Glasgow would detect that OpenOCD was connecting to it, but the OpenOCD process would have something go wrong, exit, and leave the Glasgow hanging.

Something was wrong. 

## Step 3 - Troubleshoot by building and deploying a simple pico binary

I wasn't sure if it wasn't working because I was doing something wrong, or because there was no firmware on the pico. So to rule out at least one of those possibilities, I decided to just try programming a simple rpi pico binary onto the pico. It also seemed simpler than attempting a full debug, which I knew I'd also need to plumb gdb into somehow.

The rpi pico has a git repo of examples you can just build.

### Step 3.1 - Install the RPi Pico SDK

This pdf, [Getting started with Pico](https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf), was immensely helpful. It talked about building for the pico[^1], and using OpenOCD to program it with your built binaries[^2]. 

In hindsight, I skipped `Chapter 1. Quick Pico Setup` which probably would have saved me a lot of time. But oh well, live and learn.

[^1]: See Chapter 2. The SDK
[^2]: See Chapter 5. Flash Programming with SWD

When I got up to `Chapter 2.2. Install the toolchain` I realised I was going to have issues on MacOS. Its probably doable, but I knew it would be way easier to do the actual building on Linux, and the instructions assume you're using a Debian based system. 

Luckily, I have access to a Kubernetes cluster, so I was able to make a simple Debian docker container with everything I needed. 

Since I don't think Docker and Devpod are what most people are here for, I'll document that in a separate blog post.

### Step 3.2 - Compile an RPi Pico example binary

I chose the `Blink` example, since I could verify code was running by seeing if it blinked, and I could change the blink delays then use swd to flash the new binary on, and visually verify the blink delays had changed to know if my swd programming had worked. 

It compiled fine on Linux, and I followed `3.2. Load and run "Blink"` to copy the binary onto the pico, connected directly to my Macbook as a USB mass storage device. I.e, the easy way. This was just to confirm I'd compiled the right blink example for my pico, as there are two - one for the standard pico and one for the pico w.

The pico rebooted and started flashing. Huzzah!
Ok, now for the hard part :grimacing:

### Step 3.3 - Deploy the pico binary using SWD
I updated the blink delay and recompiled the binary so I'd know if it all worked. Then I followed the instructions `Chapter 5. Flash Programming with SWD` for this part. 

I connected the GND, SWCLK and SWDIO pins on the pico to the GND, pin0 and pin1 on the A side of the Glasgow. I did have to solder header pins onto the pico first. 

![Diagram for connecting Glasgow to RPi Pico](/images/connect_glasgow_to_rpi_pico.png)

We've done the physical part, now we have to figure out the right incantations to get the logical part working. 

Referring back to [Step 2 - Using the Glasgow](#step-2---using-the-glasgow), we know we need to run two processes - `glasgow` and `openocd`. Let's start with our previous commands and see if we can get them working now we have a binary to test with. 


```bash
$ glasgow run swd-openocd tcp:localhost:2222
usage: glasgow run swd-openocd [-h] [--port SPEC] [--pin-swclk NUM]
                               [--pin-swdio NUM] [--pin-srst NUM] [-f FREQ]
                               (-V [VOLTS] | -M | --keep-voltage)
                               ENDPOINT
glasgow run swd-openocd: error: one of the arguments -V/--voltage -M/--mirror-voltage --keep-voltage is required
$ glasgow run swd-openocd -V 3.3 tcp:localhost:2222
I: g.device.hardware: generating bitstream ID 2fb453a147052ae5a9ce577d0dbc4cb0
I: g.cli: running handler for applet 'swd-openocd'
I: g.applet.interface.swd_openocd: port(s) A, B voltage set to 3.3 V
I: g.applet.interface.swd_openocd: socket: listening at tcp:localhost:2222


```

```bash
$ openocd -f target/rp2040.cfg -c 'adapter driver remote_bitbang' -c 'remote_bitbang port 2222' -c "program ../../pico/blink.elf verify reset exit"
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Debug adapter doesn't support any transports?
```


### Step 3.4 - More Troubleshooting

Ok we're getting the same errors. Time to investigate. 

I'm pretty sure we need to set a transport, but OpenOCD isn't letting me. Are we using the right adapter?

With some googling I found the OpenOCD documentation for [Debug Adapter Configuration](https://openocd.org/doc/html/Debug-Adapter-Configuration.html). 

The very first paragraph on this page hints as to what our problem is:

```
Correctly installing OpenOCD includes making your operating system give OpenOCD access to debug adapters. Once that has been done, Tcl commands are used to select which one is used, and to configure how it is used.

Note: Because OpenOCD started out with a focus purely on JTAG, you may find places where it wrongly presumes JTAG is the only transport protocol in use. Be aware that recent versions of OpenOCD are removing that limitation. JTAG remains more functional than most other transports. Other transports do not support boundary scan operations, or may be specific to a given chip vendor. Some might be usable only for programming flash memory, instead of also for debugging.

Debug Adapters/Interfaces/Dongles are normally configured through commands in an interface configuration file which is sourced by your openocd.cfg file, or through a command line -f interface/....cfg option.

source [find interface/olimex-jtag-tiny.cfg]
These commands tell OpenOCD what type of JTAG adapter you have, and how to talk to it. A few cases are so simple that you only need to say what driver to use:

# jlink interface
adapter driver jlink
Most adapters need a bit more configuration than that.
```

So it sounds like we need to specify not only that we are using SWD (which is our transport) but we also need to specify the hardware that's providing that transport (Our Glasgow is the hardware we're using). 

Let's find the Glasgow interface file and add that to our OpenOCD command. Our command should notionally look something like `$ openocd -f target/rp2040.cfg  -f interface/GLASGOW.cfg -c 'adapter driver remote_bitbang' -c 'remote_bitbang port 2222' -c "program ../../pico/blink.elf verify reset exit"`

Note we are also specifying our target, or the thing that will be debugged, as rp2040.cf which is the config for the RPi Pico. This is explained in the Pico documentation referenced above. 

Looking at the `tcl/interface/` directory there doesnt appear to be a Glasgow config. 

If we open up one of the other configs, we can get a feel for what it should look like. I'm guessing the jlink would be a good analogue since it's referenced with SWD in the documentation. 

```bash
$ cat tcl/interface/jlink.cfg
# SPDX-License-Identifier: GPL-2.0-or-later

#
# SEGGER J-Link
#
# http://www.segger.com/jlink.html
#

adapter driver jlink

# The serial number can be used to select a specific device in case more than
# one is connected to the host.
#
# Example: Select J-Link with serial number 123456789
#
# adapter serial 123456789
```

I guess these target and interface files are just groupings of commands that you could otherwise pass individually to the OpenOCD process via -c. I'm not seeing any magic here that explains why our transport isn't supported. For reference, our 'driver' value here would be `remote-bitbang` instead of `jlink` according to the documentation in `openocd/doc/manual/jtag/drivers/remote_bitbang.txt`

So what else could be wrong?


Heres another hint. If we go back to the OpenOCD documentation and look at section [8.2 Interface Drivers](https://openocd.org/doc/html/Debug-Adapter-Configuration.html) it says:


`
8.2 Interface Drivers - 
Each of the interface drivers listed here must be explicitly enabled when OpenOCD is configured, in order to be made available at run time.
`

Do we need to configure OpenOCD with Glasgow support ourselves if it's not already configured?


Lets look at the Glasgow source code to confirm if there is Glasgow support or not. 
Searching the entire repo for `glasgow` gets us two hits:
* commit: [remote_bitbang: Add SWD support ](https://github.com/openocd-org/openocd/commit/0f70c6c325785517f35bbbb9316801bef7a79d8b)
* file: [openocd/doc/manual/jtag/drivers/remote_bitbang.txt](https://github.com/openocd-org/openocd/blob/23c33e1d3a332a94ef080451d43c6dc004e34750/doc/manual/jtag/drivers/remote_bitbang.txt#L14)



The commit to add SWD support to remote_bitbang sounds like exactly what we're looking for. It adds support for our transport (SWD) to our adapter driver (remote_bitbang). Maybe this code just isn't in our installed version of OpenOCD?

I Have version `open-ocd 0.12.0_1`. Let's go to the [openocd github repo and see when this release was cut](https://github.com/openocd-org/openocd/releases), then compare it to the date the above commit was made. 

[v0.12.0](https://github.com/openocd-org/openocd/releases/tag/v0.12.0) is listed as the latest release at time of writing. 

v0.12.0 released: Jan 15, 2023
Commit adding support for SWD to remote bitbang driver: committed on Dec 3, 2023 

Aha! Thats it! Our OpenOCD install is just missing some stuff that hasn't been released yet.

If we look at the [commit](https://github.com/openocd-org/openocd/commit/0f70c6c325785517f35bbbb9316801bef7a79d8b), we can see it has been merged into master, so should be stable. 
So if we reinstall OpenOCD from the latest commit to master branch, it should *fingers crossed* **just work**.

Famous last words. 


### Step 3.5 - Reinstall OpenOCD from latest commit

```bash
$ brew install --HEAD openocd
==> Downloading https://formulae.brew.sh/api/formula.jws.json
######################################################################### 100.0%
==> Downloading https://formulae.brew.sh/api/cask.jws.json
######################################################################### 100.0%
Error: open-ocd 0.12.0_1 is already installed
To install HEAD_1, first run:
  brew unlink open-ocd

$ brew unlink open-ocd
Unlinking /opt/homebrew/Cellar/open-ocd/0.12.0_1... 6 symlinks removed.

$ brew install --HEAD open-ocd
# For me, this installed from commit HEAD-bf4be56_1
# Long output excluded for brevity

$
# Done!

```

## Step 4 - Finally actually do the thing for real this time 

In separate terminal windows, run the Glasgow and OpenOCD commands. The Glasgow command has to be executed first and listening **before** the OpenOCD command is run, since it connects over UNIX socket to the Glasgow process. 

Let's review what the Glasgow command should look like by checking the help text for the swd-openocd Glasgow applet.

```bash
$ glasgow run swd-openocd --help
usage: glasgow run swd-openocd [-h] [--port SPEC] [--pin-swclk NUM] [--pin-swdio NUM] [--pin-srst NUM] [-f FREQ] (-V [VOLTS] | -M | --keep-voltage) ENDPOINT
```

Our wiring diagram in [Step 3.3](#step-33---deploy-the-pico-binary-using-swd) says to connect GND, SWCLK and SWDIO pins on the pico to the GND, pin0 and pin1 on the A side of the Glasgow. So lets update the command to reflect that. I'm omitting options that aren't required. Included below is a photo of my setup. 

![Diagram for connecting Glasgow to RPi Pico](/images/photo_glasgow_rpi_pico.png)

```
glasgow run swd-openocd -V 3.3 --pin-swdio 0 --pin-swclk 1 tcp:localhost:2222
I: g.device.hardware: generating bitstream ID 495f556d820677effe84d423ece8f5a0
I: g.cli: running handler for applet 'swd-openocd'
I: g.applet.interface.swd_openocd: port(s) A, B voltage set to 3.3 V
I: g.applet.interface.swd_openocd: socket: listening at tcp:localhost:2222
```

```bash
$ cd openocd/tcl 
# we have to be in this dir to reference the target config file

$ openocd -c 'adapter driver remote_bitbang' -c 'transport select swd' -c 'remote_bitbang port 2222' -f target/rp2040.cfg -c "program ../../pico/blink.elf verify reset exit"
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
swd
Warn : Transport "swd" was already selected
Info : Hardware thread awareness created
Info : Hardware thread awareness created
Info : Initializing remote_bitbang driver
Info : Connecting to localhost:2222
Info : remote_bitbang driver initialized
Info : Note: The adapter "remote_bitbang" doesn't support configurable speed
Info : SWD DPIDR 0x0bc12477, DLPIDR 0x00000001
Info : SWD DPIDR 0x0bc12477, DLPIDR 0x10000001
Info : [rp2040.core0] Cortex-M0+ r0p1 processor detected
Info : [rp2040.core0] target has 4 breakpoints, 2 watchpoints
Info : [rp2040.core0] Examination succeed
Info : [rp2040.core1] Cortex-M0+ r0p1 processor detected
Info : [rp2040.core1] target has 4 breakpoints, 2 watchpoints
Info : [rp2040.core1] Examination succeed
Info : starting gdb server for rp2040.core0 on 3333
Info : Listening on port 3333 for gdb connections
[rp2040.core0] halted due to breakpoint, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ee msp: 0x20041f00
[rp2040.core1] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ee msp: 0x20041f00
** Programming Started **
Info : Found flash device 'win w25q16jv' (ID 0x001540ef)
Info : RP2040 B0 Flash Probe: 2097152 bytes @0x10000000, in 32 sectors

Info : Padding image section 1 at 0x10005498 with 104 bytes (bank write end alignment)
Warn : Adding extra erase range, 0x10005500 .. 0x1000ffff
** Programming Finished **
** Verify Started **
** Verified OK **
** Resetting Target **
shutdown command invoked
Info : remote_bitbang interface quit
$

```

It worked! The pico rebooted and is now flashing the pattern defined in blink500.elf. Huzzah!

If we wanted to make a Glasgow SWD config for OpenOCD, it could look something like this:

```tcl
# filename: glasgow_swd.cfg
adapter driver remote_bitbang
transport select swd
remote_bitbang port 2222
```

and our updated OpenOCD command would look like this:

```bash
$ openocd -f interface/glasgow.cfg -f target/rp2040.cfg -c "program ../../pico/blink.elf verify reset exit"
```



If you get the following error when you run OpenOCD, try checking your cables are seated correctly. Its telling you there's an issue connecting to the pico. The Glasgow cable group sits very loosely and pops out on me a lot resulting in this error. Just make sure everything is plugged in securely and the error should resolve. 

```bash
 $ openocd -f interface/glasgow.cfg -f target/rp2040.cfg -c "program ../../pico/blink.elf verify reset exit"
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Warn : Transport "swd" was already selected
Info : Hardware thread awareness created
Info : Hardware thread awareness created
Info : Initializing remote_bitbang driver
Info : Connecting to localhost:2222
Info : remote_bitbang driver initialized
Info : Note: The adapter "remote_bitbang" doesn't support configurable speed
Error: Failed to connect multidrop rp2040.dap0
in procedure 'program'
** OpenOCD init failed **
shutdown command invoked

Info : remote_bitbang interface quit
```

## In Conclusion




That's it for this post! In the next one, we'll take what we've learned one step further, and use the Glasgow Interface Explorer to directly debug the Raspberry Pi Pico CPU with gdb/lldb. See you then! 
<hr>
### Updates
July 10 2024
: I filed an [issue](https://github.com/GlasgowEmbedded/glasgow/issues/616) against the [Glasgow github repo](https://github.com/GlasgowEmbedded/glasgow) to update their documentation to let folks know how to work around the problem of the current OpenOCD release not having SWD support yet. Hopefully this will be helpful to some folks.
July 11 2024
: The Glasgow folks updated the help text per my suggestion and this has been merged into main branch. See [commit](https://github.com/GlasgowEmbedded/glasgow/commit/553160c7e9fba4e8405923071a188813c6cae6ec) for more info.



