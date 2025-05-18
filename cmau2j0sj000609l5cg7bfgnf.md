---
title: "os启动! 基于文件系统快照做系统初始化/恢复.md"
datePublished: Sun May 18 2025 19:47:45 GMT+0000 (Coordinated Universal Time)
cuid: cmau2j0sj000609l5cg7bfgnf
slug: os-init-and-backup-and-restore
tags: archlinux, gitops, btrfs

---

os的状态存储在磁盘上。理论上如果我们将一个磁盘的所有bit复制到另一个磁盘，那么我们就可以恢复os的状态。 但是这种程度的抽象，其实等于没有抽象。 对于一个可以正常使用的linux 系统（桌面系统。。），我想要有一种方式，能够对os进行备份和恢复。能够在离线的情况下，将任意一个磁盘变成一个可以boot的，和当前系统状态一样的系统。 这样我们可以测试不同的文件系统，不同的桌面环境，不同的内核，不同的配置。(也不用怕滚挂了)

## 磁盘，uefi,esp，boot,grub，vmlinux and initramfs

在系统启动时:

1. BIOS(UEFI?)会扫描所有磁盘,寻找带有GPT分区表的磁盘
    
2. 在GPT分区中查找ESP(EFI System Partition)分区
    
3. 从ESP分区加载并执行EFI程序 这个EFI程序比如说GRUB引导程序，会负责引导启动Linux系统。在GRUB配置中，我们可以指定要使用的Linux内核镜像(vmlinuz)和根文件系统的位置，从而完成系统的启动过程。 也就是说一个可以正常使用的磁盘，本质上分为三个部分，esp，boot，root. esp: 存储EFI程序,efi程序中引导到grub程序，grub程序中引导到vmlinux和initramfs。 vmlinux,就是我们熟悉的内核，initramfs是内核启动时需要的初始化文件。
    

对一个新的磁盘来讲，首先要将其分区，uefi是能力理解分区的。现在默认的我们用gpt分区。

## 分区

```bash
function os-wipe-and-format-disk() {
  set -x
  set -e
  local disk=$1
  local dname=$2
  sudo lsblk -o NAME,SIZE,FSTYPE,PARTUUID,UUID,SERIAL,TYPE,MOUNTPOINT
  sudo sgdisk --zap-all $disk           # 清空分区信息 # -S gptfdisk
  sudo lvchange -an $dname/root || true # 通知lvm 取消挂载
  sudo lvchange -an $dname/data || true
  sudo lvremove $dname/root || true # 删除lv
  sudo lvremove $dname/data || true
  sudo partprobe $disk

  sudo parted $disk mklabel gpt
  # 创建ESP分区
  sudo parted -a optimal $disk mkpart primary fat32 1MiB 513MiB
  # 声明这是一个ESP分区
  sudo parted $disk set 1 esp on
  # 创建boot分区
  sudo parted -a optimal $disk mkpart primary ext4 513MiB 1537MiB
  # 创建lvm分区
  sudo parted -a optimal $disk mkpart primary 1537MiB 100%
  sudo parted $disk set 3 lvm on
  if [[ $disk == *nvme* ]]; then
    sudo mkfs.vfat ${disk}p1
    sudo mkfs.ext4 -F ${disk}p2
    sudo pvcreate -ff -y ${disk}p3
    sudo vgcreate $dname ${disk}p3
  else
    sudo mkfs.vfat ${disk}1
    sudo mkfs.ext4 -F ${disk}2
    sudo pvcreate -ff -y ${disk}3
    sudo vgcreate $dname ${disk}3
  fi
  local default_size=${DEFAULT_SIZE:-256G}
  sudo lvcreate -y -L $default_size -n root $dname
  sudo lvcreate -y -L $default_size -n data $dname
  sudo mkfs.btrfs -ff /dev/$dname/root
  sudo mkfs.btrfs -ff /dev/$dname/data
  set +x
}
```

我们说快照，但是在最开始的时候，肯定是没有快照，我们只能用网络安装。

```bash
function os-init-arch-via-net() {
  local lvmpart=$1
  local disk=$2
  local dname=$3
  local start=$(date +%s)
  date
  sudo mount --mkdir /dev/mapper/$lvmpart /mnt # 挂载根分区
  sudo mount --mkdir ${disk}2 /mnt/boot        # 挂载 boot 分区
  sudo mount --mkdir ${disk}1 /mnt/boot/efi    # 挂载 efi 分区
   # 这一步最重要的是在/boot上安装内核镜像
  pacstrap -K /mnt base lvm2 linux linux-firmware grub efibootmgr reflector btrfs-progs
  # 初始化esp分区。uefi程序就是这里写进入的。
  # 同时他会向主板更新启动项，在bios页面看到的启动项的名字，就是这里配置的。
  grub-install --target=x86_64-efi --efi-directory=/mnt/boot/efi --boot-directory=/mnt/boot --bootloader-id=$dname
  # 我们用了lvm，所以要保证lvm2在HOOKS中,这样内核启动时，才能识别lvm上的卷
  arch-chroot /mnt bash -c '
    grub-mkconfig -o /boot/grub/grub.cfg
    sed -i "/^HOOKS=/c\HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block lvm2 filesystems fsck)" /etc/mkinitcpio.conf
    mkinitcpio -P
  '
  # 大致上我们可以认为这下面的操作不会动boot分区,属于用户态了
  genfstab -U -o auto,nofail /mnt >>/mnt/etc/fstab
  # base
  arch-chroot /mnt bash -c '
    reflector  --country china --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    hwclock --systohc
    sed -i "/^\s*#\s*en_US.UTF-8/s/^#\s*//" /etc/locale.gen
    sed -i "/^\s*#\s*zh_CN.UTF-8/s/^#\s*//" /etc/locale.gen
    locale-gen
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    pacman -S --noconfirm dhcpcd openssh openvpn smartdns v2ray sudo konsole firefox ntfs-3g
    echo "cong" | passwd --stdin
    systemctl enable dhcpcd
    systemctl enable sshd
'
  # dde
  arch-chroot /mnt bash -c '
    pacman -S --noconfirm  plasma sddm
    systemctl enable sddm
'
  # user apps
  #   arch-chroot /mnt bash -c '
  #     pacman -S --noconfirm kde-applications
  # '
  date
  sudo umount /mnt/boot/efi
  sudo umount /mnt/boot
  sudo umount /mnt
  efibootmgr -v
  local end=$(date +%s)
  echo "show time cost: $((end - start))s"
}
```

注意到grub-install这一步，其实就是在esp分区配置grub的引导，真正的grub可执行文件和模块和其他配置是在/boot/grub下

## 快照和恢复

此时我们可以说os安装完成了。那么我们怎么备份和恢复os的状态呢？

不同于root分区的用户态文件，esp分区和boot分区的文件是很少的。我们可以直接用tar打包。 对于root分区。我们使用btrfs的快照功能。将root分区的快照导出成一个独立的文件xx.btrfs。

```bash
function os-backup() {
  set -x
  local dest=$1
  local ts=$(date +%s-%Y-%m-%d-%H-%M-%S)
  echo $ts
  sudo btrfs subvolume list /
  sudo mkdir -p /.snapshots
  sudo mkdir -p $dest/snapshots/$ts
  sudo btrfs subvolume snapshot -r / /.snapshots/$ts
  sudo btrfs subvolume list /
  local size=$(sudo btrfs filesystem du -s /.snapshots/$ts | tail -n 1 | awk '{print $1}' | sed 's/GiB/G/g')
  echo $size
  sudo btrfs send /.snapshots/$ts | pv -s $size >$dest/snapshots/$ts/root-$ts.btrfs
  sudo tar -czf $dest/snapshots/$ts/boot-$ts.tar.gz /boot
  echo $dest/snapshots/$ts/boot-$ts.tar.gz $dest/snapshots/$ts/root-$ts.btrfs
  set +x
}
```

恢复有两个场景，一种是我们当前已经是一个正常运行的系统。我们要将其恢复到某个快照。 另一种是我们当前是一个空白的磁盘，我们要将其恢复到某个快照。

### case1 当前已经是一个正常运行的系统。我们要将其恢复到某个快照。

restore的过程有繁琐，因为我们不能直接将快照导入到@snap\_root，要用替换的方式

1. 当前的默认卷是@snap\_root
    
2. 为这个snap\_root 新建一个卷@temp,设置temp为默认卷,这样snap\_root就相当于没人用了，可以把它删掉了。
    
3. 从快照恢复的卷默认是只读的，我们为这个快照在创建一个卷@snap\_root,并设置为默认卷。(前面已经删除了@snap\_root，所以这里可以用这个名字了)
    
4. 删除@temp，这样就没有多余的卷了。
    

```bash
function os-restore() {
  set -x
  local boot=$1
  local root=$2
  rm -rf /boot/*
  tar -xzf $boot -C /
  pv $root | btrfs receive /
  local name=$(echo $root | awk -F/ '{print $(NF-1)}')
  echo $name
  mkdir -p /.snapshots
  btrfs subvolume snapshot /@snap_root /@temp
  local tmp_id=$(btrfs subvolume list / | grep @temp | tail -n 1 | awk '{print $2}')
  btrfs subvolume set-default $tmp_id /

  btrfs subvolume delete /@snap_root
  btrfs subvolume snapshot /$name /@snap_root
  btrfs subvolume delete /$name
  local root_id=$(btrfs subvolume list / | grep @snap_root | tail -n 1 | awk '{print $2}')
  btrfs subvolume set-default $root_id /
  btrfs subvlume delete /@temp
  set +x
}
```

### case2 当前是一个空白的磁盘。如何恢复os的状态呢?

我们已经导出了boot.tar和root.btrfs. 那么我们只需要将boot.tar解压到/mnt/boot,将root.btrfs挂载到/mnt/root,然后恢复快照。

```bash
function os-init-arch-via-backup() {
  set -x
  local disk=$1
  local dname=$2
  local lvmpart="$dname-root"
  local boot_tar=$3   # boot 分区的 tar 包
  local root_btrfs=$4 # root 分区的 btrfs 包
  local btrfs_name=$(echo $root_btrfs | awk -F/ '{print $(NF-1)}')
  mount --mkdir /dev/mapper/$lvmpart /mnt # 挂载根分区
  # 必须先把外部传进来的快照挂载新磁盘上，然后将这个新快照所生成的子卷作为根分区才行
  pv $root_btrfs | btrfs receive /mnt/
  btrfs subvolume snapshot /mnt/$btrfs_name /mnt/@snap_root
  btrfs subvolume delete /mnt/$btrfs_name
  local root_id=$(btrfs subvolume list /mnt | grep @snap_root | tail -n 1 | awk '{print $2}')
  btrfs subvolume set-default $root_id /mnt
  umount /mnt
  if [[ "$disk" =~ "nvme" ]]; then
    mount -o subvol=@snap_root --mkdir ${disk}p2 /mnt/boot # 挂载 boot 分区
    mount --mkdir ${disk}p1 /mnt/boot/efi                  # 挂载 efi 分区
  else
    mount --mkdir ${disk}2 /mnt/boot     # 挂载 boot 分区
    mount --mkdir ${disk}1 /mnt/boot/efi # 挂载 efi 分区
  fi
  tar -xvf $boot_tar -C /mnt/ # 解压 boot 分区
  ls -alh /mnt/boot

  # 配置 fstab
  # 重新配置 grub
  cp /etc/resolv.conf /mnt/etc/resolv.conf
  # 要用-r,否则会挂载宿主机的resolv.conf
  arch-chroot -r /mnt/ bash -c "
    set -x
    rm -rf /boot/efi/*
    # 磁盘可能不同了。所以重新生成grub配置
    grub-install --target=x86_64-efi --efi-directory=/boot/efi --boot-directory=/boot --bootloader-id=$dname
    grub-mkconfig -o /boot/grub/grub.cfg
    # 磁盘可能不同了。所以重新生成fstab
    genfstab / > /etc/fstab
    # 磁盘可能不同了。所以重新生成initramfs
    sed -i '/^HOOKS=/c\HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block lvm2 filesystems fsck)' /etc/mkinitcpio.conf
    mkinitcpio -P
  "
  efibootmgr -v
  set +x
}
```

## ext

实际上，我们可以发现制作root.btrfs的过程，其实就是我们系统的安装过程。假设我们有一个运行在btrfs上的github actions runner （或者我们在github的runner上用qemu模拟一个btrfs 卷），那么我们就可以用类似gitops的形式声明（命令式的）一个os的状态了。

## tips

### efi程序中是写了boot的卷id的。。必须要用--boot-directory指定boot分区

```bash
strings /mnt/boot/efi/EFI/GRUB/grubx64.efi |grep root
# search.fs_uuid c1408cf5-0503-4a78-bcb9-f1cff5f67a28 root 
grub-install --target=x86_64-efi --efi-directory=/mnt/boot/efi --boot-directory=/mnt/boot --bootloader-id=k2-embed --removable
```

### grub加载的模块和initramfs中加载的模块是不一样的

比如lvm模块在grub和initramfs中都要加载。因为我们用btrfs的快照，所以initramfs中也要加载lvm模块。

### why btrfs

1. 实际上我只是要一个能导出快照的文件系统,最差的情况下，我们可以用tar来保存fs。
    
2. 我们可以快速、方便地比较两个btrfs快照之间的差异。实际上，每个快照都对应于我们脚本仓库中的一个commit。每次脚本的commit都在上一个快照的基础上执行，那么每个commit就对应一个系统快照。这样，脚本的git diff就是系统快照之间的差异，实现了脚本变更与系统状态变更的对应关系。
    

### vs nix

nix的问题在于，为了实现声明式的系统管理，它选择不遵守Linux文件系统层次标准(FHS)。这导致每个软件都需要重新打包以适配nix的特殊文件系统结构。 这种基于快照的gitops，实际上对root分区里具体是什么没有要求。理论上可以用nix去构造。