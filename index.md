# Hardware CTF 5

This challenge will have you find and exploit bugs in a Network Attached Storage (NAS) device. The NAS is a commercial-off-the-shelf device with customised firmware to deliver the CTF-style challenges.

Register your name/handle with InfoSect CTF to track your progress and compare yourself to others [http://ctf.infosectcbr.com.au:8000](http://ctf.infosectcbr.com.au:8000).

Solve each level to get the flag. Once you have the flag, enter it into the CTF to get your points.

## Flag 1 - Physical Access

Gain physical access to the UART on the NAS and establish a serial console.

To do this, we need to remove the case:
1) Unscrew the single philips head (+) screew on the back of the case.
2) Slide the case off.
3) Unscrew each screw located under the two (2) rectangular black foam feet.
4) Unscrew a further 5 screws on the board to detach the green circuit board from the outer case.
5) Look for printed white text on the green PCB board that says "UART_DBG".

Read this section to learn how to connect to UART or look at the previous hardware CTFs on UART [https://infosectcbr.github.io/InfoSect-Hardware-CTF-1/] Level 1.

To connect the NAS to USB serial, connect GND on the NAS to GND on the serial bridge. TXO on the NAS to RX on the serial bridge. And RXO on the NAS to TX on the serial bridge. The NAS header pinout is as follows:

```
 (bottom view of NAS)                   (top view of NAS)
__ _          |                       |
  |_  MCU_ISP |                       |  ___
__|_        o |                       | | . |
            o |                       | | . |
            o |                       | | . | (white connector 1)
            o |                       | | . |
              |                       | '---'
__   UART_DBG |                       |  ___
  |         o | <------- GND -------> | | . |
  |         o | <------- RX0 -------> | | . |
  |         o |                       | | . | (white connector 2)
__|         o | <------- TX0 -------> | | . |
              |                       | '---'
            o |                       | |+|
            o |                       | |+|
              |                       |
  FW RECOVERY |                       |
            o |                       | o
            o |                       | o
            o |                       | o
            o |                       | o
              |                       |
```

Install minicom on a Linux VM:

```
sudo apt-get install minicom
```

Pass through the USB serial to your VM via removable devices in VMWare.

Run minicom:

```
sudo minicom -D /dev/ttyUSB0
```

Configure minicom by typing ctrl-a then z on its own. Use menu option O and select serial port setup. Disable software and hardware flow control using F. The baud rate is 115200. Exit out of the configuration and hit enter a couple of times to see your shell!

This device has a SLOW boot time. You will see log messages as it starts, but it can be up to 2 minutes before you are greeted with a login prompt.

Sometimes in UART you are presented with a root shell without needing to login. Other times you will have to enter a password...

## Flag 2 - U-Boot shell

Without credentials, we cannot log into the Linux terminal on this device. And the updates are encrypted. We need a way to get credentials.

However, as the device was booting - there was a brief pause on the serial console:
```
...
==== QNAP U-Boot Init ====
build: 202201262006
model: ts133
set gpio0_c6 = 1
set gpio0_c7 = 1
set gpio1_a5 = 1
set gpio1_a6 = 1
set gpio2_d6 = 1
get rst_btn = 0
get usb_btn = 0
==========================
Hit key to stop autoboot('CTRL+C'):
```

If we hit `CTRL+C` at this point, we get dropped into a U-Boot prompt. We can type `help` to see a list of commands and `version` to see that this is, indeed, U-Boot:
```
=> help
(list of commands displayed)
=> version
U-Boot 2017.09 (Jan 26 2022 - 20:06:30 +0800)

aarch64-QNAP-linux-gnu-gcc (Linaro GCC 5.3-2016.02) 5.3.1 20160113
GNU ld (GNU Binutils) 2.26.20160125
```

Getting access to U-boot is very powerful - we can read/write the storage of the device, change how Linux boots (Linux command line) and much more.

As we don't have credentials to Linux, lets leverage U-Boot to read the `/etc/shadow` file for us.

We can use the `ext2ls` command to view the contents of the flash storage partitions.
```
=> ext2ls
ext2ls - list files in a directory (default /)

Usage:
ext2ls <interface> <dev[:part]> [directory]
    - list files from 'dev' on 'interface' in a 'directory'
```

Excellent, but what interface/dev/part values should we use? well we can get some hints from `printenv`:
```
=> printenv
...
bootargs=earlycon=uart8250,mmio32,0xfe660000 console=ttyS2,115200 ramoops.mem_address=0x110000 ramoops.mem_size=0xf0000 ramoops.console_size=0x80000 uboot_build_date=202201262006 qnap_model=ts133
...
devnum=0
devtype=mmc
...
scriptaddr=0x00c00000
...
```

Notice how `printenv` has a lot of boot configuration - including `bootargs` for the Linux command line.

Most important though we now have the following for our `ext2ls` command:
 - interface is `mmc` (from `devtype`)
 - dev is `0` (from `devnum`)

But we don't have the parition number - but thats ok, we can try a few different parititons until we find the root one:
```
=> ext2ls mmc 0:0
Failed to mount ext2 filesystem...
** Unrecognized filesystem type **
=> ext2ls mmc 0:1
Failed to mount ext2 filesystem...
** Unrecognized filesystem type **
=> ext2ls mmc 0:2
<DIR>       1024 .
<DIR>       1024 ..
<DIR>      12288 lost+found
<DIR>       1024 boot
<DIR>       1024 etc
=> ext2ls mmc 0:3
<DIR>       1024 .
<DIR>       1024 ..
<DIR>      12288 lost+found
<DIR>       1024 boot
```

Ahh so it looks like partition `2` has an `etc` folder in it, lets see whats inside:
```
=> ext2ls mmc 0:2 /etc
<DIR>       1024 .
<DIR>       1024 ..
              78 shadow
```

Fantastic! We have found the /etc/shadow file - now we just need to read it. To read files in U-Boot, we need to load them into memory first. This can be done by re-using the `scriptaddr` address we had seen in `printenv` earlier:
```
=> ext2load
ext2load - load binary file from a Ext2 filesystem

Usage:
ext2load <interface> [<dev[:part]> [addr [filename [bytes [pos]]]]]
    - load binary file 'filename' from 'dev' on 'interface'
      to address 'addr' from ext2 filesystem.
=> printenv
...
scriptaddr=0x00c00000
...
=> ext2load mmc 0:2 0x00c00000 /etc/shadow
78 bytes read in 44 ms (1000 Bytes/s)
```

Once in memory, we can dump that memory to see the contents of the file:
```
=> md 0x00c00000
00c00000: 00000000 00000000 00000000 00000000    ................  # NOTE: contents of the file will be displayed here :)
00c00010: 00000000 00000000 00000000 00000000    ................
00c00020: 00000000 00000000 00000000 00000000    ................
00c00030: 00000000 00000000 00000000 00000000    ................
00c00040: 00000000 00000000 00000000 00000000    ................
00c00050: 00000000 00000000 00000000 00000000    ................
00c00060: 00000000 00000000 00000000 00000000    ................
00c00070: 00000000 00000000 00000000 00000000    ................
00c00080: 00000000 00000000 00000000 00000000    ................
00c00090: 00000000 00000000 00000000 00000000    ................
00c000a0: 00000000 00000000 00000000 00000000    ................
00c000b0: 00000000 00000000 00000000 00000000    ................
00c000c0: 00000000 00000000 00000000 00000000    ................
00c000d0: 00000000 00000000 00000000 00000000    ................
00c000e0: 00000000 00000000 00000000 00000000    ................
00c000f0: 00000000 00000000 00000000 00000000    ................
```

We can then take those contents and create our own copy of the shadow file which should look somthing like this:
```
admin:<FILL_IN_FROM_MD_COMMAND>:19645:0:99999:7:::
guest:<FILL_IN_FROM_MD_COMMAND>:14233:0:99999:7:::
```

And you might have also noticed the shadow file contained a comment too:)

## Flag 3 - Guest shell

For this section, use the file `/etc/shadow` you extracted using U-Boot.

HINT: The `admin` password is too hard to crack, but `guest` should be possible. Create your a copy of `shadow` file on your machine so that
only the `guest` line is present.

Use tools such as `john` or `hashcat` to crack a password for logging into the device.

Now you have a shell as `guest`, look around the filesystem for an obvious file containing the flag. There may be more than one file, but you have limited permissions.

## Flag 4 - Admin shell

We need to elevate our permissions to admin now (which is the root account).

Have a look in `/bin`, is there any binaries in there that might help us get an admin shell?

We might need to copy that file to a USB stick and reverse engineer it...

With admin, are there any other files that might be readable now?

## For fun - make some noise!

If you have remote access to a device, you can completely control it. As an example, try the following examples:

```
# must be logged in as admin
/sbin/hal_app --se_buzzer enc_id=0,mode=101
```

## For fun - Blinking lights

If you have remote access to a device, you can completely control it. As an example, try the following examples:

```
# must be logged in as admin
/sbin/hal_app --status_led_set enc_id=0,event=0,status=1  # solid green
/sbin/hal_app --status_led_set enc_id=0,event=0,status=1  # blink green
/sbin/hal_app --status_led_set enc_id=0,event=2,status=2  # blink green/red
```

## Flag 5 - Ports Incoming

On the device, run the command `netstat -a`. Use can also port scan using `nmap`.

Are there any interesting looking ports? Maybe try telnet to connect to one...

## Flag 6 - Calculated attack

In taking control of an IoT device, we can often use the tools and software found on the device to help out.

In this challenge, we are going to leverage a `python` (version 2) install found on the device to crack a binary. You will, however, have to first locate the python binary.

The binary we want to crack is called `arithmetic_expression`, and it asks a lot of questions. So lets use Python to do it for us.

We can use python's subprocess library to run and interact with other programs:
```
import subprocess

p = subprocess.Popen('<path_to_binary>', stdin=subprocess.PIPE, stdout=subprocess.PIPE)
while p.returncode is None:   # will exit loop when program terminates
    pass                      # do stuff with binary in here
p.communicate()               # will wait for program to terminate
```

Once we open a process, we can read/write the command line using the following:
```
output_as_bytes = p.stdout.read(number_of_bytes)
p.stdin.write(input_as_bytes)   # buffers inputs
p.stdin.flush()                 # send all the input
```

And, finally, some functions that might help you input/output lines to the binary:
```
def read_line(p):
    line = ''
    while True:
        line += p.stdout.read(1).decode('ascii')
        if line.endswith('\n'):
            break
    return line


def write_line(p, text):
    p.stdin.write(text.encode('ascii'))
    try:
        p.stdin.flush()
    except:
        pass
```

## Remote GDB

When developing exploits for IoT devices, we often need to debug crashes that occur on the physical hardware. The next two challenges are about using gdb to debug on-device binaries.

There are two options for on-device gdb debugging:
1. BYO gdb - put `gdb.aarch64` on a USB stick, mount it on the device, use it to debug.
2. gdbserver - use the `/bin/gdbserver` found on the device to setup a remote debugging instance.

The following instructions are for remote debugging (`gdbserver`).

On the device, setup `gdbserver` to listen on port `9010` (could be any port) and debug a binary.
```
# on device
ip addr # make a note of the device's IP address
gdbserver 0.0.0.0:9010 <path_to_binary_to_be_debugged>
```

On your laptop, you will need to install `gdb-multiarch` as the binaries we are debugging are not the same architecture.
```
sudo apt-get install gdb-multiarch
```

On your laptop, start gdb on a copy of the binary found on the device.
```
# on your laptop
gdb-multiarch <path_to_copy_of_binary_to_be_debugged>
(gdb) target remote <ip_address_of_device>:9010
```

At this point, you can use gdb as normal to debug.
Important point - standard output from binary will be displayed on-device (not in gdb on your laptop).

## Flag 7 - Hidden functionality

There is a binary on the device called `hidden_function` which has a function that will print a flag.

Find out what the function is called (using `strings` on the device might help).

Then use gdb to debug and redirect the program's execution to print the flag.

Some helpful gdb commands:
```
(gdb) b <function_name>       # Set a breakpoint on a function.
(gdb) run                     # Run program
(gdb) continue                # Continue execution until breakpoint, error or exit
(gdb) i r                     # List registers
(gdb) set $<register>=<value> # Don't forget the '$' before the register name. Value can also be a symbol/function name.
```

## Flag 8 - Stacks on

There is a binary on the device called `stack_string` which decrypts a string onto the stack before calling `exit`.

Use gdb to break after the string has been decrypted to the stack and locate the flag in memory.

Some helpful gdb commands:
```
(gdb) x/500s $<register>
```

## GPL Release

As part of the GPL Copyright requirements, the source code modifications made to this device are available on request.
