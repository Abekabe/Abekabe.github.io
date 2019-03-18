---
title: 佈署VPN及SNMP
date: 2019-03-19 19:00:05
tags: NA
---
# VPN and SNMP
這份紀錄也是我在修NA的其中一份作業，因為時間關係我就沒有做很完整的整理，只有大概把做作業時做的事記錄下來，如果剛好需要建構VPN和SNMP系統可以參考一下。


## VPN

apt-get update
apt-get install openvpn easy-rsa
gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf

### /etc/openvpn/server.conf
```
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
```

tls-auth ta.key 0 # This file is secret
key-direction 0
cipher AES-128-CBC
auth SHA256
;user nobody
;group nogroup
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"

iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE

cp -r /usr/share/easy-rsa/ /etc/openvpn
mkdir /etc/openvpn/easy-rsa/keys
vim /etc/openvpn/easy-rsa/vars
export KEY_COUNTRY=“TW”
export KEY_PROVINCE=“Taiwan”
export KEY_CITY=“Taipei”
export KEY_ORG="NCTU"
export KEY_EMAIL="ms0229643@gmail.com"
export KEY_OU="NCTU"
export KEY_NAME=“server”

openssl dhparam -out /etc/openvpn/dh2048.pem 2048

cd /etc/openvpn/easy-rsa
source ./vars
./clean-all
./build-ca
./build-key-server server
./build-dh
openvpn --genkey --secret keys/ta.key
./build-key client1
cd keys
cp ca.crt server.crt server.key ta.key dh2048.pem /etc/openvpn


echo 1 > /proc/sys/net/ipv4/ip_forward
vim /etc/sysctl.conf


### /etc/sysctl.conf
* 拿掉註解
*
net.ipv4.ip_forward=1


service openvpn start
service openvpn status

cd /etc/openvpn
mkdir -p client-configs/files
chmod 700 client-configs/files
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf client-configs/base.conf


### build.sh
```
#!/bin/bash
```

### First argument: Client identifier
```
KEY_DIR=/etc/openvpn/easy-rsa/keys
OUTPUT_DIR=/etc/openvpn/client-configs/files
BASE_CONFIG=/etc/openvpn/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
~

./build.sh client
```

## SNMP
apt-get install snmp snmpd
apt-get install mrtg
cp /etc/snmp/snmpd.conf /etc/snmp/snmpd.conf.bak

### /etc/snmp/snmpd.conf
```
rocommunity  private

#rocommunity public  default    -V systemonly
                                                 #  rocommunity6 is for IPv6
#rocommunity6 public  default   -V systemonly
 rocommunity user  default    -V systemonly
                                                 #  rocommunity6 is for IPv6
 rocommunity6 user  default   -V systemonly

;cfgmaker user@localhost > /etc/mrtg.cfg
cfgmaker -zero-speed=10000000 user@localhost > /etc/mrtg.cfg
env LANG=C mrtg /etc/mrtg.cfg
indexmaker /etc/mrtg.cfg > /var/www/mrtg/index.html
```
### mem.cfg
```
WorkDir: /var/www/mrtg
RunAsDaemon: yes
Refresh: 300
Interval: 10

Target[managemem]:.1.3.6.1.4.1.2021.4.6.0:user@localhost
Unscaled[managemem]: dwym
MaxBytes[managemem]: 2048000
Title[managemem]:Memory
ShortLegend[managemem]: &
kmg[managemem]:kB,MB
kilo[managemem]:1024
YLegend[managemem]: Memory Usage
Legend1[managemem]: Total Memory
Legend2[managemem]: Used Memory
LegendI[managemem]: Total Memory
LegendO[managemem]: Used Memory
Options[managemem]: growright,gauge,nopercent
PageTop[managemem]: <h1>Memory</h1>
```


### cpu.cfg
```
WorkDir: /var/www/mrtg
RunAsDaemon: yes
Refresh: 300
Interval: 10
LoadMIBs: /usr/share/snmp/mibs/UCD-SNMP-MIB.txt
Target[cpu]:ssCpuRawUser.0&ssCpuRawSystem.0:user@localhost

RouterUptime[cpu]: user@localhost
Title[cpu]: CPU Load
Options[cpu]:      growright, gauge, integer, nopercent
MaxBytes[cpu]:     100
YLegend[cpu]:      % of CPU used
ShortLegend[cpu]:  %
LegendI[cpu]:      &nbsp; User:
LegendO[cpu]:      &nbsp; System:
Legend1[cpu]:      User utilization
Legend2[cpu]:      System utilization
Colours[cpu]:      ORANGE#ff6128,GREEN#066928,DARK GREEN#006600,VIOLET#ff00ff
PageTop[cpu]:      <H1>CPU Loading</H1>
```

### indexmaker.sh
```
/usr/bin/indexmaker -output=/var/www/mrtg/index.html \
-title="Server Usage" \
-sort=name \
-enumerate \
/etc/mrtg.cfg \
/etc/mrtg/cpu.cfg \
/etc/mrtg/mem.cfg \
```



