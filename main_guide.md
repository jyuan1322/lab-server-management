NOTE: This guide is for educational and internal lab use.

# External drive management
## Mounting an external drive
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

## Mounting an LVM virtual drive
The 80TB Mercury2 drive has been mounted before, so you can simply run
```
sudo mount /dev/vg1/mercury2lv /media/Mercury2LV
```

If mounting a newly configured virtual drive:
(Reference: https://www.cyberciti.biz/faq/linux-mount-an-lvm-volume-partition-command/)

`sudo vim /etc/lvm/lvm.conf` --> comment out any filter that’s there

```
(base) [uname]@ubuntu:~$ sudo vgscan
 Found volume group "vg1" using metadata type lvm2
```

```
(base) [uname]@ubuntu:~$ sudo lvdisplay
 --- Logical volume ---
 LV Path        /dev/vg1/mercury2lv
 LV Name        mercury2lv
 VG Name        vg1
 LV UUID        [UUID]
 LV Write Access    read/write
 LV Creation host, time ubuntu, 2024-06-19 04:32:51 +0000
 LV Status       available
 # open         0
 LV Size        72.76 TiB
 Current LE       19073792
 Segments        4
 Allocation       inherit
 Read ahead sectors   auto
 - currently set to   256
 Block device      253:0
```

```
(base) [uname]@ubuntu:~$ sudo lvs
 LV     VG Attr    LSize Pool Origin Data% Meta% Move Log Cpy%Sync Convert
 mercury2lv vg1 -wi-a----- 72.76t
```

If you encounter this error:
```
(base) [uname]@ubuntu:/media$ sudo mount /dev/vg1/mercury2lv /media/Mercury2LV
mount: /media/Mercury2LV: can't read superblock on /dev/mapper/vg1-mercury2lv.
```
https://unix.stackexchange.com/a/697043
https://unix.stackexchange.com/q/567849
run the following. This deactivates and reactivates the lvm
```
sudo vgchange -a n vg1
sudo vgchange -a y vg1
```
mount command:
```
sudo mount /dev/vg1/mercury2lv /media/Mercury2LV
```

## Formatting a new drive for use with Linux
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

## Formatting a logical volume (LVM)

Resources:
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

(base) [uname]@ubuntu:/$ sudo pvscan
 PV /dev/sdh           lvm2 [18.19 TiB]
 PV /dev/sdi           lvm2 [18.19 TiB]
 PV /dev/sdj           lvm2 [18.19 TiB]
 PV /dev/sdk           lvm2 [18.19 TiB]
 Total: 4 [72.76 TiB] / in use: 0 [0  ] / in no VG: 4 [72.76 TiB]
```

4. Create the volume group
```
sudo vgcreate vg1 /dev/sdh /dev/sdi /dev/sdj /dev/sdk

(base) [uname]@ubuntu:/$ sudo vgdisplay
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
 VG UUID        [UUID]
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

# Creation of new users
```
sudo groupadd donglab
sudo useradd -m [uname]
sudo passwd [uname]
sudo usermod -aG donglab [uname]
# sudo usermod -aG sudo [uname] # adding user as sudoer
# sudo mkhomedir_helper [uname] # if you didn't put -m in useradd
sudo chsh [uname] -s /bin/bash # use bash as shell
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

# Installing R and other software
For all new software, I think it’s wise to adhere to the following principles:
* As much as possible, avoid global installs of software using `sudo apt-get`. We should only install software system-wide if they are well-established, like bedtools or plink. This way, there is less risk of library version conflicts between different users, or errors caused by failed installs (which we have experienced). For most situations, either use an environment manager like conda, or install from source into your local directory.
* As much as possible, avoid mixing different installation managers, like conda and pip for Python. In the past, I have found that pip tends to have more extensive package selections than conda, so I would install as much as I can with conda, then switch to pip. However, this causes a lot of weird, hard-to-resolve errors down the road because conda and pip don’t talk to one another. What you should do instead is initialize a conda environment, then use pip exclusively: this worked for me in complex installs like configuring pytorch to use the Nvidia GPU.

Here is a short tutorial for installing R on tiger. As in the second bullet point above, we will first install base R using conda, but then we will exclusively use `install.packages()` inside R, ensuring all  installed libraries point to a created conda env. DO NOT run `install.packages()`, then use conda to install further packages: this will likely cause problems down the line.

* Install conda in your home directory.
Go to https://repo.anaconda.com/archive/ to select the latest version. We have x86_64/amd64 architecture.
```
cd ~
curl -O https://repo.anaconda.com/archive/Anaconda3-2024.06-1-Linux-x86_64.sh
bash Anaconda3-2024.06-1-Linux-x86_64.sh
```
Say “yes” to initializing conda on startup. Now when you log in, your prompt should look like: `(base) [your uid]@ubuntu:~$`.

* Create an empty conda environment to contain your R environment.
By default, you are in the “base” conda environment, but you should avoid installing there.
```
conda create -n "Rdefault"
conda info -e # view your current environments
conda activate Rdefault
```
Note that in order to use your R environment, you have to activate it when you log in. If that’s annoying to you, you can put `conda activate Rdefault` inside your `~/.bashrc` file. Then upon login it’s automatically activated.

* install R through conda
```
conda install -c conda-forge r-essentials
```

* inside R, check your libPaths() variable. This is the location install.packages ()will install to. Confirm this is inside your newly created conda env.
```
R
> .libPaths()
[1] "/home/[uname]/anaconda3/envs/Rdefault/lib/R/library"
> install.packages("remotes", repos="https://cran.case.edu/")
```
Outside R, confirm that the “remotes” folder appears in `~/anaconda3/envs/Rdefault/lib/R/library`.

* Install Seurat v4 from inside R session:
```
remotes::install_version("SeuratObject", "4.1.4", repos = c("https://satijalab.r-universe.dev", getOption("repos")))
remotes::install_version("Seurat", "4.4.0", repos = c("https://satijalab.r-universe.dev", getOption("repos")))
```

# Resolve missing Ubuntu package dependency errors
Example: I am trying to install rstudio through conda. The conda install succeeds, but when I type rstudio at the command line, I get the following error:
```
Got keys from plugin meta data ("xcb")
...
"/home/[uname]/anaconda3/envs/Rdefault/plugins/platforms/libqxcb.so"
QLibraryPrivate::loadPlugin failed on "/home/[uname]/anaconda3/envs/Rdefault/plugins/platforms/libqxcb.so" : "Cannot load library /home/[uname]/anaconda3/envs/Rdefault/plugins/platforms/libqxcb.so: (libXi.so.6: cannot open shared object file: No such file or directory)"
This application failed to start because it could not find or load the Qt platform plugin "xcb"
in "".
...
```
This type of error either means that the system is missing the required libraries, or your particular user profile isn’t searching in the correct library paths. Here it looks like the problem is `libqxcb.so`

To see the library’s dependencies and their paths, use the `ldd` command:
```
(Rstudio_test) [uname]@ubuntu:~/anaconda3/envs/Rstudio_test/plugins/platforms$ ldd libqxcb.so 
        linux-vdso.so.1 (0x00007ffc6e7e6000)
        libQt5XcbQpa.so.5 => /home/[uname]/anaconda3/envs/Rstudio_test/plugins/platforms/./../../lib/libQt5XcbQpa.so.5 (0x00007f96b170e000)
        libX11-xcb.so.1 => not found
        libXi.so.6 => not found
        libdbus-1.so.3 =>
...
```
Here you can see several “not found” entries - those are the source of the problem. In this case, the libraries are simply missing from the system, so we need to install them. Go to the ubuntu packages website (https://packages.ubuntu.com/) to search for the desired library, making sure to keep the version correct. We are using Ubuntu 22.04 with an x86_64 architecture. That will tell you what to install. Alternately, go to stackoverflow.

After installing, use the `dpkg` command to check what libraries were installed.
```
sudo apt-get install libx11-xcb1
dpkg -L libx11-xcb1
/.
/usr
/usr/lib
/usr/lib/x86_64-linux-gnu
/usr/lib/x86_64-linux-gnu/libX11-xcb.so.1.0.0
/usr/share
/usr/share/doc
/usr/share/doc/libx11-xcb1
/usr/share/doc/libx11-xcb1/copyright
/usr/lib/x86_64-linux-gnu/libX11-xcb.so.1
/usr/share/doc/libx11-xcb1/changelog.Debian.gz
```

# Managing usage limits
Create the file `/etc/systemd/system/user-.slice.d/limits.conf` with the contents
```
[Slice]
TasksMax=500
MemoryMax=500G
CPUQuota=12000%
```
* TasksMax controls the total number of processes and is more appropriate for limiting parallel jobs. This has to be a high number: if too low, there is a risk that you cannot ssh into a new session because that would create another process.
* CPUQuota is reported as percentage of a single core, so CPUQuota=50% means a user can use only 50% of a single core. Setting 12000% means a user can use a maximum of 120 cores full-time, out of a total of 192 on tiger server. So a user can use at most 2/3 of total CPU time.

If making a change to this file, restart the systemd service to apply the changes with the following:
```
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart user-$(id -u).slice
```
To verify your current usage, run `systemctl status user-$(id -u).slice` to see your total number of tasks and memory. CPU time is reported in ms, so to see CPU time as a percentage of total capacity, you can also run `sudo systemd-cgtop`.

# Guide to creating modules
https://researchcomputing.princeton.edu/support/knowledge-base/custom-modules

# Initial server setup
## Enable firewall
Reference: https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu
```
sudo ufw app list
sudo ufw allow OpenSSH
suod ufw enable
sudo ufw status
```

# Monitoring user usage and activity
The `acct` library can tell us login times, cpu usage, and command history for all users. Some examples:

* Total usage time per day, per user. Allegedly this is in hours: I think mine is high because I’m using tmux with multiple panels.
```
[uname]@ubuntu:/mnt/data/referenceGenome$ ac -d -p
...
    [uname]                 1.52
    [uname]                14.87
    user                 7.59
Jul 1 total    23.98
    [uname]                35.24
    [uname]                 0.70
    user                17.08
Today  total    53.02
```
* Per-user usage in cpu minutes, elapsed time in minutes (re), cpu core usage, average I/O operations, cpu storage integral (kilo-core seconds)
```
[uname]@ubuntu:~$ sudo sa -m
                          290    1158.69re       0.07cp         0avio      3270k
root                      173    1157.36re       0.06cp         0avio      4087k
[uname]                     116       1.33re       0.00cp         0avio      2060k
man                         1       0.00re       0.00cp         0avio      2340k
```
Same output but per-command
```
[uname]@ubuntu:~$ sudo sa --print-users
...
root       0.04 cpu   383616k mem      0 io snap                                                 
root       0.00 cpu      723k mem      0 io sh                                                   
root       1.06 cpu    10282k mem      0 io apt                                                  
[uname]      0.00 cpu     2916k mem      0 io sudo            *
[uname]      0.09 cpu     2210k mem      0 io sudo             
[uname]      0.00 cpu      698k mem      0 io ac               
[uname]      0.00 cpu      698k mem      0 io ac               
[uname]      0.00 cpu      698k mem      0 io ac                
...
```
* Command history for all users.
```
[uname]@ubuntu:~$ sudo lastcomm
lastcomm               [uname]    pts/3      0.00 secs Tue Jul  2 17:29
sudo             S     [uname]    pts/3      0.00 secs Tue Jul  2 17:29
sudo              F    [uname]    pts/1      0.00 secs Tue Jul  2 17:29
lastcomm         S     root     pts/1      0.00 secs Tue Jul  2 17:29
lastcomm               [uname]    pts/3      0.00 secs Tue Jul  2 17:29
sudo             S     [uname]    pts/3      0.00 secs Tue Jul  2 17:27
sudo              F    [uname]    pts/1      0.00 secs Tue Jul  2 17:27
lastcomm         S     root     pts/1      0.00 secs Tue Jul  2 17:27
sudo             S     [uname]    pts/3      0.00 secs Tue Jul  2 17:27
sudo              F    [uname]    pts/1      0.00 secs Tue Jul  2 17:27
lastcomm         S     root     pts/1      0.00 secs Tue Jul  2 17:27
lastcomm               [uname]    pts/3      0.00 secs Tue Jul  2 17:27
sudo             S     [uname]    pts/3      0.00 secs Tue Jul  2 17:25
sudo              F    [uname]    pts/1      0.00 secs Tue Jul  2 17:25
sa               S     root     pts/1      0.00 secs Tue Jul  2 17:25
sudo             S     [uname]    pts/3      0.00 secs Tue Jul  2 17:25
```
Unfortunately, it only lists the first word of the command, so you see a lot of “sudo”. To get a more detailed look, you can open each user’s `~/.bash_history` file, which stores the full command line input for the past 500 lines (this is adjustable*). Note bash_history doesn’t automatically save history for a current open session until that session is closed.

* to have unlimited storage of all of your command history, enter the following lines into your ~/.bashrc file
```
HISTSIZE=-1
HISTFILESIZE=-1
```

# Using LSF on erisone/eristwo
Starting an interactive session with 30GB memory
```
bsub -Is -q interactive -R 'rusage[mem=30000]' /bin/bash
```
