---
title: "Debugging with SWD and the Glasgow Interface Explorer PART 2"
date: 2024-08-17 12:00:00
categories: [Hardware, SWD, Debugging]
tags: [hardware, swd, glasgow, debugging]     # TAG names should always be lowercase
published: true
---
# Summary
A guide for using the Glasgow Interface Explorer to debug a Raspberry Pi Pico using SWD (Serial Wire Debug).  PART 2 of the SWD with Glasgow series.

# Context
In the last post, we figured out how to use the Glasgow Interface Explorer (an FPGA that can be programmed to be an SWD interface) and OpenOCD to flash a binary to a Raspberry Pi Pico using the SWD (Serial Wire Debug) transport protocol. 

In this post, we'll be taking things one step further, and debugging a running Pico application processor with GDB.

I expect this will be a much shorter post than the first one. 

# The plan

I'm going to debug the blink binary we compiled last time. I'm thinking I'll set a breakpoint somewhere that controls the LED actually flashing, that way we can visually confirm things are working. 

# Step 1 - Figure out the right commands to run

Since the Glasgow is our interface ie hardware SWD dongle, and doesn't care about the commands we send over it, I think the Glasgow commands and setup should be the same as last time. 

The OpenOCD command will change though. Previously, we told it to flash a binary to the device connected to the Glasgow. Now, we need to plumb a debugger through it somehow. 

Let's start by interrogating OpenOCD's help text and go from there. 

```
$ openocd --help
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Open On-Chip Debugger
Licensed under GNU GPL v2
--help       | -h	display this help
--version    | -v	display OpenOCD version
--file       | -f	use configuration file <name>
--search     | -s	dir to search for config files and scripts
--debug      | -d	set debug level to 3
             | -d<n>	set debug level to <level>
--log_output | -l	redirect log output to file <name>
--command    | -c	run <command>

$ openocd --help --command
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
openocd: option `--command' requires an argument

$ openocd --help --command program
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Open On-Chip Debugger
Licensed under GNU GPL v2
--help       | -h	display this help
--version    | -v	display OpenOCD version
--file       | -f	use configuration file <name>
--search     | -s	dir to search for config files and scripts
--debug      | -d	set debug level to 3
             | -d<n>	set debug level to <level>
--log_output | -l	redirect log output to file <name>
--command    | -c	run <command>

$

```
Well that wasn't very helpful. The man page similarly only covered the top level arguments but gave some other insights:
`OpenOCD is an on-chip debugging, in-system programming and boundary-scan testing tool for various ARM and MIPS systems.` Oh it does Mips also, interesting!
`User interaction is realized through a telnet command line interface, a gdb (the GNU debugger) remote protocol server, and a simplified RPC connection that can be used to interface with OpenOCD's Jim Tcl engine.` Ok, good to know. 
`OpenOCD supports various different types of JTAG interfaces/programmers, please check the openocd info page for the complete list.`. Ok, let's try that.

Scrolling through the help articles, I come to `* GDB and OpenOCD:: Using GDB and OpenOCD` Yes! 

```
20 GDB and OpenOCD
******************

OpenOCD complies with the remote gdbserver protocol and, as such, can be
used to debug remote targets.  Setting up GDB to work with OpenOCD can
involve several components:

   • The OpenOCD server support for GDB may need to be configured.
     *Note GDB Configuration: gdbconfiguration.
   • GDB's support for OpenOCD may need configuration, as shown in this
     chapter.
   • If you have a GUI environment like Eclipse, that also will probably
     need to be configured.

Of course, the version of GDB you use will need to be one which has been
built to know about the target CPU you're using.  It's probably part of
the tool chain you're using.  For example, if you are doing
cross-development for ARM on an x86 PC, instead of using the native x86
‘gdb’ command you might use ‘arm-none-eabi-gdb’ if that's the tool chain
used to compile your code.
```
This is the most beginner friendly language I've ever seen in a man or info page about cross compiling. Props to the author. And hopefully this bodes well for us. 

I'm gonna skip printing the entire info page here and just point out the useful bits. 

`20.2 Sample GDB session startup` very helpfully gives us an example command log for starting and driving GDB. 

```bash
     $ arm-none-eabi-gdb example.elf
     (gdb) target extended-remote localhost:3333
     Remote debugging using localhost:3333
     ...
     (gdb) monitor reset halt
     ...
     (gdb) load
     Loading section .vectors, size 0x100 lma 0x20000000
     Loading section .text, size 0x5a0 lma 0x20000100
     Loading section .data, size 0x18 lma 0x200006a0
     Start address 0x2000061c, load size 1720
     Transfer rate: 22 KB/sec, 573 bytes/write.
     (gdb) continue
     Continuing.
     ...
```
`20.3 Configuring GDB for OpenOCD` refers to `the GDB remote server (i.e.  OpenOCD)`. This kind of nomenclature is important to keep track of, because it can become pretty ambiguous sometimes what specific hardware or software component someone is actually referring to. 

That's it really. There was some other interesting stuff about how hardware breakpoints vary between different arm systems, for now I think we have everything useful we can glean. 

# Step 2 - Install the appropriate GDB client

We're going to be debugging for the Raspberry Pi Pico, which according to its [documentation](https://www.raspberrypi.com/documentation/microcontrollers/raspberry-pi-pico.html) is a `Dual-core Arm Cortex M0+ processor`, so the `arm-none-eabi-gdb` package suggested in the OpenOCD documentation should suffice for us. 

I'm on macOS so I'm going to install it with Homebrew. 

`$ brew install arm-none-eabi-gdb`

# Step 3 - Attempt to Debug the Pico

Set up our Glasgow bridge

```bash
$ glasgow run swd-openocd -V 3.3 --pin-swdio 0 --pin-swclk 1 tcp:localhost:2222
I: g.device.hardware: generating bitstream ID 495f556d820677effe84d423ece8f5a0
I: g.cli: running handler for applet 'swd-openocd'
I: g.applet.interface.swd_openocd: port(s) A, B voltage set to 3.3 V
I: g.applet.interface.swd_openocd: socket: listening at tcp:localhost:2222
```
Now, I think we just need to run GDB and we don't need OpenOCD. OpenOCD stands for "Open On Chip Debugger" and here, we're using GDB as our debugger instead. I could be wrong, but let's run with it for now. 

I'm annotating in the code block below with #.

```bash 
$ sudo arm-none-eabi-gdb blink.elf
Password:
GNU gdb (Arm GNU Toolchain 13.2.rel1 (Build arm-13.7)) 13.2.90.20231008-git
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=aarch64-apple-darwin20.6.0 --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://bugs.linaro.org/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from blink.elf...
(gdb) target extended-remote localhost:2222
Remote debugging using localhost:2222

# At this point you'll see a new connection on the glasgow side.

# Here I'm trying to issue the commands command from the OpenOCD gdb example.
monitor reset Ignoring packet error, continuing...
warning: unrecognized item "timeout" in "qSupported" response
halt
# Here I do a ctrl+c to drop to the gdb prompt
^CQuit
(gdb) monitor reset halt
"monitor" command not supported by this target.
(gdb) load
You can't do that when your target is `exec'
(gdb)

```

Hmm. What's going on here. Even though I'm in a GDB prompt, I would expect execution to be paused due to a hardware interrupt, but my Picos' LED is still blinking. 

I wonder if there's a way to interrogate which CPU I'm attached to via GDB. Maybe I'm connected to something other than the application processor?

After some googling, I came across [this blog post](https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html) which is debugging a Pico using GDB and OpenOCD, and a PicoProbe as the hardware. We're using a Glasgow instead of a PicoProbe, but the rest should transfer. 

So it looks like we do need to run OpenOCD after all. Alright, let's give that a shot. 

```bash
$ openocd -f interface/glasgow.cfg -f target/rp2040.cfg
```

Heres my glasgow_swd.cfg for reference
```bash
$ cd openocd/tcl/ 
$ cat interface/glasgow_swd.cfg
adapter driver remote_bitbang
transport select swd
remote_bitbang port 2222

$
```
Note that you can instead pass each of those lines into OpenOCD as commands instead of specifying an interface file. Just be aware that the order matters, so if you mix them around and it doesn't work, try the command below verbatim as it worked for me. 
Also Note, when I run this, it reports swd transport is assumed so I didn't need to specify that. 

```bash
$ openocd -c "adapter driver remote_bitbang" -c "remote_bitbang port 2222" -f target/rp2040.cfg
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : Hardware thread awareness created
Info : Hardware thread awareness created
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Initializing remote_bitbang driver
Info : Connecting to localhost:2222
Info : remote_bitbang driver initialized
Info : Note: The adapter "remote_bitbang" doesn't support configurable speed
```

Ok, now let's try GDB again.

```bash
$ sudo arm-none-eabi-gdb blink.elf
GNU gdb (Arm GNU Toolchain 13.2.rel1 (Build arm-13.7)) 13.2.90.20231008-git
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=aarch64-apple-darwin20.6.0 --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://bugs.linaro.org/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from blink.elf...
(gdb) target extended-remote localhost:2222
Remote debugging using localhost:2222
Ignoring packet error, continuing...
warning: unrecognized item "timeout" in "qSupported" response
```

As soon as I ran `target extended-remote localhost:2222` the Glasgow reported a new connection then lost it right after, and the OpenOCD process exited after printing

```bash
Assertion failed: (!remote_bitbang_recv_buf_empty()), function remote_bitbang_read_sample, file remote_bitbang.c, line 200.
zsh: abort      openocd -c "adapter driver remote_bitbang" -c "remote_bitbang port 2222" -f
```

Sigh. It's never as easy as you hope. 

Before I look at the source code referenced in the error, I'll do a little googling in case there's a known issue with a simple solution. I found this on stackoverflow: [ocd assertion fault when connecting gdb](https://stackoverflow.com/questions/67041370/ocd-assertion-fault-when-connecting-gdb) 

It's using an STM instead of a Glasgow, but still might be relevant. 

The solution talks about specific hardware settings, and links to [a guide for using the stm](https://vjordan.info/log/fpga/first-steps-with-the-stm32f4.html). The guide very quickly descends into technical details on memory maps and clock settings and my eyes glaze over. 

<figure>
  <img src="/images/sigh.png" alt="my alt text"/>
  <center><figcaption>How I feel right now</figcaption></center>
</figure>

# Step 3.5 - Get frustrated and go on a wild goose chase

I took an hour break, had some coffee, came back and read the [RPi Pico tech docs](https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf) again. `Chapter 6. Debugging with SWD` walks us through `6.1. Build "Hello World" debug version`, `6.2. Installing GDB`, and `6.3. Use GDB and OpenOCD to debug Hello World`.

This actually didn't help at all, and I misread part of it that said you need a UART serial connection in order to debug using SWD, so I went on a wild goose chase setting that up, before realising, there's no console output in the single binary I compiled and flashed to the device, and there's no kernel, so there's nothing to drive UART, so what am I even doing again? 

Turns out I had missed the bit about only needing UART if you're debugging over the USB controller, which I'm not as I'm using the SWD debug headers directly. That'll teach me to read the whole chapter instead of just skimming it and attempting to bruteforce a solution. 

Maybe I need to walk away from this for a while. 

# Step 4 - Come back a day later with a fresh head

I decided to check out the `interface/raspberrypi-swd.cfg` file referenced by the Pico tech docs just in case there were extra settings I needed. The only thing it had that looked like it might make a difference was setting the adapter speed. So I added that to my OpenOCD command with `-c "adapter speed 5000"`. 

After trying again with that change, there was a warning in the OpenOCD output: `The adapter "remote_bitbang" doesn't support configurable speed`

I googled that warning and the first result was a [Github issue regarding the Glasgow, OpenOCD and Espressif boards](https://github.com/espressif/openocd-esp32/issues/135). Folks had submitted their Glasgow and OpenOCD commands, and I noticed some settings that my commands were missing. 

On the Glasgow side, folks were specifying `--port A`. I was using port A but not specifying this to the applet. Even though flashing firmware over the Glasgow worked without this, I figured why not do it anyway just in case. 

Folks were also using a voltage of 2.9V instead of 3.3V like I was. I can't even remember where I got that value from. So lets change that and see what happens. 

And finally, folks added a couple of commands at the end of their OpenOCD command that I didn't have, but which looked familiar to some GDB commands in the Pico tech docs. `-c 'init; reset; halt;'`



# Step 5 - Actually do a debug

Ok lets put all that together :muscle:

```bash
$ glasgow run swd-openocd -V 2.9 --port A --pin-swdio 0 --pin-swclk 1 tcp:localhost:2222
I: g.device.hardware: generating bitstream ID 495f556d820677effe84d423ece8f5a0
I: g.cli: running handler for applet 'swd-openocd'
I: g.applet.interface.swd_openocd: port(s) A voltage set to 2.9 V
I: g.applet.interface.swd_openocd: socket: listening at tcp:localhost:2222
```


Then in a separate terminal run OpenOCD
```bash
$ openocd -c "adapter driver remote_bitbang; transport select swd; remote_bitbang port 2222; adapter speed 5000;" -f target/rp2040.cfg -c "init; reset; halt;"
Open On-Chip Debugger 0.12.0+dev-01618-gbf4be566a (2024-06-17-11:13)
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
adapter speed: 5000 kHz
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
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
[rp2040.core0] halted due to debug-request, current mode: Thread
xPSR: 0x61000000 pc: 0x100012e0 msp: 0x20041fa8
[rp2040.core1] halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x00000178 msp: 0x20041f00
Info : accepting 'gdb' connection on tcp/3333
Info : Found flash device 'win w25q16jv' (ID 0x001540ef)
Info : RP2040 B0 Flash Probe: 2097152 bytes @0x10000000, in 32 sectors

Info : New GDB Connection: 1, Target rp2040.core0, state: halted
Error: GDB missing ack(2) - assumed good
Warn : keep_alive() was not invoked in the 1000 ms timelimit. GDB alive packet not sent! (9534 ms). Workaround: increase "set remotetimeout" in GDB
Info : dropped 'gdb' connection

#
# If your GDB connection gets dropped here, just run your 'target extended-remote localhost:2222' command again and it will reconnect
#

Info : accepting 'gdb' connection on tcp/3333
Info : New GDB Connection: 2, Target rp2040.core0, state: halted
```

This output looks very different from before! Notice its detected info about the Picos CPU, and `Info : Listening on port 3333 for gdb connections` tell us what port to connect our GDB client to.

Now lets run the GDB client and connect it to port 3333 per the OpenOCD output.

<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
> Don't forget to specify the path to the binary that is currently flashed onto the Pico. If you don't have it, this will still work, you just won't get extra options in GDB like symbols. 
{: .prompt-warning }
<!-- markdownlint-restore -->

```bash
$ sudo arm-none-eabi-gdb blink.elf
Password:
GNU gdb (Arm GNU Toolchain 13.2.rel1 (Build arm-13.7)) 13.2.90.20231008-git
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=aarch64-apple-darwin20.6.0 --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://bugs.linaro.org/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from blink.elf...

(gdb) target extended-remote localhost:3333
Remote debugging using localhost:3333
warning: multi-threaded target stopped without sending a thread-id, using first non-exited thread
time_reached (t=...) at /opt/pico-container/pico-sdk/src/rp2_common/hardware_timer/include/hardware/timer.h:116
116	/opt/pico-container/pico-sdk/src/rp2_common/hardware_timer/include/hardware/timer.h: No such file or directory.
(gdb) info reg
r0             0xb76a0             751264
r1             0xf4724             1001252
r2             0x0                 0
r3             0xd0000128          -805306072
r4             0x3d089             249993
r5             0x0                 0
r6             0xf472a             1001258
r7             0x0                 0
r8             0xffffffff          -1
r9             0xffffffff          -1
r10            0xffffffff          -1
r11            0xffffffff          -1
r12            0x20041f60          537141088
sp             0x20041fa8          0x20041fa8
lr             0x10001633          268441139
pc             0x100012e0          0x100012e0 <sleep_until+192>
--Type <RET> for more, q to quit, c to continue without paging--
xpsr           0x61000000          1627389952
msp            0x20041fa8          0x20041fa8
psp            0xfffffffc          0xfffffffc
primask        0x0                 0
basepri        0x0                 0
faultmask      0x0                 0
control        0x0                 0
(gdb) c
Continuing.

```

It worked! We can `ctrl+c` to send an interrupt to the CPU which halts execution and drops us into the GDB shell. Then we can interrogate the registers and see where we are in execution by looking at the pc/program_counter, which says we are offset 192 from the sleep_until routine. Observing the Pico, the flashing LED is no longer blinking, and is just solidly on. When I tell GDB to continue, the LED goes off and then is back to flashing regularly. 

# Conclusion

We have debugged the Pico! Huzzah!

In the next post, I'll be choosing a different COTS (Commercial Off The Shelf) device with an ARM processor that actually has some security mechanisms that we can try to bypass using the techniques we've learned so far. I'm thinking something that has UART that drops to a password protected shell or something like that. Stay tuned!