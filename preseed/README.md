# Ubuntuの自動インストール

刺し身タンポポ作業の自動化。  
CentOSならKickstart使うことになるけど、Ubuntuの場合はPreseedingというツールを使うらしい。

## 環境

- 母艦: Ubuntu 13.04 Desktop
- ISO化するもの: Ubuntu Server 12.04 LTS

## 必要なやつをインストール

```
sudo apt-get -y install syslinux mtools mbr genisoimage dvd+rw-tools
```

## 作業ディレクトリ作成

```
mkdir -p /var/tmp/ubuntu1204/{dvd,dvdr}
cd /var/tmp/ubuntu1204/
```

## マウント

isoイメージはループバックデバイスだから他に何かオプション指定しないといけない気がしたんだけど。

```
sudo mount -t iso9660 /var/tmp/ubuntu-12.04.2-server-amd64.iso /var/tmp/ubuntu1204/dvd
```

## isoイメージからファイルコピー

```
cd dvd
find . ! -type l | cpio -pdum ../dvdr/
1340648 ブロック

```

## 自動インストールを実現するためのisolinux.cfg ファイルの編集

```
cd ../dvdr/
ls -F
EFI/		    boot/	   dists/  install/   md5sum.txt  pool/
README.diskdefines  cdromupgrade*  doc/    isolinux/  pics/	  preseed/
```

`isolinux.cfg` を編集しましょうホイ。  
でも中でいろいろincludeしてるのでたどって行かないと目的のものには辿りつけないので注意。

```
sudo vi isolinux/txt.cfg

default install
label install
  menu label ^Install Ubuntu Server
  kernel /install/vmlinuz
  append  file=/cdrom/preseed/ubuntu-server.seed vga=788 initrd=/install/initrd.gz quiet --

↓

  append  auto=true pkgsel/language-pack-patterns= pkgsel/install-language-support=false file=/cdrom/preseed/preseed.cfg vga=normal initrd=/install/initrd.gz quiet --
```

読み取り専用になってるので!を付加して強制的に書き込む。

## 自動インストールを実現するためのpreseed.cfg ファイルの作成

```
cd preseed
```

```
vi preseed.cfg

d-i debian-installer/locale string en_US 
d-i localechooser/supported-locales en_US.UTF-8, ja_JP.UTF-8 
d-i console-setup/ask_detect boolean false 
d-i console-setup/layoutcode string us 8 
d-i netcfg/choose_interface select auto 
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
d-i mirror/http/mirror select CC.archive.ubuntu.com
d-i clock-setup/utc boolean false 
d-i time/zone string Japan 
d-i clock-setup/ntp boolean false 
d-i partman-auto/init_automatically_partition select biggest_free 
d-i partman-auto/disk string /dev/sda
d-i partman-auto/method string regular 
d-i partman-lvm/device_remove_lvm boolean true 
d-i partman-auto/choose_recipe select atomic 
d-i partman/default_filesystem string ext4 
d-i partman-partitioning/confirm_write_new_label boolean true 
d-i partman/choose_partition select finish 
d-i partman/confirm boolean true 
d-i partman/confirm_nooverwrite boolean true 
d-i partman-partitioning/confirm_write_new_label boolean true 
d-i partman/choose_partition select finish 
d-i partman/confirm boolean true 
d-i partman/confirm_nooverwrite boolean true 
d-i partman/mount_style select traditional
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
d-i apt-setup/use_mirror boolean false 
d-i debian-installer/allow_unauthenticated boolean true 
tasksel tasksel/first multiselect none 
d-i pkgsel/include string openssh-server build-essential
d-i pkgsel/upgrade select none 
d-i pkgsel/update-policy select none 
popularity-contest popularity-contest/participate boolean false 
d-i pkgsel/updatedb boolean true 
d-i grub-installer/grub2_instead_of_grub_legacy boolean false 
d-i grub-installer/only_debian boolean true 
d-i grub-installer/bootdev string (hd0,0) 
d-i finish-install/reboot_in_progress note
```

ちなみにサンプルで入ってるseedファイルは以下の様な感じになってた。

```
vi ubuntu-server-minimal.seed

# Suggest LVM by default.
d-i     partman-auto/init_automatically_partition       string some_device_lvm
d-i     partman-auto/init_automatically_partition       seen false
# Only install basic language packs. Let tasksel ask about tasks.
d-i     pkgsel/language-pack-patterns   string
# No language support packages.
d-i     pkgsel/install-language-support boolean false
# Only ask the UTC question if there are other operating systems installed.
d-i     clock-setup/utc-auto    boolean true
# Verbose output and no boot splash screen.
d-i     debian-installer/quiet  boolean false
d-i     debian-installer/splash boolean false
# Install the debconf oem-config frontend (if in OEM mode).
d-i     oem-config-udeb/frontend        string debconf
# Wait for two seconds in grub
d-i     grub-installer/timeout  string 2
# Add the network and tasks oem-config steps by default.
oem-config      oem-config/steps        multiselect language, timezone, keyboard, user, network, tasks
d-i     base-installer/kernel/altmeta   string lts-quantal

```

```
vi ubuntu-server.seed

# Suggest LVM by default.
d-i     partman-auto/init_automatically_partition       string some_device_lvm
d-i     partman-auto/init_automatically_partition       seen false
# Install the Ubuntu Server seed.
tasksel tasksel/force-tasks     string server
# Only install basic language packs. Let tasksel ask about tasks.
d-i     pkgsel/language-pack-patterns   string
# No language support packages.
d-i     pkgsel/install-language-support boolean false
# Only ask the UTC question if there are other operating systems installed.
d-i     clock-setup/utc-auto    boolean true
# Verbose output and no boot splash screen.
d-i     debian-installer/quiet  boolean false
d-i     debian-installer/splash boolean false
# Install the debconf oem-config frontend (if in OEM mode).
d-i     oem-config-udeb/frontend        string debconf
# Wait for two seconds in grub
d-i     grub-installer/timeout  string 2
# Add the network and tasks oem-config steps by default.
oem-config      oem-config/steps        multiselect language, timezone, keyboard, user, network, tasks
d-i     base-installer/kernel/altmeta   string lts-quantal

```

## カスタムisoイメージを作成する

```
cd /var/tmp/ubuntu1204
sudo genisoimage -N -J -R -D -V "PRESEED" -o ubuntu-12.04-server-amd64-preseed.iso -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table dvdr

(snip)

 93.17% done, estimate finish Mon Aug 26 23:25:33 2013
 94.65% done, estimate finish Mon Aug 26 23:25:33 2013
 96.13% done, estimate finish Mon Aug 26 23:25:33 2013
 97.61% done, estimate finish Mon Aug 26 23:25:34 2013
 99.08% done, estimate finish Mon Aug 26 23:25:34 2013
Total translation table size: 2048
Total rockridge attributes bytes: 371263
Total directory bytes: 1908736
Path table size(bytes): 12978
Max brk space used 369000
338099 extents written (660 MB)
```

## 実行してみる

いきなり物理マシンにやるのはけっこうしんどいので仮想マシンでやってみる。  
なんか大体は自動化できてるけど、言語周りの選択が促される。なんで。

参考サイトのcfgを参考に先頭を合わせてみた。

```
d-i debian-installer/language string en
d-i debian-installer/country string US
d-i debian-installer/locale string en_US.UTF-8
d-i localechooser/supported-locales en_US.UTF-8d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i console-setup/charmap select UTF-8
d-i keyboard-configuration/layoutcode string us
d-i netcfg/choose_interface select auto 
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
d-i mirror/http/mirror select CC.archive.ubuntu.com
d-i clock-setup/utc boolean false 
d-i time/zone string Japan 
d-i clock-setup/ntp boolean false 
d-i partman-auto/init_automatically_partition select biggest_free 
d-i partman-auto/disk string /dev/sda
d-i partman-auto/method string regular 
d-i partman-lvm/device_remove_lvm boolean true 
d-i partman-auto/choose_recipe select atomic 
d-i partman/default_filesystem string ext4 
d-i partman-partitioning/confirm_write_new_label boolean true 
d-i partman/choose_partition select finish 
d-i partman/confirm boolean true 
d-i partman/confirm_nooverwrite boolean true 
d-i partman-partitioning/confirm_write_new_label boolean true 
d-i partman/choose_partition select finish 
d-i partman/confirm boolean true 
d-i partman/confirm_nooverwrite boolean true 
d-i partman/mount_style select traditional
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
d-i apt-setup/use_mirror boolean false 
d-i debian-installer/allow_unauthenticated boolean true 
tasksel tasksel/first multiselect none 
d-i pkgsel/include string openssh-server build-essential
d-i pkgsel/upgrade select none 
d-i pkgsel/update-policy select none 
popularity-contest popularity-contest/participate boolean false 
d-i pkgsel/updatedb boolean true 
d-i grub-installer/grub2_instead_of_grub_legacy boolean false 
d-i grub-installer/only_debian boolean true 
d-i grub-installer/bootdev string (hd0,0) 
d-i finish-install/reboot_in_progress note

```

今度はうまくいったっぽい。
ただ立ち上げると一発目で言語を聞いてくるのがちょっといらない。
それとなんかstaticにしてるのにDHCP効いてるし。。。

locale=en_US.UTF-8 console-setup/charmap=UTF-8 console-setup/layoutcode=us console-setup/ask_detect=false 

```
sudo vi dvdr/isolinux/txt.cfg

  append  auto=true pkgsel/language-pack-patterns= pkgsel/install-language-support=false file=/cdrom/preseed/preseed.cfg vga=normal initrd=/install/initrd.gz quiet --

↓

  append  auto=true locale=en_US.UTF-8 console-setup/charmap=UTF-8 console-setup/layoutcode=us console-setup/ask_detect=false pkgsel/language-pack-patterns=pkgsel/install-language-support=false file=/cdrom/preseed/preseed.cfg vga=normal initrd=/install/initrd.gz quiet --
```

変わらず。  
参考サイト見たら `isolinux.cfg` を直接書き換えてるっぽい。  
メニューが表示されてうるさい。

```
isolinux.cfg

# D-I config version 2.0
include menu.cfg
default vesamenu.c32
prompt 0
timeout 0
ui gfxboot bootlogo
```

上の内容を抹消して以下の内容に全置換。

```
isolinux.cfg

default install
label install
  menu label ^Install Ubuntu Server
  kernel /install/vmlinuz
  append  auto=true locale=en_US.UTF-8 console-setup/charmap=UTF-8 console-setup/layoutcode=us console-setup/ask_detect=false pkgsel/language-pack-patterns=pkgsel/install-language-support=false file=/cdrom/preseed/preseed.cfg vga=normal initrd=/install/initrd.gz quiet --
label hd
  menu label ^Boot from first hard disk
  localboot 0x80
```

うまくいった。

## PXE+Preseedでやってみた

192.168.1.10がPXEブートサーバー兼TFTPサーバーとなっている。  
pxelinuxでブート時にTFTPサーバーからpreseedファイルを取得してきている。  
パスに注意。  
PXEブートサーバーについては

[doc/Network/pxe/ubuntu-pxe.md at master · wnoguchi/doc](https://github.com/wnoguchi/doc/blob/master/Network/pxe/ubuntu-pxe.md)

を参照。

### pxelinuxのconfig

```
# /var/lib/tftpboot/ubuntu-installer/amd64/pxelinux.cfg/default
default install
label install
  menu label ^Install Ubuntu Server
  kernel ubuntu-installer/amd64/linux
  append DEBCONF_DEBUG=5 auto=true locale=en_US.UTF-8 console-setup/charmap=UTF-8 console-setup/layoutcode=us console-setup/ask_detect=false pkgsel/language-pack-patterns=pkgsel/install-language-support=false interface=eth0 hostname=localhost domain=localdomain url=http://192.168.1.31/preseed.cfg initrd=ubuntu-installer/amd64/initrd.gz vga=normal quiet --
```

### preseed.cfg

~~うごかない、うごかない・・・。~~  
動かないというのは語弊がある。  
RX100S7のPXEとIPMI管理ポート、PXE+DHCPサーバーのセグメントと、eth0の通常のNICには  
NTTのブロードバンドルーターのDHCPの下に置いてDHCP2本立てにしたらパーティショニングの警告画面のところまで進んだ。  
DHCPは一本に絞りたいけど、同じにするとなぜかPXEクライアントがDHCP解決した後のLinuxブートシーケンスでDHCP解決に失敗する。  
どげんかせんといかん。

```
# /var/lib/tftpboot/preseed.cfg
d-i debian-installer/language string en
d-i debian-installer/country string US
d-i debian-installer/locale string en_US.UTF-8
d-i localechooser/supported-locales en_US.UTF-8d-i console-setup/ask_detect boolean false
d-i console-setup/layoutcode string us
d-i console-setup/charmap select UTF-8
d-i keyboard-configuration/layoutcode string us

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

d-i mirror/country string manual
d-i mirror/http/hostname string archive.ubuntu.com
d-i mirror/http/directory string /ubuntu
d-i mirror/http/proxy string

d-i clock-setup/utc boolean false 
d-i time/zone string Japan 
d-i clock-setup/ntp boolean false 

d-i partman-auto/init_automatically_partition select biggest_free 
d-i partman-auto/disk string /dev/sda
d-i partman-auto/method string regular 
d-i partman-lvm/device_remove_lvm boolean true 
d-i partman-auto/choose_recipe select atomic 
d-i partman/default_filesystem string ext4 
d-i partman-partitioning/confirm_write_new_label boolean true 
d-i partman/choose_partition select finish 
d-i partman/confirm boolean true 
d-i partman/confirm_nooverwrite boolean true 
d-i partman-partitioning/confirm_write_new_label boolean true 
d-i partman/choose_partition select finish 
d-i partman/confirm boolean true 
d-i partman/confirm_nooverwrite boolean true 
d-i partman/mount_style select traditional

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
d-i apt-setup/use_mirror boolean false 
d-i debian-installer/allow_unauthenticated boolean true 
tasksel tasksel/first multiselect none 
d-i pkgsel/include string openssh-server build-essential
d-i pkgsel/upgrade select none 
d-i pkgsel/update-policy select none 
popularity-contest popularity-contest/participate boolean false 
d-i pkgsel/updatedb boolean true 
d-i grub-installer/grub2_instead_of_grub_legacy boolean false 
d-i grub-installer/only_debian boolean true 
d-i grub-installer/bootdev string (hd0,0) 
d-i finish-install/reboot_in_progress note
```

## デバッグに関して

*僕はここではpartmanの設定の仕方をマスターしたいので調べました。*

カーネルパラメータに `DEBCONF_DEBUG=5` を指定する。  
出力されるログは2つ？  
2つある。どっちが正しいのかわからない。

1. `/var/log/installer/syslog`  
死ぬほど長い。なにこれ。。。
1. `/var/log/syslog`  
許せるレベル。

なんか、 `/var/log/syslog` は違う気がするんだよなあ。。。

## 参考サイト

- [Ubuntu Serverの全自動インストール環境作成 - kinneko@転職先募集中の日記](http://d.hatena.ne.jp/kinneko/20130203/p1)
- [Ubuntu Serverの完全自動インストールISOの作成（Preseeding） - SharpLab.](http://blog.sharplab.net/blog/2012/11/11/ubuntu-server%E3%81%AE%E5%AE%8C%E5%85%A8%E8%87%AA%E5%8B%95%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%ABiso%E3%81%AE%E4%BD%9C%E6%88%90%EF%BC%88preseeding%EF%BC%89/)
- [PreseedによるUbuntu ServerのインストールCD作成手順(PDF)](http://h50146.www5.hp.com/products/software/oe/linux/mainstream/support/lcc/pdf/edlin_20110804.pdf)
- [Appendix B. Automating the installation using preseeding](https://help.ubuntu.com/12.04/installation-guide/amd64/appendix-preseed.html)
- [Contents of the preconfiguration file (for precise)](https://help.ubuntu.com/lts/installation-guide/i386/preseed-contents.html)
- [B.2. preseed の利用](http://www.debian.org/releases/stable/s390x/apbs02.html.ja)
- [Advanced options](https://help.ubuntu.com/lts/installation-guide/i386/preseed-advanced.html)
- [preseedを使ってDebian GNU/Linux 5.0.4(netinst)のインストール自動化を行う手順 - 富士山は世界遺産](http://d.hatena.ne.jp/fujisan3776/20100630/1277861431)
- [Contents of the preconfiguration file (for precise)](https://help.ubuntu.com/lts/installation-guide/i386/preseed-contents.html#preseed-bootloader)
- [GPT対応のpreseedの書き方 — ペンギンと愉快な機械の日々](http://d.palmtb.net/2012/12/14/writing_preseed_for_gpt.html)
- [Notes on using expert_recipe in Debian/Ubuntu Preseed Files | Semi-Empirical Shenanigans](http://cptyesterday.wordpress.com/2012/06/17/notes-on-using-expert_recipe-in-debianubuntu-preseed-files/)
