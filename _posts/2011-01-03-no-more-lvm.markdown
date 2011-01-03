---
layout: post
title: Removing LVM
categories:
  - blog
  - linux
  - raid
---

__Removing LVM__
========================================
- - -

So, since I'm sick of LVM, and I have another couple TB of disks to add to my file server, I thought now would be a good
time as ever to remove LVM, and go to straight linear raid over mirrors.


Step 1:  Back up old data:
--------------------------------------
I'll make my new linear raid, over my new disks.  I can then make my new XFS partition
on that, back my old stuff up, swap it, remove LVM, and then add my old mirrors
to my new linear partition.

	root@central:~# mdadm --create /dev/md3 --level=mirror -n 2 /dev/sdc /dev/sdf
	mdadm: Note: this array has metadata at the start and
	    may not be suitable as a boot device.  If you plan to
	    store '/boot' on this device please ensure that
	    your boot-loader understands md/v1.x metadata, or use
	    --metadata=0.90
	Continue creating array? y
	mdadm: Defaulting to version 1.2 metadata
	mdadm: array /dev/md3 started.
	root@central:~# mdadm --create /dev/md100 --force --level=linear -n 1 /dev/md3
	mdadm: Defaulting to version 1.2 metadata
	mdadm: array /dev/md100 started.
	root@central:~# mkfs.xfs /dev/md100
	meta-data=/dev/md100             isize=256    agcount=4, agsize=91571025 blks
		 =                       sectsz=512   attr=2
	data     =                       bsize=4096   blocks=366284100, imaxpct=5
		 =                       sunit=0      swidth=0 blks
	naming   =version 2              bsize=4096   ascii-ci=0
	log      =internal log           bsize=4096   blocks=178849, version=2
		 =                       sectsz=512   sunit=0 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0
	root@central:~# mount /dev/md100 /mnt
	root@central:~# df
	Filesystem           1K-blocks      Used Available Use% Mounted on

	/dev/mapper/vg--md--all-data--xfs
			     943667200 552745476 390921724  59% /data
	/dev/md100           1464421004      4256 1464416748   1% /mnt



	root@central:~# cat /proc/mdstat
	Personalities : [raid1] [linear]
	md100 : active linear md3[0]
	      1465136400 blocks super 1.2 0k rounding
	[... snip ...]
	md3 : active raid1 sdf[1] sdc[0]
	      1465137424 blocks super 1.2 [2/2] [UU]
	      [>....................]  resync =  3.8% (56198208/1465137424) finish=241.5min speed=97207K/sec


A couple hours of rsync running on top of the rebuild, and we're up:

	root@central:/# df -h
	Filesystem            Size  Used Avail Use% Mounted on

	/dev/mapper/vg--md--all-data--xfs
			      900G  528G  373G  59% /data
	/dev/md100            1.4T  537G  860G  39% /mnt

Inspection showed that the extra disk space came from 2 sources, me leving -H and -S off the rsync.  I checked, and by syncing small portions of the tree, with -H and -S, their sizes matched, but really, a couple of gigs isn't work a whole resync.

Step 2:  Remove LVM
======================================================
I've been waiting for this step for a few years know.  Since my data is now on my my new XFS filesystem, 
all I need to do is switch to the new one, and remove LVM:

	root@central:/# mkdir /data.old
	root@central:/# mount --move /data /data.old
	root@central:/# mount --move /mnt /data
	root@central:/# umount /data.old

	root@central:/# invoke-rc.d lvm2 stop
	Shutting down LVM Volume Groups  0 logical volume(s) in volume group "vg-md-all" now active
	.
	root@central:/# dpkg --purge lvm2
	(Reading database ... 46953 files and directories currently installed.)
	Removing lvm2 ...
	Purging configuration files for lvm2 ...
	dpkg: warning: while removing lvm2, directory '/etc/lvm' not empty so not removed.
	Processing triggers for man-db ...
	root@central:/# 

And just to make sure:

	root@central:/# dd if=/dev/zero of=/dev/md0
	^C
	190513+0 records in
	190513+0 records out
	97542656 bytes (98 MB) copied, 10.8053 s, 9.0 MB/s

	root@central:/# dd if=/dev/zero of=/dev/md1
	^C
	273273+0 records in
	273273+0 records out
	139915776 bytes (140 MB) copied, 8.44314 s, 16.6 MB/s

	root@central:/#

Yes, I really don't like LVM.

Step 3: Clean up old mirrors
===========================================

MD set md1 is my pair of small 160GB disks.  I'm going to break thtt up, and use those 2 disks for "expermients".

	root@central:/# mdadm --stop /dev/md0
	mdadm: stopped /dev/md0
	root@central:/# cat /proc/mdstat 
	Personalities : [raid1] [linear] 
	md100 : active linear md3[0]
	      1465136400 blocks super 1.2 0k rounding
	      
	md3 : active raid1 sdf[1] sdc[0]
	      1465137424 blocks super 1.2 [2/2] [UU]
	      
	md1 : active raid1 sdb[0] sde[1]
	      976762496 blocks [2/2] [UU]
	      
	unused devices: <none>
	root@central:/# mdadm --zero-superblock /dev/sda /dev/sdd


Now md1 is my pair of 1TB disks, but that was made old old mdadm, and has old superblock version.  Supposedly superblock version 1.2 is better for filesystems aligning to sectors underneath.  So let's change it, then add it to our linear array:

	root@central:/# mdadm --stop /dev/md1
	mdadm: stopped /dev/md1
	root@central:/# mdadm --zero-superblock /dev/sdb /dev/sde

	root@central:/# mdadm --create /dev/md0 --name central:mirror-1TB --assume-clean --level=mirror -n 2 /dev/sdb /dev/sde
	mdadm: Note: this array has metadata at the start and
	    may not be suitable as a boot device.  If you plan to
	    store '/boot' on this device please ensure that
	    your boot-loader understands md/v1.x metadata, or use
	    --metadata=0.90
	Continue creating array? y
	mdadm: Defaulting to version 1.2 metadata
	mdadm: array /dev/md0 started.

Step 4: Extend our linear set
===========================================================
Because this was the whole point (well, besides getting rid of LVM):

	root@central:/# mdadm /dev/md100  --grow --add /dev/md0
	root@central:/# cat /proc/mdstat 
	Personalities : [raid1] [linear] 
	md0 : active raid1 sde[1] sdb[0]
	      976761424 blocks super 1.2 [2/2] [UU]
	      
	md100 : active linear md0[1] md3[0]
	      2441896800 blocks super 1.2 0k rounding
	      
	md3 : active raid1 sdf[1] sdc[0]
	      1465137424 blocks super 1.2 [2/2] [UU]
	      
	unused devices: <none>
	root@central:/# grep md100 /proc/partitions 
	   9      100 2441896800 md100


And now, of course, we need to extend XFS:

	root@central:/# xfs_growfs /data
	meta-data=/dev/md100             isize=256    agcount=4, agsize=91571025 blks
		 =                       sectsz=512   attr=2
	data     =                       bsize=4096   blocks=366284100, imaxpct=5
		 =                       sunit=0      swidth=0 blks
	naming   =version 2              bsize=4096   ascii-ci=0
	log      =internal               bsize=4096   blocks=178849, version=2
		 =                       sectsz=512   sunit=0 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0
	data blocks changed from 366284100 to 610474200
	root@central:/# df -h
	Filesystem            Size  Used Avail Use% Mounted on
	
	/dev/md100            2.3T  538G  1.8T  24% /data


Step 5: Final configuration
=============================================
And, before you quit, make sure to touchup /etc/mdadm/mdadm.conf:

	mdadm -Es >> /etc/mdadm/mdadm.conf
	vi /etc/mdadm/mdadm.conf

And since I've got a new FS for my /data, I need to remember to update fstab:

	/dev/disk/by-id/md-uuid-0bdd41ba:e402e25a:dc8404a6:5f7db4d5     /data   xfs     defaults        0       0






