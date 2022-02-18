# debootstrap Debian / Ubuntu / Pop! OS on Native Encrypted zfs through Ansible

This role will allow you to bootstrap a system running from a Live-USB/PXE boot
over SSH using ansible. The author personally uses this for all his Debian-based
distribution installation needs.

## Features
* **Your own partition layout**: You can choose your own partition layout
* Designed with root-on-native-encrypted-**ZFS** in mind
* Boot with **rEFIind** & **zfsbootmanager** (see: https://github.com/zbm-dev/zfsbootmenu)

## Limitations
* I suck at testing

## Supported distributions
* Debian
  * Bullseye
  * Bookworm
* Ubuntu
  * hirsute
  * impish
  * Pop! OS 21.04 & 21.10

Minor modifications will likely make it possible to install newer and 
older versions as well, or even other Debian based distributions. There are
some sanity checks executed, depending on lists in vars/main.yml, so you would
need to add the distribution codename there. Pull requests welcome.

## Host system caveats

This will make a few modifications to the system that is being used as install
medium. This includes installing a few packages, otherwise everything should
be cleaned up afterwards.

# Configuration
## Global variables *(ansible-debootstrap/vars/all.yml)*
* `fqdn`: The fully-qualified domain name of the target  
* `release`: The release codename (**required**, example: *hirsute*)  
* `pop_os`: bool; whether to install the pop-os version of `release`. *Note: Do not combine with a non-ubuntu release*  
* `repos`: Additional repositories to use for installation
* `keys`: GPG keys for additional repositories

  ####   Example repos/keys entry:
  ```
  repos:
    - name: pop-os-apps
      deb: deb http://apt.pop-os.org/staging-proprietary {{ release }} main
    - name: pop-os-release
      deb: deb http://apt.pop-os.org/staging/master {{ release }} main
    - name: pop-os-staging-ubuntu
    - name: jonathonf-zfs
      deb: deb [allow-insecure=yes allow-downgrade-to-insecure=yes] http://ppa.launchpad.net/jonathonf/zfs/ubuntu hirsute main
  keys:
    - name: system76
      key: 63C46DF0140D738961429F4E204DD8AEC33A7AFF
      keyserver: keyserver.ubuntu.com
    - name: jonothonf
      key: 4AB0F789CBA31744CC7DA76A8CF63AD3F06FC659
      keyserver: keyserver.ubuntu.com
  ```
* `wipe`: Set to to string "ABSOLUTELY" if you wish to wipe the disk, this will
remove any disklabels present as well as issue a TRIM/UNMAP for the device.
Useful when you want to overwrite a target. **Please use extreme caution!** 
Electing not to wipe will result in the (mostly untested) deployment of a new 
installation alongside an existing one. **Note:** *in this case the existing 
installation zfs pool must match the new one, as the role will simply create
a new zvol for the root of the new install.*  
* `timezone:` timezone for new installation; *e.g.* `America/New_York"`  
* `locale:` locale for new installation; *e.g.* `"en_US.UTF-8"`  
* `layout`: Dictionary of partitions / devices (**required**, see below)  
  #### Example layout for single NVME device with partitions for EFI system partition & ZFS:
  ```
  layout:
    - device: '/dev/nvme0n1'
      partitions:
        - num: 1
          size: 512M
          type: ef00
          fs: vfat
          mount: /boot/efi
          mountopts: defaults,noauto
        - num: 2
          type: 8200
          fs: zfs_member
  ```
* `zpool_root:` Name of zpool to use; *e.g.,* `rpool`  
* `zfs_root`: zfs dataset for installation root; *e.g.,* `"{{ zpool_root }}/ROOT/{{ release }}"`  
* `zfs_pool`: ZFS pool definition
  #### Example ZFS Pool definition
  ```
  zfs_pool:
    - poolname: "{{ zpool_root }}"
      devices:
        - /dev/nvme0n1p2
      options:
        - ashift=12
        - autotrim=on
      fs_options:
        - encryption=aes-256-gcm
        - acltype=posixacl
        - keyformat=passphrase
        - keylocation=prompt
        - canmount=off
        - mountpoint=none
        - compression=lz4
        - dnodesize=auto
        - relatime=on
        - normalization=formD
        - xattr=sa
      passphrase: !vault |
        $ANSIBLE_VAULT;1.1;AES256
        <<redacted>>
      zbm_timeout: 15
      zbm_cmdline:
        - ro
        - verbose
        - intel_iommu=on
        - iommu=pt
        - rd.driver.blacklist=nouveau
        - rd.module.blacklist=nouveau
  
  ```
* `zfs_fs`: ZFS filesystems (see ZFS section)  
* `install_packages`: List of packages to install  


## Debootstrap user options
`dbstrp_user`:  
  `name`: A name of debootstrap user (**default**: *debootstrap*)  
  `uid`: UID of debootstrap user (**default**: *65533*)  
  `group`: A group name of debootstrap user (**default**: *name of debootstrap user*)  
  `gid`: GID of debootstrap user (**default**: *uid of debootstrap user*)  
  `password`: A hashed password of debootstrap user (**default**: *\**)  
  `non_unique`: Ability to create non unique user (**default**: *yes*)  

## Partition Layout `layout`
Layout is a list of dictionaries, every dictionary representing a target
device. The dictionary contains the device names, as well as another list of
dictionaries representing partitions (as you can see, I like my complex data
structures).

Elements in the device dictionary:  
`device`: Path to the device  
`partitions`: List of partition dictionary  
`skip_grub`: *boolean* set to yes if you want to skip installing grub on this
device (**default** *False*)  

### Partition dictionary
`num`: Ascending number, a limitation here, just make sure it increments with
every element  
`size`: size with qualifier (M for Megabytes etc.), leave empty for the rest
of available space  
`type`: GTID typecode, for example 8200 for linux, see table below for common
type codes  
`fs`: Target filesystem (**optional**, for example ext4, xfs, not needed for
zfs)  
`mount`: Where to mount this device (**optional**, example */boot*)  
`encrypt`: Set to yes if you want encryption  
`target`: Target name for device mapper (**required** when using encryption,
for example *cryptroot*)  
`label` filesystem label to use  
`partlabel` partition label to use, defaults to `label` if defined, otherwise
no label will be set.  

| Type code | Description |
|---|---|
| 8200 | Linux Swap |
| 8300 | Linux Filesystem |
| ef02 | BIOS Boot partition (for grub)|
| fd00 | Linux RAID |
| 8e00 | Linux LVM |

### DM-RAID devices
Within the `md` list you can set up DM RAID devices. List items are
dictionaries supporting the following keys:

`level`: RAID level to be used (one of 0, 1, 4, 5, 6, 10) **required**  
`chunk_size`: RAID chunk size (required for all RAID levels except 1)  
`name`: Device name **required**  
`members`: List of devices to add to RAID  

Encryption and mount options are supported here. Please note that the order is
1. RAID, 2. Encryption, 3. LVM, so it is currently not possible to create a
RAID of two LUKS devices or encrypt an LVM volume. 

### Encryption Options
`passphrase`: Passphrase for encryption, use ansible-vault here please.
(**required** when using encryption)  
`cipher`: Encryption cipher (**default** *aes-xts-plain64*)  
`hash`: Hash type for LUKS (**default** *sha512*)  
`iter-time`: Time to spend on passphrase processing (**default** *5000*)
`key-size`: Encryption key size (**default** *256*, values depend on cipher, for AES *128*, *256*, *512*)  
`luks-type`: LUKS metadata type (**Default** *luks2*)  
`luks-sector-size`: Sector size for LUKS encryption (**default** *512*, possible values: *512*, *4096*)   
`target`: Device name to be used (**required**)  

### LVM configuration
LVM pvs can be created on disks, partitions as well as encrypted devices (use
/dev/mapper/`target` for LUKS). This is a list of dictionaries, dictionary keys
from partition dictionary can be used (except encryption) like `mount`, `fs`
etc. 

`lvm`: Dictionary of lvol options, these are passed to the [lvol noduule](https://docs.ansible.com/ansible/latest/modules/lvol_module.html)
as such, all options available to that module can be used. 



## ZFS configuration
### Pool definition `zfs_pool`
This defines the devices to use for the ZFS pool, you can use devices and
targets defined in `layout`. A list of dictionaries with the following
elements:

`poolname`: name of the ZFS pool (**required**, example *rpool*)  
`devices`: List of devices, you can insert key words like mirror, raidz etc.
like you would when using zpool create. (**required**, example below)  
`options`: List of options for the pool  
`fs_options`: Options for all filesystems in that pool  

### ZFS Filesystems / Datasets to create `zfs_fs`
This looks a lot like the pool definition above, again a list of dictionaries
(they call me the one trick pony). Dictionary definition:

`path`: Path of the dataset, make sure that they are in the correct order,
(**required**, example: *rpool/root*)  
`options`: List of filesystem options, see examples

### Set the root / boot fs for ZFS `zfs_root`
This is used when you want to use ZFS as your root filesystem. Set it to the
dataset which you want to use as root, example: *rpool/ROOT/Ubuntu*

#### ZFS example:
This example is plucked from my own configuration, it will create a lot of
datasets with different options and should give you a good overview over
the possibilities.

```
zfs_pool:
  - poolname: rpool
    devices:
    - /dev/mapper/cryptroot
    options:
    - ashift=12
    fs_options:
    - canmount=off
    - mountpoint=/
    - compression=lz4
    - atime=off
    - normalization=formD

zfs_fs:
  - path: 'rpool/ROOT'
    options:
    - canmount=off
    - mountpoint=none
  - path: 'rpool/ROOT/ubuntu'
    options:
    - mountpoint=/
  - path: 'rpool/home'
    options:
    - setuid=off
  - path: 'rpool/home/root'
    options:
    - mountpoint=/root
  - path: 'rpool/var'
    options:
    - exec=off
    - setuid=off
    - mountpoint=legacy
    mount: /var
  - path: 'rpool/var/cache'
    options:
    - 'com.sun:auto-snapshot=false'
    - mountpoint=legacy
    mount: /var/cache
  - path: 'rpool/var/log'
    optons:
    - mountpoint=legacy
    mount: /var/log
  - path: 'rpool/var/spool'
    optons:
    - mountpoint=legacy
    mount: /var/spool
  - path: 'rpool/var/tmp'
    optons:
    - mountpoint=legacy
    mount: /var/tmp
  - path: 'rpool/var/lib'
    optons:
    - mountpoint=legacy
    mount: /var/lib
  - path: 'rpool/var/lib/dpkg'
    options:
    - exec=on
    - mountpoint=legacy
    mount: /var/lib/dpkg
  - path: 'rpool/srv'

zfs_root: 'rpool/ROOT/ubuntu'
```

## Test playbook for vagrant
The directory meta/tests contains a test playbook, inventories and Vagrantfile for
local testing. The vagrant box by default contains three devices, one for the
source vagrant box and target devices for your install (/dev/sdb, /dev/sdc). To
test your new installation you would have to switch boot devices in the SeaBIOS
boot menu (easily achieved via the VirtualBox GUI). **Currently, only
VirtualBox is supported**.
