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

# Formatting a logical volume (LVM)

Resources:\
* https://help.ubuntu.com/community/SettingUpLVM-WithoutACleanInstall
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/logical_volume_manager_administration/physvol_admin

1. In /etc/lvm/lvm.conf, change filter to only detect the drives of interest
```
filter = [ "a|/dev/sd[hijk]|", "r|.*|" ]
```

2. Wipe any existing disk partition (warning this deletes all data on these drives)
```
sudo dd if=/dev/zero of=/dev/sdh bs=512 count=1
sudo dd if=/dev/zero of=/dev/sdi bs=512 count=1
sudo dd if=/dev/zero of=/dev/sdj bs=512 count=1
sudo dd if=/dev/zero of=/dev/sdk bs=512 count=1
```

3. Initialize the physical volumes to be combined
```
sudo pvcreate /dev/sdh /dev/sdi /dev/sdj /dev/sdk

(base) jy1008@ubuntu:/$ sudo pvscan
 PV /dev/sdh           lvm2 [18.19 TiB]
 PV /dev/sdi           lvm2 [18.19 TiB]
 PV /dev/sdj           lvm2 [18.19 TiB]
 PV /dev/sdk           lvm2 [18.19 TiB]
 Total: 4 [72.76 TiB] / in use: 0 [0  ] / in no VG: 4 [72.76 TiB]
```

4. Create the volume group
```
sudo vgcreate vg1 /dev/sdh /dev/sdi /dev/sdj /dev/sdk

(base) jy1008@ubuntu:/$ sudo vgdisplay
 --- Volume group ---
 VG Name        vg1
 System ID       
 Format        lvm2
 Metadata Areas    4
 Metadata Sequence No 1
 VG Access       read/write
 VG Status       resizable
 MAX LV        0
 Cur LV        0
 Open LV        0
 Max PV        0
 Cur PV        4
 Act PV        4
 VG Size        72.76 TiB
 PE Size        4.00 MiB
 Total PE       19073792
 Alloc PE / Size    0 / 0  
 Free PE / Size    19073792 / 72.76 TiB
 VG UUID        QORAPd-a291-F6b8-vhSc-aUFS-utYV-nENq9G
```

5. Create the logical volume with 100% of available space in the volume group
```
lvcreate -l 100%FREE -n mercury2lv vg1

sudo fdisk -l
Disk /dev/mapper/vg1-mercury2lv: 72.78 TiB, 80001282080768 bytes, 19531563008 sectors
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 268431360 bytes
```

6. Format and mount the drive
```
sudo mkfs -t xfs /dev/mapper/vg1-mercury2lv

sudo mount /dev/mapper/vg1-mercury2lv /media/Mercury2LV
```

Running `df -h`, you can see the 80TB logical volume ends up with 73 TB of available space. [Why?](https://www.reddit.com/r/explainlikeimfive/comments/45ugwl/eli5why_is_about_10_of_my_hard_drive_unusable/?rdt=46846)
```
Filesystem                  Size  Used Avail Use% Mounted on
...
/dev/mapper/vg1-mercury2lv   73T  520G   73T   1% /media/Mercury2LV
```


# Transferring files between servers and drives
This command allows for resuming interrupted transfers. no-perms/owner/group is meant to circumvent any issues with different users/groups when transferring between servers.
```
rsync -Aarv --partial --progress --no-perms --no-owner --no-group -e "ssh -oHostKeyAlgorithms=+ssh-rsa" "[username]@[server_ip]:/path/to/file" "[username]@[server_ip]:/path/to/file"
```

# Handling temporary files
Temporary files are written by default to `/tmp` or `/var/tmp`. These need to be periodically cleared, e.g. by using `tmpreaper`, or redirected to a personal directory.
```
export TMPDIR=~/tmp
export TMUX_TMPDIR=~/tmp
```

Delete files that haven't been accessed in 10 days
```
sudo find /tmp -type f -atime +10 -delete
```

# Guide to creating modules
https://researchcomputing.princeton.edu/support/knowledge-base/custom-modules

# Using LSF on erisone/eristwo
Starting an interactive session with 30GB memory
```
bsub -Is -q interactive -R 'rusage[mem=30000]' /bin/bash
```
