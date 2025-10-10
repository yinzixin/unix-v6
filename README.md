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

I'll generally follow the steps in the above articles but add more explanation of the meaning of each step.

### 1.1 Prepare V6 bootable image
The following scripts work in Linux/MacOS or WSL on Windows.
First of all, get Simh ready in your host computer and get a copy of the system tape.

`curl -O https://www.tuhs.org/Archive/Distributions/Research/Ken_Wellsch_v6/v6.tape.gz`

Get a copy of Wolfgang’s fixes and the enblock and deblock program source code.

`curl -O https://www.tuhs.org/Archive/Distributions/Research/Bug_Fixes/V6enb/v6enb.tar.gz`

Upack

<code>
gunzip v6.tape.gz

tar xvf v6enb.tar.gz
</code>


> Tips: Why need enblock?

*Ken Wellsch’s “V6 tape” refers to the UNIX Version 6 (1975) system tape image.
The original V6 tape was written for PDP-11 machines using a magnetic tape block format.
However, modern emulators like SimH expect tape images in a specific structure:*

- *Fixed 512-byte blocks,*
- *Explicit file boundaries (EOF markers), and*
- *File length indicators.*

*Ken Wellsch’s original V6 tape file is just a raw byte stream, without those structural markers.So we need a small “conversion utility” to repackage (“block”) it into the format SimH expects.*
 
