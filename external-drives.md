# Handling 80TB servers
The mounting process:
1. run `sudo parted -l` or `lsblk` to list detected drives.
```
$ sudo parted -l
...
Model: Mercury Elite Pro Quad B (scsi)
Disk /dev/sdd: 20.0TB 
Sector size (logical/physical): 4096B/4096B 
Partition Table: gpt
Disk Flags: 

Number Start  End   Size  File system Name Flags
 1   24.6kB 2347kB 2322kB
 2   2347kB 9687kB 7340kB
 3   9687kB 20.0TB 20.0TB
...
```

We have two Mercury drives comprising 4 20TB drives each. Mercury1 is kept as 4 separate drives (A,B,C,D), whereas Mercury2 is combined into one virtual drive using LVM.
```
$ lsblk
NAME             MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sda                8:0    0 140.1T  0 disk /mnt/data
sdb                8:16   0  18.2T  0 disk 
└─sdb1             8:17   0  18.2T  0 part 
sdc                8:32   0  18.2T  0 disk 
└─sdc1             8:33   0  18.2T  0 part 
sdd                8:48   0  18.2T  0 disk 
└─vg1-mercury2lv 253:0    0  72.8T  0 lvm  
sde                8:64   0  18.2T  0 disk 
└─sde1             8:65   0  18.2T  0 part 
sdf                8:80   0  18.2T  0 disk 
└─vg1-mercury2lv 253:0    0  72.8T  0 lvm  
sdg                8:96   0  18.2T  0 disk 
└─sdg1             8:97   0  18.2T  0 part 
sdh                8:112  0  18.2T  0 disk 
└─vg1-mercury2lv 253:0    0  72.8T  0 lvm  
sdi                8:128  0  18.2T  0 disk 
└─vg1-mercury2lv 253:0    0  72.8T  0 lvm  
...
```

1. Mount the drive. The directory `Mercury1_B` should be already created. After creation, you should also chmod 755 that directory so that you don’t need sudo to write to it. Once it’s mounted, you should be able to visit the directory /media/Mercury1_B and read/write to it as usual.
```
sudo mount /dev/sdd1 /media/Mercury1_B
```

# Formatting a new drive for use with Linux
Format the drive according to directions here: https://wiki.docking.org/index.php/Formatting_an_drive_for_use_in_Linux

For the Quad B drive above, it would be the following commands. (Note the disk name `/dev/sdd`. Do not confuse it with another disk; for example `/dev/sdb` is my 14TB drive which is also connected - don’t reformat it by mistake as it will delete the contents).
```
$ sudo parted /dev/sdd mklabel gpt
Warning: The existing disk label on /dev/sdd will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? yes
$ sudo parted -a optimal /dev/sdd mkpart primary 0% 100%
$ sudo parted /dev/sdd print
$ sudo mkfs -t xfs /dev/sdd1
```


