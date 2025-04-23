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
