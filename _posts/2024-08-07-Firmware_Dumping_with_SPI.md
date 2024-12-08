---
title: "Firmware Dumping with SPI"
date: 2024-09-06 12:00:00
categories: [Hardware, Electronics]
tags: [hardware, electronics, spi, isp]     # TAG names should always be lowercase
published: true
---

# Summary
Take an old home router, find the microchip containing the filesystem, and dump it over SPI (Serial Peripheral Interface) using a [Glasgow Interface Explorer](https://glasgow-embedded.org). 

# Context
I'll be posting the teardown of this device soon, keep an eye out for it.

I've done SPI dumping and flashing once before, but using different programmer hardware, with a GUI, and it was on Windows. And someone was teaching me how to do it. In this post I'll be relearning how to do it myself using the [Glasgow Interface Explorer](https://glasgow-embedded.org), a versatile FPGA device that can be programmed to do lots of different things. 


# Step 1 - Identify the chip
In a future post, I'll explain how to perform a teardown of devices like this, including looking at each chip and component on the PCB, determining what they do, and developing an initial attack surface for hardware penetration testing purposes. For now, I want to focus on the Firmware dumping workflow, so I have already identified which chip is our target. See photo below. 

![Photo of Cfeon Q64-104hip in situ on TPLink Router main board](/images/tplink_router/cfeon-qhs4a-104hip-serial-flash.png)

Looking at the chip up close, we can read off the manufacturer and the model: cFeon QH64A-104HIP.
Googling this, we find it is this chip:
Cfeon Q64-104hip - 64 Megabit Serial Flash Memory
[Datasheet PDF](https://www.alldatasheet.com/datasheet-pdf/view/313095/EON/EN56Q64-104HIP.html)


# Step 2 - Confirm SPI is supported
Page 1 of the [datasheet](https://www.alldatasheet.com/datasheet-pdf/view/313095/EON/EN56Q64-104HIP.html) says this chip supports SPI. We definitely can use that to dump and flash (Download and Upload) data to/from this chip. 

# Step 3 - Powering the Chip - Device power or external power?

To interact with the chip using SPI, we actually need to power the chip. There's a couple of different ways to do this:

1. Turn the device on, and let it power the chip
- Pros:
  - No risk of over-volting or under-volting the device
  - No extra power supply needed
- Cons:
  - The ChipSelect (CS) pin must be pulled down, and sometimes the device pulls this pin high to prevent line noise or interference, which means extra steps are needed to pull CS down. 
2. Using some external device (for us, the Glasgow) power the device ourselves
- Pros:
  - Don't need to worry about line noise or pulling CS down with resistors
- Cons:
  - Not all hobby grade SPI capable devices can supply variable voltages out of the box - for example, using an arduino without a separate voltage shifter can only supply 5v, which is too high for our chip.

# Step 4 - Identify the SPI Pins

Reading the [Datasheet](https://www.alldatasheet.com/datasheet-pdf/view/313095/EON/EN56Q64-104HIP.html), we find this diagram on page 2:
![serial flash output](/images/tplink_router/serial_flash_pinout.png)

Then on page 3 we find a table with descriptions of each pin:
![cfeon flash pin names](/images/tplink_router/cfeon_flash_pin_names.png)

Page 5 then has detailed info on each pin. 

Page 8 has info on the supported SPI modes. 

Theres also some useful info at the top of page 1:
> The EN25QH64A is a 64 Megabit (8,192K-byte) Serial Flash memory, with advanced write protection mechanisms. The EN25QH64A supports the single bit and four bits serial input and output commands via standard Serial Peripheral Interface (SPI) pins: Serial Clock, Chip Select, Serial DQ0 (DI) and DQ1(DO), DQ2(WP#) and DQ3(HOLD#/RESET#). SPI clock frequencies of up to 104MHz are supported allowing equivalent clock rates of 416MHz (104Mhz x 4) for Quad Output while using the Quad Output Read instructions. The memory can be programmed 1 to 256 bytes at a time, using the Page Program instruction.

So the chip supports write protection - but we don't know if it's enabled on our specific chip. For now we just want to dump the firmware, not flash any, so we'll just note this and move on. 


## Step 4.1 - Choose which pins we're using

Ok - Next, we need to know which pins we're actually going to use.

I want to start simple to reduce the complexity of this task, so I'm not going to use any of the faster SPI modes.

Doing basic SPI, we need 4 of the 8 pins present on our cFeon chip: Chip Select, Data Out, Data In, and Clock.

Additionally, since we've decided to power the chip ourselves (see Step 3 - Powering the chip), we also need the VCC and Ground pins. Since the Glasgow will be supplying our VCC voltage, we need to connect the chips GND pin to our Glasgows GND to establish a common ground plane.

So the chip pins we'll be connecting to are:
- Pin 1 - Chip Select (AKA: CS, CE)
- Pin 2 - Data Out (AKA: DO, SDO, CIPO, POCI, MISO)
- Pin 4 - VSS (AKA: Ground)
- Pin 5 - Data In (AKA: DI, SDI, COPI, PICO, MOSI)
- Pin 6 - Clock (AKA: CLK, SCK)
- Pin 8 - VCC (AKA: Supply Voltage)

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> Note that Pin 1 is always adjacent the dot/circle in one of the corners of the chip housing. 
{: .prompt-info }
<!-- markdownlint-restore -->



# Step 5 - Choose test clips for the flash chip

Ideally we would use something like this [SOIC 8 pin Test Clip](https://www.sparkfun.com/products/13153) to just clip right onto the chip. I don't have one of those, so let's explore some other options. 

<b>[IC Hook Test Clips / Parrot Clips](https://www.sparkfun.com/products/9741)</b> - This is what I ended up using. They clip directly onto the legs of the chip, which makes for a stable connection, but risks damaging the legs or their attachment to the PCB if handled roughly. I have some that came with my [Saleae logic analyser](https://www.saleae.com) but you can buy them super cheap online. 

<b>[PCBite Test Probes](https://sensepeek.com/pcbite-20)</b> - I didn't use these probes, but I did use the magnetic plate and holders to keep everything from shifting around too much on my desk. I originally planned to use these probes, and they are great for some use cases, but was worried about them slipping off the legs, or scratching PCB traces. They're fantastic if all your chips are BGA[^1] and all you have access to is vias[^2] or you need to scrape some of the Conformal Coating[^3] off a trace to access it, but in this case we have convenient chip legs exposed we could grab onto. The basic set is also pretty expensive. 

<b>[Chip Socket](https://au.mouser.com/ProductDetail/Chip-Quik/PA-SOCKET-SOICN-8-127) and Desoldering the chip </b> - This is a good solution if you're going to be doing a lot of testing on the chip, or before you assemble a PCB. But theres always risk of overheating or otherwise damaging the PCB or chip itself when you start desoldering things. It also means if you want to use or test the device, you either have to resolder the chip or bridge the chip holder the PCB somehow. So for our use case, this solution is unneccessary and more effort than its worth. 

[^1]: BGA - [Ball Grid Array](https://en.wikipedia.org/wiki/Ball_grid_array) - A type of space efficient chip packaging where the connection points are not easily accessible and generally require desoldering to interface with. 
[^2]: [PCB Vias](https://en.wikipedia.org/wiki/Via_(electronics)) - Small conductive holes that facilitate circuits spanning different layers of a multilayer PCB. Can also be a convenient place to anchor a probe during testing. 
[^3]: [Conformal coating](https://en.wikipedia.org/wiki/Conformal_coating) - A durable semi transparent coating applied over top of etched PCBs that protect circuits from damage and short circuiting.

There's probably a bunch of other options, but these are the ones I'm familiar with. 

Let's also settle on a colour scheme for our pins, this will make things easier to verify and debug.
- Pin 1 - Chip Select - <span style="font-weight:bold; background-color:blue; color:blue">---</span> Blue
- Pin 2 - Data Out - <span style="font-weight:bold; background-color:purple; color:purple">---</span> Purple
- Pin 4 - VSS/GND - <span style="font-weight:bold; background-color:green; color:green">---</span> Green
- Pin 5 - Data In - <span style="font-weight:bold; background-color:yellow; color:yellow">---</span> Yellow
- Pin 6 - Clock - <span style="font-weight:bold; background-color:black; color:black">---</span> Black
- Pin 8 - VCC - <span style="font-weight:bold; background-color:red; color:red">---</span> Red
![cfeon connected pins annotated](/images/tplink_router/cfeon-connected-pins-annotated.png)


# Step 6 - Prepare SPI Programming hardware

Continuing with the last few blog posts, we're going to be using the [Glasgow Interface Explorer](https://glasgow-embedded.org) as our hardware interface to the flash chip. 

The Glasgow is an FPGA, and can be loaded with applets specific to a goal. See my previous [SWD with Glasgow](https://cattusqq.github.io/posts/SWD_with_Glasgow/) post for examples of using the `uart` and `swd-openocd` applets on the Glasgow.


For todays exercise, we need the spi-flashrom applet loaded to the Glasgow. I chose this applet by running `glasgow -h` and choosing the most appropriate applet from the output.

```bash
$ glasgow --version
$ glasgow build --rev C3 spi-flashrom
```

Maybe I should do a post on FPGAs generally at some point. Let me know in the comments if you'd be interested in that!

And connecting our coloured leads to the glasgow looks like this:
![](/images/tplink_router/glasgow_spi_pinout.png)

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> The first column of two pins in the image above is special - the top one is Voltage, which we have connected the red wire to, with the rest of the pins in that column being GND. The second pin in the first column is SENSE, with the rest of the pins on that row being our general purpose pins. 
I don't know what SENSE is for, but we don't need it for this activity.
{: .prompt-info }
<!-- markdownlint-restore -->



# Step 7 - Attempt connecting to the Flash Chip

Ok - We're getting to the pointy end now!

Let's see the spi-flashrom applets help text to give us an idea where to go from here.

```
$ glasgow run spi-flashrom -h
usage: glasgow run spi-flashrom [-h] [--port SPEC] [-f FREQ] [--sck-idle LEVEL]
                                [--sck-edge EDGE] [--cs-active LEVEL]
                                [--pin-cs NUM] [--pin-cipo NUM] [--pin-wp NUM]
                                [--pin-copi NUM] [--pin-sck NUM]
                                [--pin-hold NUM]
                                (-V [VOLTS] | -M | --keep-voltage)
                                ENDPOINT

Expose SPI via a socket using the flashrom serprog protocol; see
https://www.flashrom.org.

Usage:

    glasgow run spi-flashrom -V 3.3 --pin-cs 0 --pin-cipo 1 --pin-copi 2 --pin-sck 3 \
        --freq 4000 tcp::2222
    /sbin/flashrom -p serprog:ip=localhost:2222

It is also possible to flash 25-series flash chips using the `memory-25x`
applet, which does not require a third-party tool. The advantage of using the
`spi-flashrom` applet is that flashrom offers compatibility with a wider variety
of devices, some of which may not be supported by the `memory-25x` applet.

positional arguments:
  ENDPOINT                  listen at ENDPOINT, either unix:PATH or
                            tcp:HOST:PORT

options:
  -h, --help                show this help message and exit

build arguments:
  --port SPEC               bind the applet to port SPEC (default: AB)
  -f FREQ, --frequency FREQ
                            set SPI clock frequency to FREQ kHz (default: 100)
  --sck-idle LEVEL          set idle clock level to LEVEL (default: 0)
  --sck-edge EDGE           latch data at clock edge EDGE (default: rising)
  --cs-active LEVEL         set active chip select level to LEVEL (default: 0)
  --pin-cs NUM              bind the applet I/O line 'cs' to pin NUM (default:
                            0)
  --pin-cipo NUM            bind the applet I/O line 'cipo' to pin NUM (default:
                            1)
  --pin-wp NUM              bind the applet I/O line 'wp' to pin NUM (default:
                            2)
  --pin-copi NUM            bind the applet I/O line 'copi' to pin NUM (default:
                            3)
  --pin-sck NUM             bind the applet I/O line 'sck' to pin NUM (default:
                            4)
  --pin-hold NUM            bind the applet I/O line 'hold' to pin NUM (default:
                            5)

run arguments:
  -V [VOLTS], --voltage [VOLTS]
                            set I/O port voltage explicitly
  -M, --mirror-voltage      sense and mirror I/O port voltage
  --keep-voltage            do not change I/O port voltage
```

The help text says:
> Expose SPI via a socket using the flashrom serprog protocol; see
https://www.flashrom.org.

So lets take a look at that [link](https://www.flashrom.org).

> flashrom is a utility for detecting, reading, writing, verifying and erasing flash chips. 

Ok, looks like thats something we need to install.

## Step 7.1 - Install flashrom
I'm using MacOS, so I used Homebrew to install Flashrom. I got version 1.4.0

`$ brew install flashrom`


## Step 7.2 - Figure out what commands to use

The Glasgow help text above gave us a sample command:

```
$ glasgow run spi-flashrom -V 3.3 --pin-cs 0 --pin-cipo 1 --pin-copi 2 --pin-sck 3 \
    --freq 4000 tcp::2222

$ /sbin/flashrom -p serprog:ip=localhost:2222
```

Let's modify it for our use case. 

Our chip datasheet should tell us the voltage. The very first "feature" listed on page one says:
```
- Single power supply operation
  - Full voltage range: 2.7-3.6 volt
```

3.3 volts, suggested by the glasgow help text, is in that range, so I guess that's a standard SPI voltage value to use? Seems like it won't hurt the chip even if it's wrong, so let's try it. 

In step 3.2 we identified which pins we're using; I'll copy it here for convenience. 
```
- Pin 1 - Chip Select (AKA: CS, CE)
- Pin 2 - Data Out (AKA: DO, SDO, CIPO, POCI, MISO)
- Pin 4 - VSS (AKA: Ground)
- Pin 5 - Data In (AKA: DI, SDI, COPI, PICO, MOSI)
- Pin 6 - Clock (AKA: CLK, SCK)
- Pin 8 - VCC (AKA: Supply Voltage)
```

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> We indexed our pins on the chip from 1, whereas the glasgow app expects them indexed from 0. Keep track of this, or choose one to standardise in your notes to avoid confusion.
{: .prompt-warning }
<!-- markdownlint-restore -->

Ok, so our commands will be:

```
$ glasgow run spi-flashrom -V 3.3 --pin-cs 0 --pin-cipo 1 --pin-copi 2 --pin-sck 3 --freq 4000 tcp::2222

$ flashrom -p serprog:ip=localhost:2222

```

Brew added flashrom to my path so I'm not using an absolute path, but you can find it using `which` if it's on your path - ie `$ which flashrom`. Mine installed to `/opt/homebrew/bin/flashrom` for reference.

Before I run the commands, I want to check what they will actually do. Will they do a read or write? Will they just detect the chip and print some basic info about it? Will they do nothing and drop me into some interactive cli? At this stage, I don't want to accidentally brick my device because I blindly ran a command that interfaces directly with my flash chip.

Checking the flashrom help:
```
$ flashrom -h
 -p | --programmer <name>[:<param>] specify the programmer device. One of
    dummy, raiden_debug_spi, ft2232_spi, serprog, buspirate_spi, dediprog,
    developerbox, pony_spi, usbblaster_spi, pickit2_spi, ch341a_spi, ch347_spi,
    digilent_spi, stlinkv3_spi, dirtyjtag_spi.

```
Ok, so we're just specifying what kind of device our glasgow is acting as and performing any actions requires additional switches. Hmmm. I wonder if we can use one of the existing devices, or if this will be a similar case to my previous post on [SWD with Glasgow](https://cattusqq.github.io/posts/SWD_with_Glasgow/) where we had to checkout a specific commit of OpenOCD because the current release didn't include SWD support for the adapter driver used by the Glasgow hardware yet. [^4]

[^4]: While the Glasgow has been out for a little while now, it's still gaining popularity and some features are 'mostly' implemented, or fully implemented in the Glasgow but glasgow-support hasnt been added to other software yet. This is just a factor of it being a new and volunteer based project and should improve over time.  

Let's see if there's any other flashrom documentation we can find on the `serprog` option since that's the one in the Glasgow example. 

The official Flashrom documentation appears to be [here](https://www.flashrom.org/), and it has a page just for [serprog](https://www.flashrom.org/supported_hw/supported_prog/serprog/overview.html). It mentions various devices that can be used with the serprog protocol, but the Glasgow is not on the list. If I can get it working, I might contact them to update their page. 

So it seems like serprog is a protocol that can be used by many devices, and since we arent yet telling flashrom to actually do anything, I suspect if this command works, it will briefly connect, collect some info on the chip which will allow us to verify we are properly connected, then return and exit. Then we can tweak the flashrom command from there to actually 'do stuff'.

Alright, give it a try!


```
$ glasgow run spi-flashrom -V 3.3 --pin-cs 0 --pin-cipo 1 --pin-sck 2 --pin-copi 3 --freq 4000 tcp::2222
I: g.device.hardware: generating bitstream ID f4f6e944669fb61f8d3b2f1e3d914444
I: g.cli: running handler for applet 'spi-flashrom'
I: g.applet.interface.spi_flashrom: port(s) A, B voltage set to 3.3 V
I: g.applet.interface.spi_flashrom: socket: listening at tcp:localhost:2222
I: g.applet.interface.spi_flashrom: socket: new connection from [127.0.0.1]:50506
I: g.applet.interface.spi_flashrom: socket: connection from [127.0.0.1]:50506 lost

```

```
$ flashrom -p serprog:ip=localhost:2222
flashrom 1.4.0 on Darwin 22.4.0 (arm64)
flashrom is free software, get the source code at https://flashrom.org

Error: Could not flush serial port incoming buffer: Device not configured
serprog: Programmer name is "Glasgow serprog"
serprog: requested mapping AT45CS1282 is incompatible: 0x1080000 bytes at 0x00000000fef80000.
serprog: requested mapping GD25LQ255E is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping GD25LB256E/GD25LR256E is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping GD25LR512ME is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping GD25Q256D/GD25Q256E is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping IS25LP256 is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping IS25WP256 is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping MX25L25635F/MX25L25645G is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping MX25U25635F is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping MX25U25643G is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping MX25U51245G is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping MX66L51235F/MX25L51245G is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping MX66L1G45G is incompatible: 0x8000000 bytes at 0x00000000f8000000.
serprog: requested mapping MX77L25650F is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping N25Q00A..1G is incompatible: 0x8000000 bytes at 0x00000000f8000000.
serprog: requested mapping N25Q00A..3G is incompatible: 0x8000000 bytes at 0x00000000f8000000.
serprog: requested mapping N25Q256..1E is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping N25Q256..3E is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping N25Q512..1G is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping N25Q512..3G is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping MT25QL01G is incompatible: 0x8000000 bytes at 0x00000000f8000000.
serprog: requested mapping MT25QU01G is incompatible: 0x8000000 bytes at 0x00000000f8000000.
serprog: requested mapping MT25QL02G is incompatible: 0x10000000 bytes at 0x00000000f0000000.
serprog: requested mapping MT25QU02G is incompatible: 0x10000000 bytes at 0x00000000f0000000.
serprog: requested mapping MT25QL256 is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping MT25QU256 is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping MT25QL512 is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping MT25QU512 is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping S25FL256L is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping S25FL256S......0 is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping S25FL512S is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping W25Q256FV is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping W25Q256JV_Q is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping W25Q256JV_M is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping W25Q256JW is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping W25Q256JW_DTR is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping W25Q512JV is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping W25Q512NW-IM is incompatible: 0x4000000 bytes at 0x00000000fc000000.
serprog: requested mapping XM25QH256C is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping XM25QU256C is incompatible: 0x2000000 bytes at 0x00000000fe000000.
serprog: requested mapping XM25RU256C is incompatible: 0x2000000 bytes at 0x00000000fe000000.
No EEPROM/flash device found.
Note: flashrom can never write if the flash chip isn't found automatically.
```

Hmm, it looks like flashrom is talking to our chip, but a mapping for our chip was not automatically found. So I'm guessing our options are:

1. Double check we've connected everything correctly, and actually have the correct chip pinout
2. Write a mapping for flashrom - see [here](https://riverloopsecurity.com/blog/2021/07/flashrom/) for a guide. 
3. Find an existing mapping that matches our chip and just explicitly select it.

For 2, it looks like the chip vendor ID is already in flashrom - see [source](https://github.com/flashrom/flashrom/blob/00e02a61840d0f230d25f8988d2f30100ae1388d/include/flashchips.h#L298).


Chip device id:
.name		= "EN25QH64",
https://github.com/flashrom/flashrom/blob/00e02a61840d0f230d25f8988d2f30100ae1388d/flashchips.c#L5430

.name		= "EN25QH64A",
https://github.com/flashrom/flashrom/blob/00e02a61840d0f230d25f8988d2f30100ae1388d/flashchips.c#L5474


`$ flashrom -p serprog:ip=localhost:15000 -c EN25QH64A`


The datasheet does actually tell us the mmanufacturer id and chip id on pg 16.

![](/images/tplink_router/cefeon-manufactuerANDdeviceid-opcodes-abh-90h-9fh.png)

This is the flashrom chip id entry for our chip: 
[Permalink to source code](https://github.com/flashrom/flashrom/blob/00e02a61840d0f230d25f8988d2f30100ae1388d/include/flashchips.h#L298)
```
line 298: #define EON_EN25QH64		0x7017
```
And it does specify the correct hex bytes of `0x7017` for the chip id. 

So option 2 isn't going to help us, lets cross that off the list. 

After trying the command a few more times, with and without the verbose switch, I was seeing weird values for the chip being detected: manufacturer ids of 0x1f with chip id of 0xffff, then after jiggling the cables it changed to 0x00 and 0x0000. This does seem more like a connection issue than a software one. 

So lets look at option 1, double checking all of our connections. 


## Step 7.3 - Remember that SPI is sensitive

Why is flashrom not detecting the correct values for our chips id?

I thought back to a few years ago when a friend was showing me how to do SPI using an XGECU programmer. I remember he said the cable lengths have to all be the same, and you get less interference if the cables are all bundled together or in a ribbon where the individual cables hadn't been pulled apart yet. It didn't really make sense to me at the time why there would be less interference with them all next to each other, and honestly it still doesn't, but I followed the advice anyway. 

I had a bit of a frankenstein situation going on, where I had female-to-female jumper wires connected to the clips that were on the chip legs, then a male-to-female connecting that to the glasgow. I figured that additional interface might be causing issues, so took out all the male-to-female cables and made sure all the female-to-female ones were all the same length. 


Annnnnd it worked!

The total length of the frankenstein cables was about 20cm, which is right on the cusp of what is recommended - the shorter the cable, the better your chance with SPI. 

So I replaced the multi hop cables with single short 10cm cables..... and it worked!

```
glasgow run spi-flashrom -V 3.3 --port A --pin-cs 0 --pin-cipo 1 --pin-sck 2 --pin-copi 3 --freq 4000 tcp::15000
I: g.device.hardware: generating bitstream ID 3ebde1c08549a8aeb2e3c08de069e196
I: g.cli: running handler for applet 'spi-flashrom'
I: g.applet.interface.spi_flashrom: port(s) A voltage set to 3.3 V
I: g.applet.interface.spi_flashrom: socket: listening at tcp:localhost:15000
I: g.applet.interface.spi_flashrom: socket: new connection from [127.0.0.1]:54439
I: g.applet.interface.spi_flashrom: socket: connection from [127.0.0.1]:54439 lost
```


```
$ flashrom -p serprog:ip=localhost:15000 --flash-name --verbose
flashrom 1.4.0 on Darwin 22.4.0 (arm64)
flashrom is free software, get the source code at https://flashrom.org

flashrom was built with LLVM Clang 14.0.3 (clang-1403.0.22.14.1), little endian
Command line (4 args): flashrom -p serprog:ip=localhost:15000 --flash-name --verbose
Initializing serprog programmer
serprog: IP localhost port 15000
serprog: connected - attempting to synchronize
Error: Could not flush serial port incoming buffer: Device not configured
.
serprog: Synchronized
serprog: Interface version ok.
serprog: Bus support: parallel=off, LPC=off, FWH=off, SPI=on
Warning: Automatic command availability check failed for cmd 0x08 - won't execute cmd
Warning: Automatic command availability check failed for cmd 0x11 - won't execute cmd
serprog: Programmer name is "Glasgow serprog"
serprog: Serial buffer size is 65535
serprog: Warning: Programmer does not support toggling its output drivers
The following protocols are supported: SPI.
Probing for AMIC A25L010, 128 kB: compare_id: id1 0x1c, id2 0x7017
Probing for AMIC A25L016, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for AMIC A25L020, 256 kB: compare_id: id1 0x1c, id2 0x7017
...
Output truncated for brevity
...
Probing for Boya/BoHong Microelectronics B.25D16A, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Boya/BoHong Microelectronics B.25D80A, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Boya/BoHong Microelectronics B.25Q64AS, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Boya/BoHong Microelectronics B.25Q128AS, 16384 kB: compare_id: id1 0x1c, id2 0x7017
Probing for ESI ES25P16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for ESI ES25P40, 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for ESI ES25P80, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for ESMT F25L008A, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for ESMT F25L32PA, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B05, 64 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B05T, 64 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B10, 128 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B10T, 128 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B16T, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B20, 256 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B20T, 256 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B32, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B32T, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B40, 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B40T, 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B64, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B64T, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B80, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25B80T, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F05, 64 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F10, 128 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F20, 256 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F32, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F40, 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F64, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25F80, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P05, 64 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P10, 128 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P20, 256 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P32, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P40, 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P64, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25P80, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25Q128, 16384 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25Q16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25Q32(A/B), 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25Q40, 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25Q64, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25Q80(A), 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25QH128, 16384 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25QH16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25QH32, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25QH32B, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25QH64, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Added layout entry 00000000 - 007fffff named complete flash
Found Eon flash chip "EN25QH64" (8192 kB, SPI) on serprog.
Chip status register is 0x00.
Chip status register: Status Register Write Disable (SRWD, SRP, ...) is not set
Chip status register: Bit 6 is not set
Chip status register: Block Protect 3 (BP3) is not set
Chip status register: Block Protect 2 (BP2) is not set
Chip status register: Block Protect 1 (BP1) is not set
Chip status register: Block Protect 0 (BP0) is not set
Chip status register: Write Enable Latch (WEL) is not set
Chip status register: Write In Progress (WIP/BUSY) is not set
Probing for Eon EN25QH64A, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Added layout entry 00000000 - 007fffff named complete flash
Found Eon flash chip "EN25QH64A" (8192 kB, SPI) on serprog.
Chip status register is 0x00.
Chip status register: Status Register Write Disable (SRWD, SRP, ...) is not set
Chip status register: Bit 6 is not set
Chip status register: Block Protect 3 (BP3) is not set
Chip status register: Block Protect 2 (BP2) is not set
Chip status register: Block Protect 1 (BP1) is not set
Chip status register: Block Protect 0 (BP0) is not set
Chip status register: Write Enable Latch (WEL) is not set
Chip status register: Write In Progress (WIP/BUSY) is not set
Probing for Eon EN25S10, 128 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25S16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25S20, 256 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25S32, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25S40, 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25S64, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Eon EN25S80, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Fudan FM25F005, 64 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Fudan FM25F01, 128 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Fudan FM25F02(A), 256 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Fudan FM25F04(A), 512 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Fudan FM25Q08, 1024 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Fudan FM25Q16, 2048 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Fudan FM25Q32, 4096 kB: compare_id: id1 0x1c, id2 0x7017
Probing for GigaDevice GD25B128B/GD25Q128B, 16384 kB: compare_id: id1 0x1c, id2 0x7017
...
Output truncated for brevity
...
Probing for Sanyo unknown Sanyo SPI chip, 0 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Winbond unknown Winbond (ex Nexcom) SPI chip, 0 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Generic unknown SPI chip (RDID), 0 kB: compare_id: id1 0x1c, id2 0x7017
Probing for Generic unknown SPI chip (REMS), 0 kB: compare_id: id1 0x1c, id2 0x16
Multiple flash chip definitions match the detected chip(s): "EN25QH64", "EN25QH64A"
Please specify which chip definition to use with the -c <chipname> option.
```

Huzzah! Its detected the chip manufacturer and chip id, so looks like that was our problem the whole time!
Thats a lesson I won't soon forget. 

Lets keep going. We have to choose one of the two chip profiles it auto matched us with. I chose the `EN25QH64A` because the markings on the chip have a `64A` in them: `QH64A-10HIP`.


```
$ flashrom -p serprog:ip=localhost:15000 --flash-name --verbose -c EN25QH64A
flashrom 1.4.0 on Darwin 22.4.0 (arm64)
flashrom is free software, get the source code at https://flashrom.org

flashrom was built with LLVM Clang 14.0.3 (clang-1403.0.22.14.1), little endian
Command line (6 args): flashrom -p serprog:ip=localhost:15000 --flash-name --verbose -c EN25QH64A
Initializing serprog programmer
serprog: IP localhost port 15000
serprog: connected - attempting to synchronize
Error: Could not flush serial port incoming buffer: Device not configured
.
serprog: Synchronized
serprog: Interface version ok.
serprog: Bus support: parallel=off, LPC=off, FWH=off, SPI=on
Warning: Automatic command availability check failed for cmd 0x08 - won't execute cmd
Warning: Automatic command availability check failed for cmd 0x11 - won't execute cmd
serprog: Programmer name is "Glasgow serprog"
serprog: Serial buffer size is 65535
serprog: Warning: Programmer does not support toggling its output drivers
The following protocols are supported: SPI.
Probing for Eon EN25QH64A, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Added layout entry 00000000 - 007fffff named complete flash
Found Eon flash chip "EN25QH64A" (8192 kB, SPI) on serprog.
Chip status register is 0x00.
Chip status register: Status Register Write Disable (SRWD, SRP, ...) is not set
Chip status register: Bit 6 is not set
Chip status register: Block Protect 3 (BP3) is not set
Chip status register: Block Protect 2 (BP2) is not set
Chip status register: Block Protect 1 (BP1) is not set
Chip status register: Block Protect 0 (BP0) is not set
Chip status register: Write Enable Latch (WEL) is not set
Chip status register: Write In Progress (WIP/BUSY) is not set
This chip may contain one-time programmable memory. flashrom cannot read
and may never be able to write it, hence it may not be able to completely
clone the contents of this chip (see man page for details).
===
This flash part has status UNTESTED for operations: WP
The test status of this chip may have been updated in the latest development
version of flashrom. If you are running the latest development version,
please email a report to flashrom@flashrom.org if any of the above operations
work correctly for you with this flash chip. Please include the flashrom log
file for all operations you tested (see the man page for details), and mention
which mainboard or programmer you tested in the subject line.
You can also try to follow the instructions here:
https://www.flashrom.org/contrib_howtos/how_to_mark_chip_tested.html
Thanks for your help!
vendor="Eon" name="EN25QH64A"
```
Huzzzaaaaah!

Interestingly, it looks like write-protection is not set on this chip. We might be able to modify the firmware and reflash it over top. Exciting!

# Step 8 - Dump the firmware
And finally, lets actually try to read the flash and dump to file. flashroms help text tells us how to do this. 

```
$ flashrom -h
...
 -r | --read <file>                 read flash and save to <file>
...
```

```
$ flashrom -p serprog:ip=localhost:15000 --verbose -c EN25QH64A --read ./cfeon_flash_dump.out
flashrom 1.4.0 on Darwin 22.4.0 (arm64)
flashrom is free software, get the source code at https://flashrom.org

flashrom was built with LLVM Clang 14.0.3 (clang-1403.0.22.14.1), little endian
Command line (7 args): flashrom -p serprog:ip=localhost:15000 --verbose -c EN25QH64A --read ./cfeon_flash_dump.out
Initializing serprog programmer
serprog: IP localhost port 15000
serprog: connected - attempting to synchronize
Error: Could not flush serial port incoming buffer: Device not configured
.
serprog: Synchronized
serprog: Interface version ok.
serprog: Bus support: parallel=off, LPC=off, FWH=off, SPI=on
Warning: Automatic command availability check failed for cmd 0x08 - won't execute cmd
Warning: Automatic command availability check failed for cmd 0x11 - won't execute cmd
serprog: Programmer name is "Glasgow serprog"
serprog: Serial buffer size is 65535
serprog: Warning: Programmer does not support toggling its output drivers
The following protocols are supported: SPI.
Probing for Eon EN25QH64A, 8192 kB: compare_id: id1 0x1c, id2 0x7017
Added layout entry 00000000 - 007fffff named complete flash
Found Eon flash chip "EN25QH64A" (8192 kB, SPI) on serprog.
Chip status register is 0x00.
Chip status register: Status Register Write Disable (SRWD, SRP, ...) is not set
Chip status register: Bit 6 is not set
Chip status register: Block Protect 3 (BP3) is not set
Chip status register: Block Protect 2 (BP2) is not set
Chip status register: Block Protect 1 (BP1) is not set
Chip status register: Block Protect 0 (BP0) is not set
Chip status register: Write Enable Latch (WEL) is not set
Chip status register: Write In Progress (WIP/BUSY) is not set
This chip may contain one-time programmable memory. flashrom cannot read
and may never be able to write it, hence it may not be able to completely
clone the contents of this chip (see man page for details).
===
This flash part has status UNTESTED for operations: WP
The test status of this chip may have been updated in the latest development
version of flashrom. If you are running the latest development version,
please email a report to flashrom@flashrom.org if any of the above operations
work correctly for you with this flash chip. Please include the flashrom log
file for all operations you tested (see the man page for details), and mention
which mainboard or programmer you tested in the subject line.
You can also try to follow the instructions here:
https://www.flashrom.org/contrib_howtos/how_to_mark_chip_tested.html
Thanks for your help!
Reading flash... read_flash:  region (00000000..0x7fffff) is readable, reading range (00000000..0x7fffff).
done.
```

Woohoo!! 

# Step 9 - Verify the dumped data

Now, before we get too excited, we have to check that what we just downloaded isn't gobbledeegook. I've always had pretty good success with binutils binwalk, but there are definitely other options out there. 

I installed binutils via homebrew with `brew install binwalk`, but if youre on linux you'll need to install binutils, which is the package that contains binwalk. 

Ok, lets see if we can pull this binary file apart and look inside. We'll use the -e switch for extract. The result will be saved to a folder that has the same name as the file you extracted, but with an underscore prepended. 

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> Sometimes you need to install additional packages to enable binwalk to extract certain filesystem types, notably squashfs when running binwalk on linux systems. However on MacOS I didnt seem to need this. In any case, the binwalk output will have an error message that tells you how to resolve it, if it happens. 
{: .prompt-info }
<!-- markdownlint-restore -->



```
$ binwalk -e cfeon_flash_dump.out
/opt/homebrew/bin/binwalk:4: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  __import__('pkg_resources').run_script('binwalk==2.3.3', 'binwalk')

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
96144         0x17790         U-Boot version string, "U-Boot 1.1.3 (Nov  8 2021 - 08:59:56)"
111343        0x1B2EF         HTML document header
130938        0x1FF7A         HTML document footer
277200        0x43AD0         U-Boot version string, "U-Boot 1.1.3 (Nov  8 2021 - 08:59:53)"
328192        0x50200         LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, uncompressed size: 3441784 bytes

WARNING: Extractor.execute failed to run external extractor 'sasquatch -p 1 -le -d 'squashfs-root-0' '%e'': [Errno 2] No such file or directory: 'sasquatch', 'sasquatch -p 1 -le -d 'squashfs-root-0' '%e'' might not be installed correctly

WARNING: Extractor.execute failed to run external extractor 'sasquatch -p 1 -be -d 'squashfs-root-0' '%e'': [Errno 2] No such file or directory: 'sasquatch', 'sasquatch -p 1 -be -d 'squashfs-root-0' '%e'' might not be installed correctly


1638400       0x190000        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 6351600 bytes, 743 inodes, blocksize: 131072 bytes, created: 2021-11-08 01:16:51

```

Ok, there were a few warnings, but it seems to have completed. Lets check the output. 


```

$ cd _cfeon_flash_dump.out.extracted
$ ls
190000.squashfs	50200		50200.7z	squashfs-root	squashfs-root-0
$ cd squashfs-root
$ ls
bin	dev	etc	lib	linuxrc	mnt	proc	sbin	sys	usr	var	web
$ cd etc
$ ls
MT7613_EEPROM.bin		SingleSKU_5G_US_1.dat		TZ				iptables-stop
MT7628_AP_2T2R-4L_V15.BIN	SingleSKU_5G_VN.dat		default_config.xml		l1profile.dat
MT7628_EEPROM_20140317.bin	SingleSKU_BF_5G_CE.dat		default_config_CA.xml		passwd
RT2860AP.dat			SingleSKU_BF_5G_FCC.dat		default_config_EU.xml		passwd.bak
RT2860AP5G.dat			SingleSKU_CA.dat		default_config_JP.xml		ppp
SingleSKU_5G_CA.dat		SingleSKU_CE.dat		default_config_KR.xml		reduced_data_model.xml
SingleSKU_5G_CE.dat		SingleSKU_FCC.dat		default_config_TW.xml		resolv.conf
SingleSKU_5G_DE.dat		SingleSKU_JP.dat		default_config_US.xml		samba
SingleSKU_5G_FCC.dat		SingleSKU_KR.dat		default_config_VN.xml		services
SingleSKU_5G_JP.dat		SingleSKU_TW.dat		fstab				web
SingleSKU_5G_KR.dat		SingleSKU_US.dat		group				wifidog
SingleSKU_5G_TW.dat		SingleSKU_US_1.dat		init.d
SingleSKU_5G_US.dat		SingleSKU_VN.dat		inittab

```

Yaaaaaaas! 🥳🎉 That definitely looks like a recovered filesystem. 

Of course the first thing I did then was check /etc/passwd. Which was empty, however there was a file called /etc/passwd.bak, which contained an account hash.

```
$ cat passwd.bak
admin:$1$$iC.dUsGpxNNJGeOm1dFio/:0:0:root:/:/bin/sh
dropbear:x:500:500:dropbear:/var/dropbear:/bin/sh
nobody:*:0:0:nobody:/:/bin/sh
```

Googling it quickly, since this is an old device with old firmware, revealed this [security researcher blog post](https://pierrekim.github.io/blog/2017-02-09-tplink-c2-and-c20i-vulnerable.html) with the cracked out password.

![](/images/tplink_router/tplink%20hash%20and%20cracked%20md5%20pass%20.png)

So while our test device is old and its firmware is out of date and has since been patched, the value of doing this kind of 'firmware dumping' activity in a digital security context is pretty clear.

# Step 10 - Next steps

## Follow on blog post(s)
We know theres no write protection enabled on this chip, so a post on using the ability to update the chips firmware as an access vector to getting a root shell could be a fun exercise. 

If we didnt want to mess with writing to the chip, theres still a lot of vulnerability research we can do just with having the filesystem - we could look for bugs in implementations of networking components or security mechanisms. Then once we find something, we can test against the running device to verify the bug exists, and work up an exploit or capability. 

We could even use an emulator to emulate parts of the device, mocking out whichever components we're less interested in. This is great for testing and development as it can be faster and less expensive, but of course the gold standard of behaviour verification is still to use the device itself. 

## Be a good open source citizen 
- [ ] Get [this flashrom page](https://www.flashrom.org/supported_hw/supported_prog/serprog/overview.html) updated to include that the serprog protocol does work with the glasgow c3.
> UPDATE: Submitted Patch: https://review.coreboot.org/c/flashrom/+/85527/1

Submit PRs to flashrom
- [ ] print the expected chip id and received chip id clearly - the verbose output as it is, is kind of ambiguous, at least I struggled at first to understand what it was conveying in verbose mode. Maybe write a verbose-verbose option with more info?
- [ ] check if the manufacturer and chip ids are both all 0x0s, or -1(0x1f, which also happens to be Atmels code) AND the chip id is 0xFFFF (which is not a valid atmel code? confirm this), and if it is, suggest to the user to check the connection. Also, update the help text with this info. 
- [ ] make chip scanning more efficient - `Found Eon flash chip "EN25QH64" (8192 kB, SPI) on serprog.`. But then a few lines down we see its still scanning to see if we match other chip manufacturers. It should probably stop when it finds the first match. Like I get there might be multiple chips in one manufacturer that might match, but unless in the case of a manufacturer id collision (we know of at least one) it should ideally not run through the rest. Maybe theres a reason they do it this way, its more space efficient and its running on an fpga or something, but if this is the code in flashrom, it could potentially be improved. 

For the Glasgow:
- [ ] The original error I was getting about port 5 something Amaranth error, submit a PR suggesting to check your cables connections in that error message. Do some testing to confirm exactly what conditions (using two connected cables for each pin vs one, not having VCC and/or GND connected) triggered what errors and output.
- [ ] Also update the help text with troubleshooting info. 


# Useful Links
[Rapid7 - Memory Extraction from SPI Flash Devices](https://www.youtube.com/watch?v=ZCeisJ4zWp8)

[Hackaday - Taking a deep dive into SPI](https://hackaday.com/2021/09/16/taking-a-deep-dive-into-spi/)

[Hackyourmom - Part 8. Hacking the hardware part of the system. (SPI and I2C)](https://hackyourmom.com/en/osvita/chastyna-8-zlom-aparatnoyi-chastyny-systemy-spi-ta-i2c/)



# My environment
This section is for anyone who's trying to replicate my steps and has issues. You can check by making your environment the same as mine was in this guide. 


My Glasgow software version is 
`Glasgow 0.1.dev2051+gc97d2d8 (CPython 3.12.5 on macOS-13.3.1-arm64-arm-64bit)`

I'm using [Glasgow hardware revision c3](https://glasgow-embedded.org/latest/revisions/index.html#revc3).

Flashrom - v 1.4.0 - Installed via Homebrew. 

MacOS Sonoma 14.6.1 
