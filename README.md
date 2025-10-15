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
cat > tboot.ini << "EOF"
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
EOF
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

CTRL+E to break the emulation.  Now rk0 is a bootable disk. You can backup it in your host system.

### 1.2 Boot Unix V6

This is a lot easier. Prepare another ini file dboot.ini:

```bash
cat > dboot.ini << "EOF"
set cpu 11/40
set tto 7b
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
attach ptr ptr.txt
attach ptp ptp.txt
attach lpt lpt.txt
dep system sr 173030
boot rk0
EOF
```
Note the last line is to boot from the disk we prepared.

Run

`pdp11 dboot.ini`

It will show the boot prompt  @ .

Currently, the V6 binary is located at /rkunix , so input rkunix and press enter. Now, you're logged into Unix V6.

```
@rkunix
mem = 1036
# LS
BIN
DEV
ETC
HPUNIX
LIB
MNT
RKUNIX
RPUNIX
TMP
UNIX
USR

```

Use 

```
STTY -LCASE
```

to set the terminal to use lower case.

## Step 2: Prepare environments for building the source

### 2.1 Mount extra disks and devices

The disk size of rk0 is very small, so we need do the building in another disk.
Simh already simulated a few devices, we need create special file for them.
```
/etc/mknod /dev/rk0 b 0 0
/etc/mknod /dev/rk1 b 0 1
/etc/mknod /dev/rk2 b 0 2
/etc/mknod /dev/rk3 b 0 3
/etc/mknod /dev/mt0 b 3 0
/etc/mknod /dev/tap0 b 4 0
/etc/mknod /dev/rrk0 c 9 0
/etc/mknod /dev/rrk1 c 9 1
/etc/mknod /dev/rrk2 c 9 2
/etc/mknod /dev/rrk3 c 9 3

```
Mount rk3 to /usr/data

```
/etc/mkfs /dev/rk3 4800
mkdir /usr/data
/etc/mount /dev/rk3 /usr/data
```

### 2.2 Copy source code to new disk

The source code in this distributions is in /usr/sys, more specifically, there are 3 sub folders in /usr/sys, the conf folder, which contains bootloaders and configurations; the ken folder, contains main source codes of v6, and dmr, contains drivers for devices. We need copy all these files to /usr/data . The v6 does not support cp *  command yet, we can just copy one by one.

```
chdir /usr/data

cp /usr/sys/buf.h .
cp /usr/sys/conf.h .
cp /usr/sys/file.h .
cp /usr/sys/filsys.h .
cp /usr/sys/ino.h .
cp /usr/sys/inode.h .
cp /usr/sys/param.h .
cp /usr/sys/proc.h .
cp /usr/sys/reg.h .
cp /usr/sys/seg.h .
cp /usr/sys/systm.h .
cp /usr/sys/text.h .
cp /usr/sys/tty.h .
cp /usr/sys/user.h .

mkdir ken

cp /usr/sys/ken/alloc.c ken
cp /usr/sys/ken/clock.c ken
cp /usr/sys/ken/fio.c ken
cp /usr/sys/ken/iget.c ken
cp /usr/sys/ken/main.c ken
cp /usr/sys/ken/malloc.c ken
cp /usr/sys/ken/nami.c ken
cp /usr/sys/ken/pipe.c ken
cp /usr/sys/ken/prf.c ken
cp /usr/sys/ken/rdwri.c ken
cp /usr/sys/ken/sig.c ken
cp /usr/sys/ken/slp.c ken
cp /usr/sys/ken/subr.c ken
cp /usr/sys/ken/sys1.c ken
cp /usr/sys/ken/sys2.c ken
cp /usr/sys/ken/sys3.c ken
cp /usr/sys/ken/sys4.c ken
cp /usr/sys/ken/sysent.c ken
cp /usr/sys/ken/text.c ken
cp /usr/sys/ken/trap.c ken

mkdir dmr
cp /usr/sys/dmr/bio.c dmr
cp /usr/sys/dmr/cat.c dmr
cp /usr/sys/dmr/dc.c dmr
cp /usr/sys/dmr/dh.c dmr
cp /usr/sys/dmr/dhdm.c dmr
cp /usr/sys/dmr/dhfdm.c dmr
cp /usr/sys/dmr/dn.c dmr
cp /usr/sys/dmr/dp.c dmr
cp /usr/sys/dmr/hp.c dmr
cp /usr/sys/dmr/hs.c dmr
cp /usr/sys/dmr/ht.c dmr
cp /usr/sys/dmr/kl.c dmr
cp /usr/sys/dmr/lp.c dmr
cp /usr/sys/dmr/mem.c dmr
cp /usr/sys/dmr/partab.c dmr
cp /usr/sys/dmr/pc.c dmr
cp /usr/sys/dmr/rf.c dmr
cp /usr/sys/dmr/rk.c dmr
cp /usr/sys/dmr/rp.c dmr
cp /usr/sys/dmr/sys.c dmr
cp /usr/sys/dmr/tc.c dmr
cp /usr/sys/dmr/tm.c dmr
cp /usr/sys/dmr/tty.c dmr
cp /usr/sys/dmr/vs.c dmr
cp /usr/sys/dmr/vt.c dmr

mkdir conf
cp /usr/sys/conf/c.c conf
cp /usr/sys/conf/data.s conf
cp /usr/sys/conf/l.s conf
cp /usr/sys/conf/m40.s conf
cp /usr/sys/conf/m45.s conf
cp /usr/sys/conf/mkconf.c conf
cp /usr/sys/conf/sysfix conf
cp /usr/sys/conf/sysfix.c conf

```

### 2.3 Tweak the source code

Ed is the only tool avaliable to edit the file.
```
cd /usr/data/ken
ed main.c
75
i
    printf("Hello")
.
w
q
```
This will print a new line when system boots.

### 2.4 Build the code

Build the makeconf tool.

```
chdir /usr/data/conf
cc mkconf.c
mv a.out mkconf
```
mkconf is what configures the system to support various device types. Run makeconf,tell it about our attached devices - rk05’s, tape reader and tape punch, magtape, DECtape, serial terminals, and line printer:
```
./mkconf
rk
pc
tm
tc
8dc
lp
done
```
It will generate c.c based on hardware configuration.

m40.s is the machine language assist file
c.c is the configuration table containing the major device switches for each device class, block or character.
l.s is the trap vectors for the devices

```
mkdir ../bin

as m40.s
mv a.out m40.o
cc -c c.c
as l.s
mv a.out l.o

mv c.o ../bin
mv l.o ../bin
mv m40.o ../bin
```

Then go to ken and dmr and compile allthe .c files and copy all the *.o file to the bin folder.
```
chdir ../ken
cc -c *.c

mv alloc.o ../bin
mv clock.o ../bin
mv fio.o ../bin
mv iget.o ../bin
mv main.o ../bin
mv malloc.o ../bin
mv nami.o ../bin
mv pipe.o ../bin
mv prf.o ../bin
mv rdwri.o ../bin
mv sig.o ../bin
mv slp.o ../bin
mv subr.o ../bin
mv sys1.o ../bin
mv sys2.o ../bin
mv sys3.o ../bin
mv sys4.o ../bin
mv sysent.o ../bin
mv text.o ../bin
mv trap.o ../bin

chdir ../dmr
cc -c *.c
mv bio.o ../bin
mv cat.o ../bin
mv dc.o ../bin
mv dh.o ../bin
mv dhdm.o ../bin
mv dhfdm.o ../bin
mv dn.o ../bin
mv dp.o ../bin
mv hp.o ../bin
mv hs.o ../bin
mv ht.o ../bin
mv kl.o ../bin
mv lp.o ../bin
mv mem.o ../bin
mv partab.o ../bin
mv pc.o ../bin
mv rf.o ../bin
mv rk.o ../bin
mv rp.o ../bin
mv sys.o ../bin
mv tc.o ../bin
mv tm.o ../bin
mv tty.o ../bin
mv vs.o ../bin
mv vt.o ../bin
```

The final step, link all the o files:

```
ld -X -n \
m40.o l.o \
main.o prf.o slp.o text.o subr.o trap.o\
sys.o sys1.o sys2.o sys3.o sys4.o sysent.o sig.o \
alloc.o bio.o fio.o iget.o nami.o pipe.o rdwri.o \
clock.o malloc.o mem.o partab.o pc.o \
tty.o kl.o rk.o lp.o tm.o \
c.o
```
Note the order of the o files are important, m40.o and l.o must be the first.
The parameter -n will remove the default a.out header and create only pure binaries, making sure the low core is at address 0.

```
mv a.out /unix
```

Reboot Simh, and enter @unix, to let the machine boot from the binaries we have just built. Now you should be able to see the "Hello" message.

