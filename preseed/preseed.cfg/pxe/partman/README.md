# PXE boot Preseeding + partmanによるパーティショニング

- PXEブートタイプの設定ファイル
- パーティショニングの設定に主眼を置く

## 超基本的な設定

以下の条件を仮定する。

- 実装メモリ: 16GB
- HDD: 2TB

実装メモリ16GBなので最大32GBまでswapを伸長する。  
そしてrootパーティション以外の領域はすべて `/srv/extra` に割り当てる。

```
# Destroy All RAID device settings
d-i partman-md/device_remove_md boolean true
# Destroy All LVM device settings
d-i partman-lvm/device_remove_lvm boolean true

d-i partman-auto/disk string /dev/sda
d-i partman-auto/method string regular
d-i partman-auto/expert_recipe string root :: 19000 50 50000 ext4 \
        $primary{ } $bootable{ } method{ format } \
        format{ } use_filesystem{ } filesystem{ ext4 } \
        mountpoint{ / } \
    . \
    32768 90 32768 linux-swap \
        $primary{ } method{ swap } format{ } \
    . \
    100 100 10000000000 ext3 \
        $primary{ } method{ format } format{ } \
        use_filesystem{ } filesystem{ ext4 } \
        mountpoint{ /srv/extra } \
    .
d-i partman-auto/choose_recipe select root
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select Finish partitioning and write changes to disk
d-i partman/confirm boolean true
```

以下、解説。

まず、既にRAIDの設定がしてあると「RAID消すけどいいですか？」って確認画面が出てしまうので「どうぞ削除してください」っていう命令を出す。

```
d-i partman-md/device_remove_md boolean true
```

次、「LVM消すけどいいですか？」って確認画面が出るので「どうぞ削除してください」と答える。

```
d-i partman-lvm/device_remove_lvm boolean true
```

## ソフトウェアRAID1 + LVMを構成する

以下、まだうまくいってない。

```
# Destroy All RAID device settings
d-i partman-md/device_remove_md boolean true
# Destroy All LVM device settings
d-i partman-lvm/device_remove_lvm boolean true

# RAIDアレイに加えるディスクを指定する
d-i     partman-auto/disk string /dev/sda /dev/sdb

# RAIDを構成する
d-i     partman-auto/method string raid

# LVMを構成する
d-i     partman-lvm/confirm boolean true

# partmanエキスパート: boot-rootレシピを選択する
d-i     partman-auto/choose_recipe select boot-root

# LVMボリュームグループの名前
d-i     partman-auto-lvm/new_vg_name string cinder-volumes

# boot-rootレシピの定義
d-i     partman-auto/expert_recipe string        \
           boot-root ::                          \
             1024 30 1024 raid                   \
                $lvmignore{ }                    \
                $primary{ } $bootable{ } method{ raid }       \
                format{ }                        \
             .                                   \
             50000 50 50000 raid                 \
                $lvmignore{ }                    \
                method{ raid }                   \
                format{ }                        \
             .                                   \
             32768 60 32768 raid                 \
                $defaultignore{ }                \
                $lvmok{ }                        \
                method{ raid }                   \
                format{ }                        \
            .                                    
d-i partman-auto-raid/recipe string \
    1 2 0 ext4 /boot                \
          /dev/sda1#/dev/sdb1       \
    .                               \
    1 2 0 ext4 /                    \
          /dev/sda2#/dev/sdb2       \
    .                               \
    1 2 0 lvm -                     \
          /dev/sda3#/dev/sdb3       \
    .                               \
    1 2 0 swap -                    \
          /dev/sda4#/dev/sdb4       \
    .                               
d-i     mdadm/boot_degraded boolean false
d-i     partman-md/confirm boolean true
d-i     partman-partitioning/confirm_write_new_label boolean true
d-i     partman/choose_partition select Finish partitioning and write changes to disk
d-i     partman/confirm boolean true
d-i     partman-md/confirm_nooverwrite  boolean true
d-i     partman/confirm_nooverwrite boolean true
```

PXEブート方式でネットワークインストールしてるから死ぬほど遅い。。。  
partmanの実験に支障が出るのでベーシックなパッケージはDVDに焼いてpreseed.cfgだけHTTP/TFTPで取得するようにする。

```
# isolinux/isolinux.cfg
default install
label install
  menu label ^Install Ubuntu Server
  kernel /install/vmlinuz
  append DEBCONF_DEBUG=5 auto=true locale=en_US.UTF-8 console-setup/charmap=UTF-8 console-setup/layoutcode=us console-setup/ask_detect=false pkgsel/language-pack-patterns=pkgsel/install-language-support=false interface=eth0 hostname=localhost domain=localdomain url=http://gist.github.com/wnoguchi/blahblahblah/raw/preseed.cfg vga=normal initrd=/install/initrd.gz quiet --
label hd
  menu label ^Boot from first hard disk
  localboot 0x80
```

ちなみにpreseed.cfgはgistからとってくるようにしてる。  
変更内容もわかるし、気軽に設定変更できるのですごく便利。  
HTTPSしか無理なのかと思ったけどHTTPでもいける。  
シークレットなgistにしてるのでちょっとだけ安全。  
でも認証かからないからあんまり機密なデータは入れちゃいけません。

![](img/preseedcfg-on-gist.png)

~~インストールはできたけど起動しない。。。  
ブートローダーのインストール周りがいけないのかな。~~

以下のようにGRUBの部分を設定してみる。

```
d-i grub-installer/grub2_instead_of_grub_legacy boolean true 
d-i grub-installer/only_debian boolean true 
d-i grub-installer/bootdev string /dev/sda /dev/sdb
```

ブートするようになってうまく動くようになったと思った。  
ところがぎっちょん。  
12.04をインストールしたはずなのに13.04と認識している。

![](img/2013-09-15_22h15_17.png)

`sudo apt-get -y update` してみた。

![](img/2013-09-15_22h16_27.png)

エラーログをApacheインストールして取得しようとしたらエラーになる。

```
E: Unable to correct problems, you have held broken packages
```

![](img/2013-09-15_22h17_06.png)

パッケージリポジトリの選択をしくってる気がする。  
RAIDの設定周りもなんかおかしい感じがするのでVirtualBoxで適宜スクリーンショットを取りながら進めよう。

以下の設定をfalseからtrueにしたらうまくいった。  
本家がぶっ壊れてる？

```
d-i apt-setup/use_mirror boolean true 
```

apt-getするとjpのミラーが選択されていることを確認する。  
以下のようになるのが正しい。

![](img/2013-09-16_00h36_02.png)

とりあえず動いた設定は以下のようになった。

```
#===========================================================================================
# BOOT SEQUENCE CONFIGURATIONS START
# ENDの設定のところまではDVDメディア、USBメディアに同梱している場合にのみ有効になる設定。
# PXEブートの場合はこのセクションは無視される。
# この場合はpxelinuxのconfigのappendに直接記述する必要がある。
#===========================================================================================
d-i debian-installer/language string en
d-i debian-installer/country string US
d-i debian-installer/locale string en_US.UTF-8
d-i localechooser/supported-locales en_US.UTF-8
d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i console-setup/charmap select UTF-8

# キーボードレイアウトの特性の設定（日本語キーボード）
d-i keyboard-configuration/layoutcode string jp
d-i keyboard-configuration/modelcode jp106

#===========================================================================================
# ネットワークまわりの設定
#-------------------------------------------------------------------------------------------
# 静的IP
#-------------------------------------------------------------------------------------------
# preseed.cfgを外から持ってこようとするとどうしてもいったんDHCP解決しないといけない。
# そして以下の netcfg 項目は一回目は無視されるので d-i preseed/run のところで
# ネットワーク設定をリセットするハックが必要になる。
# そうすると静的IPとして設定を直してくれるようになる。
#
# 詳しくは以下:
# - https://help.ubuntu.com/lts/installation-guide/i386/preseed-contents.html
# - http://debian.2.n7.nabble.com/Bug-688273-Preseed-netcfg-use-autoconfig-and-netcfg-disable-dhcp-doesn-t-work-td1910023.html
#
# 以下の2項目を設定しないと静的IPとして処理されないので重要
d-i netcfg/use_autoconfig boolean false 
d-i netcfg/disable_autoconfig boolean true 

d-i netcfg/choose_interface select eth0 
d-i netcfg/disable_dhcp boolean true 
d-i netcfg/get_nameservers string 8.8.8.8 
d-i netcfg/get_ipaddress string 192.168.1.50 
d-i netcfg/get_netmask string 255.255.255.0 
d-i netcfg/get_gateway string 192.168.1.1 
d-i netcfg/confirm_static boolean true 
d-i netcfg/get_hostname string openstack 
d-i netcfg/get_domain string sv.pg1x.com 
d-i netcfg/wireless_wep string 
#-------------------------------------------------------------------------------------------
# DHCPのとき
#-------------------------------------------------------------------------------------------
#d-i netcfg/choose_interface select eth0 
#d-i netcfg/disable_autoconfig boolean false
#d-i netcfg/get_hostname string openstack 
#d-i netcfg/get_domain string sv.pg1x.com 
#d-i netcfg/wireless_wep string 

# いったんリセット
d-i preseed/run string http://gist.github.com/wnoguchi/7e6fa3b7efb3115eb1df/raw/prescript.sh
#===========================================================================================
# BOOT SEQUENCE CONFIGURATIONS END
#===========================================================================================

# インストーラパッケージをダウンロードするミラーを選択する
#d-i mirror/protocol http
d-i mirror/country string manual
d-i mirror/http/hostname string jp.archive.ubuntu.com
d-i mirror/http/directory string /ubuntu/
d-i mirror/http/proxy string

# インストールするスイートを選択
d-i mirror/suite precise

d-i clock-setup/utc boolean false 
d-i time/zone string Japan 
d-i clock-setup/ntp boolean false 

#===========================================================================================
# PARTMAN PARTITIONING SECTION START
#===========================================================================================
# すべてのRAIDデバイス構成を破棄する
d-i partman-md/device_remove_md boolean true
# すべてのLVMデバイス構成を破棄する
d-i partman-lvm/device_remove_lvm boolean true

d-i partman/confirm_nooverwrite boolean true

# RAIDアレイを構成するディスクを並べる
d-i     partman-auto/disk string /dev/sda /dev/sdb

# RAIDを構成する
d-i     partman-auto/method string raid
d-i     partman-md/confirm boolean true
# 間違い？
d-i     partman-lvm/confirm boolean true
d-i     partman-auto-lvm/guided_size string max

# レシピの選択
d-i     partman-auto/choose_recipe select boot-root

# ボリュームグループの名前を設定
d-i     partman-auto-lvm/new_vg_name string volume-group1

d-i     partman-auto/expert_recipe string        \
           boot-root ::                          \
             1024 30 1024 raid                   \
                $lvmignore{ }                    \
                $primary{ } $bootable{ } method{ raid }       \
             .                                   \
             1000 35 100000000 raid              \
                $lvmignore{ }                    \
                $primary{ } method{ raid }       \
             .                                   \
             50000 50 50000 ext4                 \
                $defaultignore{ }                \
                $lvmok{ }                        \
                lv_name{ root }                  \
                method{ format }                 \
                format{ }                        \
                use_filesystem{ }                \
                filesystem{ ext4 }               \
                mountpoint{ / }                  \
             .                                   \
             16384 60 32768 swap                 \
                $defaultignore{ }                \
                $lvmok{ }                        \
                lv_name{ swap }                  \
                method{ swap }                   \
                format{ }                        \
            .                                    
d-i partman-auto-raid/recipe string \
    1 2 0 ext2 /boot                \
          /dev/sda1#/dev/sdb1       \
    .                               \
    1 2 0 lvm -                     \
          /dev/sda2#/dev/sdb2       \
    .                               
d-i     mdadm/boot_degraded boolean false
d-i     partman-md/confirm boolean true
d-i     partman-partitioning/confirm_write_new_label boolean true
d-i     partman/choose_partition select Finish partitioning and write changes to disk
d-i     partman/confirm boolean true
d-i     partman-md/confirm_nooverwrite  boolean true
d-i     partman/confirm_nooverwrite boolean true
#===========================================================================================
# PARTMAN PARTITIONING SECTION END
#===========================================================================================

d-i base-installer/install-recommends boolean true 
d-i base-installer/kernel/image string linux-generic 

d-i passwd/root-login boolean true 
d-i passwd/make-user boolean false 
d-i passwd/root-password password password 
d-i passwd/root-password-again password password 
d-i passwd/user-fullname string testuser 
d-i passwd/username string testuser 
d-i passwd/user-password password insecure 
d-i passwd/user-password-again password insecure 
d-i user-setup/allow-password-weak boolean true 
d-i user-setup/encrypt-home boolean false 

d-i apt-setup/use_mirror boolean true 

d-i debian-installer/allow_unauthenticated boolean true 
tasksel tasksel/first multiselect none 
d-i pkgsel/include string openssh-server build-essential
d-i pkgsel/upgrade select none 
d-i pkgsel/update-policy select none 
popularity-contest popularity-contest/participate boolean false 
d-i pkgsel/updatedb boolean true 

# GRUBインストーラー
d-i grub-installer/grub2_instead_of_grub_legacy boolean true 
d-i grub-installer/only_debian boolean true 
d-i grub-installer/bootdev string /dev/sda /dev/sdb

# インストールが終了したらサーバー再起動
d-i finish-install/reboot_in_progress note
```

### partman結果

それよりpartmanの結果が見たい。

#### マウント状況

```
root@openstack:~# df -h
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/volume--group1-root   46G  923M   43G   3% /
none                             4.0K     0  4.0K   0% /sys/fs/cgroup
udev                             8.9G  4.0K  8.9G   1% /dev
tmpfs                            1.8G  296K  1.8G   1% /run
none                             5.0M     0  5.0M   0% /run/lock
none                             8.9G     0  8.9G   0% /run/shm
none                             100M     0  100M   0% /run/user
/dev/md0                         915M   30M  837M   4% /boot
```

#### RAID状況

ちゃんとRAIDアレイできてる。

```
root@openstack:~# ls -l /dev/md*
brw-rw---- 1 root disk 9, 0 Sep 16 01:03 /dev/md0
brw-rw---- 1 root disk 9, 1 Sep 16 01:03 /dev/md1

```

#### LVM

```
root@openstack:~# vgdisplay
  --- Volume group ---
  VG Name               volume-group1
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               1.82 TiB
  PE Size               4.00 MiB
  Total PE              476655
  Alloc PE / Size       476655 / 1.82 TiB
  Free  PE / Size       0 / 0
  VG UUID               wryrWz-bf7C-Jwj5-dLew-PUjy-Fej4-jtUiO3

root@openstack:~# lvdisplay
  --- Logical volume ---
  LV Path                /dev/volume-group1/root
  LV Name                root
  VG Name                volume-group1
  LV UUID                1QK02w-XJWj-2d2V-3CPl-ermS-6SrY-MTcUmD
  LV Write Access        read/write
  LV Creation host, time openstack, 2013-09-16 09:51:10 +0900
  LV Status              available
  # open                 1
  LV Size                46.56 GiB
  Current LE             11920
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:0

  --- Logical volume ---
  LV Path                /dev/volume-group1/swap
  LV Name                swap
  VG Name                volume-group1
  LV UUID                4Ws0uH-ai1G-BJKV-F83W-qwiC-1P82-pgW8sG
  LV Write Access        read/write
  LV Creation host, time openstack, 2013-09-16 09:51:10 +0900
  LV Status              available
  # open                 2
  LV Size                1.77 TiB
  Current LE             464735
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:1

```

#### fdiskの結果

```
root@openstack:~# fdisk /dev/sda

The device presents a logical sector size that is smaller than
the physical sector size. Aligning to a physical sector (or optimal
I/O) size boundary is recommended, or performance may be impacted.

Command (m for help): p

Disk /dev/sda: 2000.4 GB, 2000398934016 bytes
255 heads, 63 sectors/track, 243201 cylinders, total 3907029168 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk identifier: 0x0003731a

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2000895      999424   fd  Linux raid autodetect
/dev/sda2         2000896  3907028991  1952514048   fd  Linux raid autodetect
```

## 参考サイト

- [Notes on using expert_recipe in Debian/Ubuntu Preseed Files | Semi-Empirical Shenanigans](http://cptyesterday.wordpress.com/2012/06/17/notes-on-using-expert_recipe-in-debianubuntu-preseed-files/)
- [PartMan - Wikitech](https://wikitech.wikimedia.org/wiki/PartMan)
