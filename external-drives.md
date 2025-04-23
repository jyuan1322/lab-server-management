# Handling 80TB servers
The mounting process:
1. run `sudo parted -l` or `lsblk` to list detected drives.
```
Model: Mercury Elite Pro Quad B (scsi)
Disk /dev/sdd: 20.0TB 
Sector size (logical/physical): 4096B/4096B 
Partition Table: gpt
Disk Flags: 

Number Start  End   Size  File system Name Flags
 1   24.6kB 2347kB 2322kB
 2   2347kB 9687kB 7340kB
 3   9687kB 20.0TB 20.0TB
```

2. Format the drive according to directions here. For the Quad B drive above, it would be the following commands. (Note the disk name `/dev/sdd`. Do not confuse it with another disk; for example `/dev/sdb` is my 14TB drive which is also connected - don’t reformat it by mistake as it will delete the contents).
```
sudo parted /dev/sdd mklabel gpt
Warning: The existing disk label on /dev/sdd will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? yes
sudo parted -a optimal /dev/sdd mkpart primary 0% 100%
sudo parted /dev/sdd print
sudo mkfs -t xfs /dev/sdd1
```

3. Mount the drive. The directory Mercury1_B is one I created. After creation, you should also chmod 755 that directory so that you don’t need sudo to write to it. Once it’s mounted, you should be able to visit the directory /media/Mercury1_B and read/write to it as usual.
```
sudo mount /dev/sdd1 /media/Mercury1_B
```
