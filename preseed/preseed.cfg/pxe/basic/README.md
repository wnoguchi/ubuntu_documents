# PXE boot Basic Preseeding

- PXEブートタイプの設定ファイル
- 一番原始的な設定

## preseed.cfg

RX100S7のPXEとIPMI管理ポート、PXE+DHCPサーバーのセグメントと、eth0の通常のNICには  
NTTのブロードバンドルーターのDHCPの下に置いてDHCP2本立てにしたらパーティショニングの警告画面のところまで進んだ。  
DHCPは一本に絞りたいけど、同じにするとなぜかPXEクライアントがDHCP解決した後のLinuxブートシーケンスでDHCP解決に失敗する。  
どげんかせんといかん。
