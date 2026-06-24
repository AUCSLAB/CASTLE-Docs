# CASTLE Server Infrastructure
This page describes how to set up and maintain services used in the CASTLE Lab.

# OS installation
* Install latest Debian(13)
* 
# RAID

+ install mdadm:

```
sudo apt install mdadm
```
+ Clear the old partition tables from the disks:
```bash
sudo dd if=/dev/zero of=/dev/nvme0n1 bs=4096 count=1000
sudo dd if=/dev/zero of=/dev/nvme1n1 bs=4096 count=1000
sudo dd if=/dev/zero of=/dev/nvme2n1 bs=4096 count=1000
```

+ Then create the RAID 5 array:

```
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1
```

