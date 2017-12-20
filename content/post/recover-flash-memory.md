+++
title = "How to recover a corrupted flash memory using F3 tools under Linux"
date = "2017-11-20"
lastmod = "2017-11-29"
tags = [ "Flash", "Linux", "F3"]
categories = [
    "Linux"
]
description= "Step by step tutorial for restoring the working part of a corrupted Flash memory using F3 tools under Linux"
+++

**Note:** This tutorial requires Linux. If you want to test your flash under windows check out: [test-and-detect-fake-flash-with-h2testw](https://www.raymond.cc/blog/test-and-detect-fake-or-counterfeit-usb-flash-drives-bought-from-ebay-with-h2testw)

So you have a corrupted flash memory, you copy files into it and when you try to open it later, they don't work. There are number of reasons for this problem, one of them is aging, [flash memories have a lifetime too](https://en.wikipedia.org/wiki/Flash_memory#Write_endurance) and there is a limited number of writes. If this was the cause of the problem, then it is time to retire it. Another reason could be that you have a counterfeit or defected flash, I will show you how to get something out of it in this case, but don't expect much.

Recently someone brought to me a 200GB flash memory having this problem, from the looks of it you can tell for sure that it is a counterfiet. After using F3 tools on it, I found that its actual size is 100MB. F3 was able to recover this 100MB and it was working fine as a flash memory with a partition size equal to 0.005% of it is declared size!

In this tutorial, I will show you how to test your flash memory for problems using F3 command line tools, and if possible, recover its working part.

So what is F3?
====

[F3](http://oss.digirati.com.br/f3/) is an open source command line utility created by Michel Machado. You can find the package in the official Debian repositories and probably in your distrubtion repositories. The package description says:

>F3 (Fight Flash Fraud or Fight Fake Flash) tests the full capacity of a
 flash card (flash drive, flash disk, pen drive).

>F3 writes to the card and then checks if can read it. It will assure you have not been bought a card with a smaller capacity than stated. Note that the main goal of F3 is not to fix your removable media. However, there are resources to mark the invalid areas.

>This package provides these executables: f3write, f3read, f3brew, f3fix
 and f3probe.

Fair enough, lets get started. Install the package or build from [source](https://github.com/AltraMayor/f3):

{{< highlight console >}}
sudo apt-get install f3
{{< /highlight >}}

Test the flash
====

First, we will test the flash thoroughly. We will fill it with data, then check this data for errors. Format the flash, mount it, then pass the mount folder to `f3write` to start filling it:

{{< highlight console >}}
f3write /media/user/mounted_flash
{{< /highlight >}}


Then do the check:

{{< highlight console >}}
f3read /media/user/mounted_flash
{{< /highlight >}}

The time this check takes depends on the declared size of your flash. If the flash passed this test without a problem, then your flash is working fine. If you get data losses, then, unfortunately, it's corrupted. Keep reading to find out its actual size and see if you can get something out of it.

Fix attempt
====

I had an old 1GB flash which had a defected part from the time I got it. Here is the result I had from testing it using f3write and f3read:

{{< highlight console >}}

                  SECTORS      ok/corrupted/changed/overwritten
Validating file 1.h2w ... 1021326/   991976/      0/      2

  Data OK: 498.69 MB (1021326 sectors)
Data LOST: 484.36 MB (991978 sectors)
               Corrupted: 484.36 MB (991976 sectors)
        Slightly changed: 0.00 Byte (0 sectors)
             Overwritten: 1.00 KB (2 sectors)
Average reading speed: 21.21 MB/s

{{< /highlight >}}

The results show clearly that half of my old flash is corrupted. `f3probe` probes flash for counterfeits and identifies its real size. However, this tool is experimental according to the author. It is data destructive, takes a device as an argument. Use it on unmounted flashes.

**Take extreme care** not to pass the wrong device. Double check by running `lsblk`, here is the first lines of my output of `lsblk`:

{{< highlight console >}}

> lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sr0 11:0 1 1024M 0 rom
sdc 8:32 1 986M 0 disk
└─sdc1 8:33 1 985M 0 part
...

{{< /highlight >}}

My device is `/dev/sdc`. I made `f3probe /dev/sdc` and after a few seconds I got the following:

{{< highlight console >}}

f3 probe 6.0
Copyright (C) 2010 Digirati Internet LTDA.
This is free software; see the source for copying conditions.

WARNING: Probing normally takes from a few seconds to 15 minutes, but
         it can take longer. Please be patient.

Probe finished, recovering blocks... Done

Bad news: The device `/dev/sdc' is a counterfeit of type limbo

You can "fix" this device using the following command:
f3fix --last-sec=1015872 /dev/sdc

Device geometry:
                 *Usable* size: 496.03 MB (1015873 blocks)
                Announced size: 986.00 MB (2019328 blocks)
                        Module: 1.00 GB (2^30 Bytes)
        Approximate cache size: 0.00 Byte (0 blocks), need-reset=yes
           Physical block size: 512.00 Byte (2^9 Bytes)

Probe time: 16.10s

{{< /highlight >}}

`f3probe` output says my device is a counterfeit and its real size is 496 MB which is similar to the working part size from the read and write test. Further more, it gave me the command to fix it, your fix command will be different from mine. It fixes the flash by creating a partition from the working part only and discard the fake part. I disconnected the device and inserted it again before running the given command:

{{< highlight console >}}

# sudo ./f3fix --last-sec=1015872 /dev/sdc
F3 fix 6.0
Copyright (C) 2010 Digirati Internet LTDA.
This is free software; see the source for copying conditions.

Drive `/dev/sdc' was successfully fixed

{{< /highlight >}}

That's it, this last command took no time at all. I formatted my flash as usual and it was working fine. To understand what `f3fix` did, here is a screenshot of my flash device in [Gparted](https://gparted.org/) partition editor.

{{< figure src="/images/gparted.png" title="Gparted screenshot of the fixed flash" >}}

The green rectangle is the newly created partition that `f3fix` created. The rest is the fake part which is now left without partitioning. It is clear that this is not a permanent fix, you can recreate partitions anytime, that's what I did for the purpose of this tutorial. But if you left the partition table as `f3fix` did, you should be able to format this partition and use it without a problem.

At last, you need to test the flash partition. Sometimes the first test fails after fixing. From the `f3` docs:

>If you get some sectors corrupted, repeat the f3write/f3read test. Some drives recover from these failures on a second full write cycle. However, if the corrupted sectors persist, the drive is junk because not only is it a fake drive, but its real memory is already failing

I tested mine and this was the result:

{{< highlight console >}}

                  SECTORS      ok/corrupted/changed/overwritten
Validating file 1.h2w ... 1013280/        0/      0/      0

  Data OK: 494.77 MB (1013280 sectors)
Data LOST: 0.00 Byte (0 sectors)
               Corrupted: 0.00 Byte (0 sectors)
        Slightly changed: 0.00 Byte (0 sectors)
             Overwritten: 0.00 Byte (0 sectors)
Average reading speed: 15.32 MB/s

{{< /highlight >}}

That will be all. Hope you found this tutorial useful.
