This tutorial is written in Markdown language, use a Markdown editor to view for best readability.


Digital Ocean's Droplet has been used for demostration purpose on Centos 8

Requirement:
- Create a CentOS VM($10)
- Create 4 block volumes and attach to the VM (2GB each) for RAID 10 purpose

# System Administration


#### Create RAID 10

Update yum repo and install mdadm
```
$ yum update -y && yum install mdadm cryptsetup nfs-utils -y
```

Check available disks, unmount the disk if they are mounted (sda, sdb, sdc, sdd)
```
$ lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
NAME    SIZE FSTYPE  TYPE MOUNTPOINT
sda       2G ext4    disk 
sdb       2G ext4    disk
sdc       2G ext4    disk
sdd       2G ext4    disk
vda      25G         disk
└─vda1   25G xfs     part /
vdb     456K iso9660 disk
```

Create RAID 10 over 4 disks
```
$ mdadm --create --verbose /dev/md0 --level=10 --raid-devices=4 /dev/sda /dev/sdb /dev/sdc /dev/sdd
mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: /dev/sda appears to contain an ext2fs file system
       size=2097152K  mtime=Wed May 13 13:23:32 2020
mdadm: /dev/sdb appears to contain an ext2fs file system
       size=524288K  mtime=Thu Jan  1 00:00:00 1970
mdadm: /dev/sdc appears to contain an ext2fs file system
       size=524288K  mtime=Thu Jan  1 00:00:00 1970
mdadm: /dev/sdd appears to contain an ext2fs file system
       size=524288K  mtime=Thu Jan  1 00:00:00 1970
mdadm: size set to 2094080K
Continue creating array? yes
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

Verify the RAID was successfully created
```
$ cat /proc/mdstat
$ mdadm --query /dev/md0
$ lsblk
```

Save array layout to config file
```
$ mdadm --verbose --detail --scan > /etc/mdadm.conf
```

#### Create LUKS encrypted partition over the RAID array

Configure LUKS on the RAID array
```
$ cryptsetup luksFormat /dev/md0  -v

WARNING!
========
This will overwrite data on /dev/md0 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/md0:
Verify passphrase:
Existing 'ext4' superblock signature on device /dev/md0 will be wiped.
Key slot 0 created.
Command successful.
```

Unencrypt filesytem to make filesystem
```
$ cryptsetup luksOpen /dev/md0 luks_md0
```

Verify the details of the mapped filesystem
```
$ cryptsetup status luks_md0
/dev/mapper/luks_md0 is active.
  type:    LUKS2
  cipher:  aes-xts-plain64
  keysize: 512 bits
  key location: keyring
  device:  /dev/md0
  sector size:  512
  offset:  32768 sectors
  size:    8343552 sectors
  mode:    read/write
```

Make an EXT filesytem and mount install
```
$ mkfs.ext4 /dev/mapper/luks_md0
$ mount /dev/mapper/luks_md0 /raid
```

To automatically mount the filesystem at boot
```
$ echo "LUKS-PASSWD" > /tmp/luks-pass
$ chmod 0400 /tmp/luks-pass
$ cryptsetup luksAddKey /dev/md0 /root/luks-pass
$ echo "luks_md0 /dev/md0 /tmp/luks-pass luks" > /etc/crypttab
$ echo '/dev/mapper/luks_md0 /raid ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```

#### Configure NFS server

Install and enable related services
```
$ yum install -y nfs-utils-lib nfs-server nfs-lock nfs-idmap
$ systemctl enable nfs-utils-lib
$ systemctl enable nfs-server
$ systemctl enable nfs-lock
$ systemctl enable nfs-idmap
```

Share the LUKS protected partition on top of RAID array and make it mount upon system booting
```
$ echo "/raid *(rw,sync,no_root_squash,no_all_squash)" >> /etc/exports && systemctl restart nfs-server
$ exportfs -r
```

Do a reboot to verify everything working as expected
