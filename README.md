# UNIX 6th Edition Kernel Source Code

from [www.tom-yam.or.jp/2238/src](http://www.tom-yam.or.jp/2238/src).

# Documents
- [Commentary on the Sixth Edition UNIX Operating System](http://www.lemis.com/grog/Documentation/Lions/)
- [The Unix Tree (Minnie's Home Page)](http://minnie.tuhs.org/cgi-bin/utree.pl)

# Instructions for building and running kernal source Code

## Step 1: Get Unix V6 up and running


There are already some good articles about how to run Unix V6 in Simh in the Internet. Here are the links,

- [Installing and Using Unix V6 in the Open SIMH PDP-11 Emulator](https://decuser.github.io/unix/research-unix/v6/2022/10/19/installing-and-using-research-unix-v6-in-open-simh-pdp-11-emulator.html)

- [Installing UNIX v6 (PDP-11) on SIMH](https://gunkies.org/wiki/Installing_UNIX_v6_(PDP-11)_on_SIMH#Rebuilding_the_kernel)

I'll generally follow the steps in the above articles but add more explanation and background knowledge of each step.

### 1.1 Prepare V6 bootable image
The following scripts work in Linux/MacOS or WSL on Windows.
First of all, get Simh ready in your host computer and get a copy of the system tape.

`curl -O https://www.tuhs.org/Archive/Distributions/Research/Ken_Wellsch_v6/v6.tape.gz`

Get a copy of Wolfgang’s fixes and the enblock and deblock program source code.

`curl -O https://www.tuhs.org/Archive/Distributions/Research/Bug_Fixes/V6enb/v6enb.tar.gz`

Upack
```bash
gunzip v6.tape.gz
tar xvf v6enb.tar.gz
```

Build the enblock and deblock utilities
Warnings are non-fatal and are related to the dialect of C that Wolfgang is using:

```bash
cd v6enb
cc -Wno-implicit-function-declaration enblock.c -o enblock
cc -Wno-implicit-function-declaration deblock.c -o deblock
```

Use enblock to read v6.tape and convert it into dist.tap

./enblock < v6.tape > dist.tap
> Tips: Why need enblock?

*Ken Wellsch’s “V6 tape” refers to the UNIX Version 6 (1975) system tape image.
The original V6 tape was written for PDP-11 machines using a magnetic tape block format.
However, modern emulators like SimH expect tape images in a specific structure:*

- *Fixed 512-byte blocks,*
- *Explicit file boundaries (EOF markers), and*
- *File length indicators.*

*Ken Wellsch’s original V6 tape file is just a raw byte stream, without those structural markers.So we need a small “conversion utility” to repackage (“block”) it into the format SimH expects.*
 
Now we have a tape that Simh can read, The tape is composed of 512-byte blocks:

- Blocks 0 - 100 are the tape bootstrap stuff
- Blocks 101 - 4100 are the RK05 root image
- Blocks 4101 - 8100 are the /usr RK05 image
- Blocks 8101 - 12100 are the /doc RK05 image.

Boot blocks for various types of device are stored at different locations:

- Block 100 is the RK05 boot block
- Block 99 is the RP03 boot block
- Block 98 is the RP04 boot block

The boot block is a piece of code that will read the OS image and load it into memeory at address 0,and then CPU will start execute the code from memory, and then the system boots.

RK05, RP03,RP04 are different disk types. RK05 is around 2.5MB, RP03 is 33MB, RP04 is 67MB.  RK05 is default for V6.

So next step is we will read the data from the tape and copy it to a RK05 disks attached to Simh PDP11.

Create an ini file with the script:

```
set cpu 11/40
set tm0 locked
attach tm0 dist.tap
set rk0 en noautosize
set rk1 en noautosize
set rk2 en noautosize
set rk3 en noautosize
attach rk0 rk0
attach rk1 rk1
attach rk2 rk2
attach rk3 rk3

d cpu 100000 012700
d cpu 100002 172526
d cpu 100004 010040
d cpu 100006 012740
d cpu 100010 060003
d cpu 100012 000777
g 100000
```

The first few lines are Simh instructions, it sets the CPU type and attached the tape file, and 3 disks. 
d cpu is the raw instruction that write machine code at specific address, and g 10000 means starting executing code at the address. What this a few lines of machine code do is:
1. Loads the tape controller’s register address into R0.

2. Writes the command 060003 to the controller, which tells it to:

3. perform a read operation,

4. use device unit 0,

5. transfer one block to memory starting at address 0.

6. The last instruction (BR .-2) loops forever, waiting for the controller to finish the read.

7. Once the first block (the boot block) has been read into memory location 0, the operator typically enters g 0 to execute the loaded boot code.

This code is the PDP-11 bootstrap sequence used in early UNIX V6 installation procedures.
Operators used to enter these words manually on the front panel switches of the PDP-11.

When this code is excuted, the first block of tape is loaded into memory at address 0, which is a second stage disk loader, it has some utitilies such as tmrk which we will use later.

Start the Open SimH PDP-11 emulator with tboot.ini

`./pdp11 tboot.ini`

You will see:

```
PDP-11 simulator V4.0-0 Current        git commit id: 4dfb3508
Disabling XQ
./tboot.ini-1> set cpu 11/40
%SIM-INFO: RQ: RQDX3 controller not valid on a Unibus system, changing to UDA50
%SIM-INFO: RQB: RQDX3 controller not valid on a Unibus system, changing to UDA50
%SIM-INFO: RQC: RQDX3 controller not valid on a Unibus system, changing to UDA50
%SIM-INFO: RQD: RQDX3 controller not valid on a Unibus system, changing to UDA50
./tboot.ini-3> attach tm0 dist.tap
%SIM-INFO: TM0: Tape Image 'dist.tap' scanned as SIMH format

```

Now CPU is in a infinite loop and first block is in memory. Press Ctrl+E to simulate the restart of CPU, and enter `g 0` to let CPU execute the loader program.

```
Simulation stopped, PC: 100012 (BR 100012)
sim> g 0
=
```

The = is the prompt of the loader program. Now we can use tmrk to copy tape into disk.

Use the tmrk utility to copy the disk bootstrap program from tape to the disk block 0:

```
=tmrk 
disk offset
0
tape offset
100
count
1
```

Use tmrk to copy the root filesystem (tape position 101-4100) to disk
```
=tmrk
disk offset
1
tape offset
101
count
3999
=
```

CTRL+E to break the emulation.  Now rk0 is a bootable disk. You can backup it.

