# houndigrade-test-data

This repo is just a collection of data files for testing houndigrade.

## Disk files summary

| filename               | partition | RHEL? | description                                                   |
| ---------------------- | --------- | ----- | ------------------------------------------------------------- |
| disks/centos_release   | 1         | no    | has CentOS release file                                       |
| disks/misc_bad_files   | 1         | no    | has a "bad" non-text release file, yum.conf, and RPM DB       |
| disks/misc_empty       | 1         | no    | empty ext4 filesystem                                         |
| disks/misc_ntfs        | 1         | no    | Windows-like NTFS filesystem                                  |
| disks/rhel_all         | 1         | yes   | has RHEL product cert, release file, yum repo, and signed RPM |
| disks/rhel_cert        | 1         | yes   | has RHEL product cert in ``/etc/pki/product/``                |
| disks/rhel_cert2       | 1         | yes   | has RHEL product cert in ``/etc/pki/product-default/``        |
| disks/rhel_partitioned | 1         | no    | is swap                                                       |
| disks/rhel_partitioned | 2         | yes   | has RHEL release file                                         |
| disks/rhel_partitioned | 3         | no    | empty filesystem                                              |
| disks/rhel_release     | 1         | yes   | has RHEL release file                                         |
| disks/rhel_repo        | 1         | yes   | has RHEL yum repo enabled                                     |
| disks/rhel_rpms        | 1         | yes   | has a Red Hat-signed RPM installed (according to RPM DB)      |

## Disk files HOWTO

These disk files need to be created and manipulated from a Linux system, preferably something that behaves like RHEL.

If your primary system is not running something RHEL-like, consider using a CentOS container:

```
docker run --privileged -v=$(pwd)/disks:/disks:rw -v=/dev:/dev -it centos:7 sh
```

Using this bare-bones Docker image, you may need to install additional packages such as `e2fsprogs`, `ntfs-3g`, and `ntfsprogs`.

### Creating a new file-based disk

The smallest a disk must be to format for ext4 is roughly 70 KiB. The following commands assume you want the smallest possible disk for that format.

Create the file and give it at least one partition:
```
dd bs=1024 count=70 if=/dev/zero of=/disks/my_new_disk
fdisk -C1 /disks/my_new_disk
```

Use a loop device to attach the disk and format it:
```
losetup -D  # detach any existing loop devices
losetup -P /dev/loop0 /disks/my_new_disk
mkfs.ext4 /dev/loop0p1
```

Mount the new partition and put things in it:
```
rm -rf /mnt/my_new_disk; mkdir -p /mnt/my_new_disk
mount -t auto,rw /dev/loop0p1 /mnt/my_new_disk
# write files into /mnt/my_new_disk
```

Unmount and detach the device when done:
```
umount /mnt/my_new_disk
losetup -D
```

### Creating a tiny RPM DB

When creating these minimal filesystems, you may need to create an RPM DB for houndigrade inspection purposes. Unfortunately, the RPM DB directory in a stock RHEL OS install is about 50 MB. If you [find and download](https://access.redhat.com/downloads/content/package-browser) a small Red Hat-signed RPM (e.g. `tree`), however, you can install it locally against a nonstandard path to generate a minimal RPM DB.

For example:

```
rm -rf /tmp/rpmdb; mkdir /tmp/rpmdb
rpm --dbpath=/tmp/rpmdb --nodeps -i tree-1.7.0-15.el8.x86_64.rpm
rpm -qa --dbpath=/tmp/rpmdb/ --qf '%{NAME} %{SIGPGP:pgpsig}\n'
# tree RSA/SHA256, Wed Nov  7 17:20:51 2018, Key ID 199e2f91fd431d51
# "199e2f91fd431d51" is a key houndigrade will match for Red Hat

du -sh /tmp/rpmdb/
# 520K	/tmp/rpmdb/
```
