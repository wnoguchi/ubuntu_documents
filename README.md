# Ubuntu Documents

Ubuntuのドキュメント。

## ネットワーク

```
# etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp
```

### DHCP

```
# etc/network/interfaces
auto eth0
iface eth0 inet dhcp
```

### 静的IP

DNSの設定は `resolv.conf` に直接書くんじゃなくて interfaces に書くようにする。  
`dns-nameservers` で設定する。スペース区切り。

```
# etc/network/interfaces
auto eth0
iface eth0 inet static
    address 192.168.0.10
    netmask 255.255.255.0
    network 192.168.0.0
    broadcast 192.168.0.255
    gateway 192.168.0.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

して

```
sudo initctl restart networking
```

上のやつ、Ubuntu13.04 Desktopでかましたら内部エラーとかいわれちゃう。なんで。。。
