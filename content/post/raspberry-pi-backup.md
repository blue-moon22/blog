---
title: Make an automatic back-up server using an old raspberry pi
tags: ["raspberry pi"]
date: 2017-08-08
---

This is a nice and easy project if you need to back up lots of data automatically from your workspace - in my case genomic data from an HPC cluster I wanted to back-up locally in case of disaster. I had an old raspberry pi lying around and I thought I'd put it to good use. The benefits of using a raspberry pi instead of a laptop or PC is you can just leave your raspberry pi on and let it do its thing. (Note, I do not cover transferring and copying files with version control. Instead, I would recommend [this tutorial](https://opensource.com/life/16/3/turn-your-old-raspberry-pi-automatic-backup-server))

You will need:

* a raspberry pi (including SD card with Raspbian installed)
* a hard drive
* accessories for your raspberry pi, i.e. monitor with HDMI cable, USB keyboard, USB mouse
* an ethernet cable and an ethernet connection


### STEP 1: Setup your pi and hard drive

Switch on your raspberry pi and connect your mouse, keyboard, hard drive, ethernet cable and monitor. (I realised that my raspberry pi had to share 2 USB ports with 3 USB devices - mouse, keyboard and hard drive, which meant I had to sometimes exchange my mouse and keyboard. Fun times. You could get around this by getting a multiport USB device if you have to)

It is likely you may not be able to write to your hard drive when you connect it to your pi. Resolve this in your terminal by unmounting your hard drive and re-mounting to a new directory.

Unmount your drive
```
sudo umount /media/name-of-your-drive
```

Create a new directory (it can be any name, like backup-drive)
```
sudo mkdir /media/backup-drive
```

Re-mount your drive to the new directory (/dev/sda1 should be the id the computer assigns the drive)
```
sudo mount -t ntfs -o uid=pi,gid=pi /dev/sda1 /media/backup-drive/
```

Check the permissions
```
cd /media/backup-drive/
ls -l
```
If it outputs something like "rwxrwxrwx" for the directories in your drive, then it should be writable.


### STEP 2: Create your backup script

First create a new bash script
```
nano backup.sh
```

In your script type the following, press Ctrl X, Y then Enter to save the file
```
#!/bin/bash

rsync -avz <username>@<login-node>:~/path/to/folder/ /media/backup-drive/path/to/folder --ignore-existing --delete
```
The - -ignore-existing option means the file on your hard drive is not replaced again if the same file is still in the cluster. The - -delete option means the file on your hard drive is not removed if the file in the cluster is removed. I would recommend running the command first in the terminal with the - -dry-run option to check the output list of files you plan to transfer over.

Finally you need to allow permission to execute backup.sh. In the terminal:
```
chmod 755 backup.sh
```

### STEP 3: Make the login automatic

To avoid having to manually enter your password for the cluster every time you execute the script, create a public key. (This is less secure though so make sure you're happy to do this.)
```
ssh-keygen -t rsa
```
Enter, and leave the next fields blank to create a blank private key.
```
Enter file in which to save the key:
```
Press enter

```
Enter passphrase (empty for no passphrase):
```
Press enter for no passphrase. (This is less secure but has the benefit of not having to enter a passphrase when executing the backup script)

Then transfer the public key to a .ssh directory in your cluster home directory and enter your password
```
cat ~/.ssh/id_rsa.pub | ssh <username>@<login-node> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```


### STEP 4: Schedule your backup script to run

You need to create a cron schedule:
```
crontab -e
```
Read the instructions and add your schedule at the end of the script. For example if you want to execute the backup script every Monday at 9am then write:
```
0 9 * * 1 /path/to/backup.sh
```

Now your raspberry pi is all set! As long as you leave your raspberry pi connected to the power and the ethernet, it should back up automatically.
