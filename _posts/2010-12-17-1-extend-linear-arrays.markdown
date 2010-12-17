---
layout: default
title: Extending linear arrays
categories:
  - linux
  - raid
  - mdadm
---


Extending linear raid arrays
============================

So everybody pretty much know my hate for LVM.  It nukes performance, and nukes reliability guarentees (barriers).

Since 2.6.22, linux md raid system has supported "growing" linear arrays.  Adding another disk.

Since I know do all my disk based stuff as raid1 pairs using disks from different vendors, I like being able to toss another TB or 2 into my current file server.

But since I hate LVM, that's always meant a new FS.   No longer.

I start with 2 devices (sdb1, sdb2) for my initial array:

	root@spud:/home/mountie# mdadm --create /dev/md0 --force --level=linear -n 2 /dev/sdb1 /dev/sdb2
	mdadm: Defaulting to version 1.2 metadata
	mdadm: array /dev/md0 started.
	root@spud:/home/mountie# cat /proc/mdstat 
	Personalities : [linear] 
	md0 : active linear sdb2[1] sdb1[0]
	      32127920 blocks super 1.2 0k rounding
	      
	unused devices: <none>
	root@spud:/home/mountie# grep md0 /proc/partitions 
	   9        0   32127920 md0
	root@spud:/home/mountie/test# mkfs.xfs /dev/md0
	meta-data=/dev/md0               isize=256    agcount=4, agsize=2007995 blks
	   =                       sectsz=512   attr=2, projid32bit=0
	data     =                       bsize=4096   blocks=8031980, imaxpct=25
	   =                       sunit=0      swidth=0 blks
	naming   =version 2              bsize=4096   ascii-ci=0
	log      =internal log           bsize=4096   blocks=3921, version=2
	   =                       sectsz=512   sunit=0 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0
	root@spud:/home/mountie/test# mount /dev/md0 /data
	root@spud:/home/mountie/test# df -h /data
	Filesystem            Size  Used Avail Use% Mounted on
	/dev/md0               31G   33M   31G   1% /data

So we have our default file system, spaning 2 devices, and w/ 30GB.  Yes, my devices are small.

But time passes, and I now need more space.  Buy more disks, and add another device:

	root@spud:/home/mountie# mdadm /dev/md0 --grow --add /dev/sdb3
	root@spud:/home/mountie# cat /proc/mdstat 
	Personalities : [linear] 
	md0 : active linear sdb3[2] sdb2[1] sdb1[0]
	      48191896 blocks super 1.2 0k rounding
	      
	unused devices: <none>
	root@spud:/home/mountie# grep md0 /proc/partitions 
	   9        0   48191896 md0

	root@spud:/home/mountie/test# xfs_growfs /data
	meta-data=/dev/md0               isize=256    agcount=4, agsize=2007995 blks
	   =                       sectsz=512   attr=2
	data     =                       bsize=4096   blocks=8031980, imaxpct=25
	   =                       sunit=0      swidth=0 blks
	naming   =version 2              bsize=4096   ascii-ci=0
	log      =internal               bsize=4096   blocks=3921, version=2
	   =                       sectsz=512   sunit=0 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0
	data blocks changed from 8031980 to 12047970
	root@spud:/home/mountie/test# df -h /data
	Filesystem            Size  Used Avail Use% Mounted on
	/dev/md0               46G   33M   46G   1% /data  

There we go.  All online, and another 15GB.  Yes, I added another small device.

Btu more time passes, and I need more space again.  Buy yet another round of disks:

	root@spud:/home/mountie# mdadm /dev/md0 --grow --add /dev/sdb4
	root@spud:/home/mountie# cat /proc/mdstat 
	Personalities : [linear] 
	md0 : active linear sdb4[3] sdb3[2] sdb2[1] sdb1[0]
	      71027270 blocks super 1.2 0k rounding
	      
	unused devices: <none>
	root@spud:/home/mountie# grep md0 /proc/partitions 
	   9        0   71027270 md0

	root@spud:/home/mountie/test# xfs_growfs /data
	meta-data=/dev/md0               isize=256    agcount=6, agsize=2007995 blks
	   =                       sectsz=512   attr=2
	data     =                       bsize=4096   blocks=12047970, imaxpct=25
	   =                       sunit=0      swidth=0 blks
	naming   =version 2              bsize=4096   ascii-ci=0
	log      =internal               bsize=4096   blocks=3921, version=2
	   =                       sectsz=512   sunit=0 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0
	data blocks changed from 12047970 to 17756817
	root@spud:/home/mountie/test# df -h /data
	Filesystem            Size  Used Avail Use% Mounted on
	/dev/md0               68G   33M   68G   1% /data  

And this time, I splurged, and my device was 22GB.  But still, all online!

To apply this to real life:

	   s/sdb/md/

and then my "15GB" devices all become 1-2 TB raid pairs.
