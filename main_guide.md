# transferring files between servers and drives
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
