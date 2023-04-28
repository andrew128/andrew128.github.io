---
layout: post
title: Building a PC
---

Documenting my journey buying the parts for and building a pc.

## Outline
- [Part Selection](#part-selection)
- [Putting together the hardware](#putting-together-the-hardware)
- [Setting up the software](#setting-up-the-software)
- [Reflection](#reflection)

## Part Selection
My thoughts/criteria when building the pc was this:
- be good enough to play the games that I wanted to play
- budget for ~1k knowing that I would probably go over (my final cost was 1.3k USD)

### CPU
I decided to go with an AMD Ryzen 5700X because it seemed like a reasonable relatively cheap entry level CPU.
I had originally picked a 5600 CPU but was recommended this CPU instead because it had 2 extra cores, which could come in handy. 

### CPU Cooler
Because the CPU I chose did not come with a cooler I picked a relatively highly rated one on pc part picker. 

### Motherboard
I went with a b550. 
Through my research it fit all the other parts (and handled their specs) that I wanted (e.g. CPU, RAM, GPU, SSD). 

### RAM
Based on my cursor research online it seemed like 16GB would've been good enough for most current applications but I decided to up my RAM to 32 GB for future proofing. 

### GPU
I wasn't sure whether I wanted to pick a Radeon or a GeForce GPU but it seemed like the Radeon Rx 6800 was a solid pick especially given that it was cheaper than its rough GeForce equivalents (i.e. the 3070). 

### SSD
I went with a 1 TB PCIe 4.0 NVME SSD. 
1 TB seemed like a decent amount of storage to start off with that I could connect extra cheaper forms of storage (e.g. HDDs) to later on as I see fit. 
I decided that it was worth it to splurge the extra money to go from a 3.0 to 4.0 for the speed. 

### Case
I went with a mid-tower because it seemed to be the most common size and I wasn't looking for anything special (e.g. super optimized micro build, lots of extra gpus that would justify a larger case).

### PSU
I went for a platinum rated PSU at the rec of a friend to save power in the long term.
I picked a 750 W PSU because it was a decent chunk above the estimated wattage of my setup by pcpartpicker and because it was recommended for my gpu.

### Fans
I originally thought of going with an Arctic setup for the price but decided to pick Noctua because of the recommendation of a friend. 

## Putting together the hardware
Putting together the hardware wasn't insanely complicated.
It was mostly figuring out what parts plugged into where and then inserting the in the best order into the case.
The part that required the most thought was the fan setup. 

### Fan Setup
Based off of [this](https://www.tomshardware.com/how-to/set-up-pc-case-fans-for-airflow-and-performance) site, it seems like the general idea is to have your fans positioned so that cool air is blowing in from the front bottom of the case and fans are blowing hot air out of the top back.
The case came with an existing fan, I had a CPU cooler, I bought 1 140MM fan and 2 120MM fans. 
Unfortunately the motherboard only had slots for CHA_FAN_1, CHA_FAN_2, CPU_OPT, CPU_FAN. 
After some research I went with the following configuration:
- CPU_FAN - connect the cpu cooler's FAN (want this to be increase/decrease according to the CPU's temperature)
- CHA_FAN_1 - connect the air cooler's pump (so that it's always running)
- CHA_FAN_2 - connect the 140MM fan in the back. 
- CPU_OPT - attached the 120MM fan on top of the CPU so that it would also be controlled by the CPU

## Setting up the software

I had to re insert the SSD because the BIOS wasn't consistently recognizing that it was even connected.

I got Windows 11 Pro from the Microsoft online store.

After downloading the .iso file, I tried to copy it over directly onto a USB disk drive but ran into "Error loading operating system" when I tried to boot up the OS through the bios. 
After doing some research I realized that the usb had to be formatted in a certain way and that the files needed to be properly extracted from the .iso file. 

Following [this](https://www.freecodecamp.org/news/how-make-a-windows-10-usb-using-your-mac-build-a-bootable-iso-from-your-macs-terminal/) website, I was able to create the bootdrive.
Below are the instructions with notes for posterity (basically the same as the website). 
Note that most sites I found containing instructions for how to create a bootable USB drive for Windows on a Mac contained roughly the same instructions and used the same brew library so while it's slightly more involved than I would've thought it seemed like the standard way to go. 

### Identify disk USB drive is mounted on
The following command lists the local disks and shows which disk the USB is mounted to.  
Mine was mounted to `/dev/disk2`. 
```
diskutil list
```

### Format USB drive to work windows
The following command formats the USB drive.
The `eraseDisk` option erases any data that might already be on the usb drive and "writes out a new partitioning schema containing one new empty file system" (according to the man pages).
The new disk is named "WIN11" in this case. 

MS-DOS (i.e. FAT) is chosen as the formatting (used if the size of the disk is 32 GB or less according to [Apple's support pages](https://support.apple.com/lt-lt/guide/disk-utility/dskutl1010/mac)). 
I might have been able to choose ExFAT (used if the size of the disk is over 32 GB) but had already followed the commands given by the site and it seemed to work.
MS-DOS is `diskutil`'s name for FAT32 filesystem. 
Through some cursory online searching, it seems to be widely used due to its flexibility between computing systems (which in this case applies to moving files between Macs and PCs). 
Unfortunately, the downside of FAT32 is that it can't support files larger than 4GB which becomes a problem when copying over a particular file of size ~5 GB in the Windows 11 .iso that is solved later.
```
diskutil eraseDisk MS-DOS "WIN11" GPT /dev/disk2
```

### Mount Windows 11 .iso
Previously I tried to copy paste the .iso file into the usb bootdrive which resulted in an "Error loading operating system error" when I plugged the USB into my computer. 
To understand this I looked more into what iso files even were.

".iso" files are optical disk images.
They reflect exactly what would be stored on an optical disc.  
They are a "sector by sector copy of the data on an optical disc, stored inside a binary file" ([according to wikipedia](https://en.wikipedia.org/wiki/Optical_disc_image)). 
In order to actually use the data stored inside this format, the ISO has to be mounted.
In other words, the operating system would treat the binary as if it were a physical optical disk. 
On the mac that I'm using, this can be done with the `hdiutil` command. 
The site above used mount but I believe attach can be used as well. 
They both tell the computer to connect the disk image to the computer (just as if a CD were connected to the computer externally). 
```
hdiutil mount ~/Downloads/<downloaded windows iso file>.iso
```

### Copy over Windows ISO to USB Drive
As mentioned before, there is a problem with using MS-DOS (FAT32) as the formatting system for our USB. 
We aren't able to fully copy over the entire ISO file in one go because the ISO contains a paricular file (`sources/install.wim`) that is ~5GB which is greater than the 4GB limit of MS-DOS. 
To get around this, we first copy over all the files except for `install.wim`.
I noticed the `rsync` was used instead of `cp`. 
I'm not entirely sure why this was the case but 

Then we use `wimlib`, a library for touching `.wim` files installed through `brew`, a popular mac os package manager.

Note that the destination `install.swm` file has the file extension `.swm` instead of `.wim`.
I believe that this is to let the installer know that the file has been split. 
```
brew install wimlib && rsync -vha --exclude=sources/install.wim /Volumes/CCCOMA_X64FRE_EN-US_DV9/* /Volumes/WIN11 && wimlib-imagex split /Volumes/CCCOMA_X64FRE_EN-US_DV9/sources/install.wim /Volumes/WIN11/sources/install.swm 3800
```

### Formatting the SSD Properly
Since I was running into cryptic error messages trying to boot directly from my usb drive that I made bootable, I decided to copy over the boot data from my usb directly onto my SSD and load from there (and then delete the installation partition afterwards). 
Some things I learned:
- A partition is space crafted on a disk (i.e. partition the disk into separate spaces). There are two partitioning schemes: GPT and MBR. MBR is for older BIOS versions. GPT is for newer UEFI BIOS versions which I have. 
- A volume is a partition that has been formatted into a file system (e.g. NTFS, FAT32)

```
diskpart // start up the Windows disk partitioning tool
list disk // identify my SSD
select disk <disk number> // select my SSD

// the following commands reformat and exit diskpart
clean
convert gpt
create partition primary size=60000
format fs=ntfs quick
// assigns drive letter
assign

// Exit windows disk partitioning tool
exit

// Now copy from usb drive into volume 
// See https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/xcopy for doc on command line flags
xcopy <source volume> <destination volume> /e /h /k
```

NOTE: make sure to remove the usb drive after copying over the data from the usb.
Otherwise the installer will crash again. 


## Reflection
In the future I may try to have better cable management but it's not a priority right now. 
We'll see how well the specific parts hold up for my various use cases. 
I may buy more storage (perhaps in the form of HDDs) since I don't imagine 1TB will hold up in the longer term.
I plan to install some linux distro (maybe Ubuntu for starters?) which given the struggle of installing Windows may be a whole other journey. 