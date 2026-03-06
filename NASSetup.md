We required an operating system that could run multiple services - such as NAS, VMs, and other services our instructer would require. We decided to go with Ubuntu as it was recommended to us, along with long-term support and built-in services that will help us in the future. 

Once Ubuntu was installed, the first step was setting up the harddisk that will later become the NAS's storage drive. We first had to format the drive which was done with the following commands:

```
sudo fdisk /dev/sdb
```
n - new partition
w - write

Format it was ext4 (basic filesystem format, works best for nextcloud)
```
sudo mkfs.ext4 /dev/sdb1
```
