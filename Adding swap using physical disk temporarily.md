# Adding swap using physical disk temporarily

Compiling software on "RAM-poor" servers is often a pain, and we need to use a physical disk as a SWAP to add to your system when running out of RAM.

First, make a disk image using ``dd``

```bash
dd if=/dev/zero of=/swapfile bs=64M count=16
```

Change ``count=16`` to a larger number if more swap space is required.
Then, run

```bash
mkswap /swapfile
swapon /swapfile
```

After using or a reboot, remember to delete the swap image to free your disk space.

```bash
swapoff /swapfile
rm -rf /swapfile
```