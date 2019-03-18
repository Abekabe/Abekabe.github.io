---
title: 使用BIND架設DNS server
date: 2018-07-20 18:20:57
tags: NA
---
# 使用BIND架設DNS server
這次的NA作業是在伺服器提供DNS（Domain Name System）的服務，甚至管理log檔與簽章都是作業的規格之一，這份作業我花了將近30小時才順利完成（汗，以下就來講解DNS server的架設與管理吧！


## BIND
BIND使用zone來區分不同Domain name的指向，原本該設定的named.conf include了三個檔案方便分別放不同功能的設定，而zone的設定通常是打在named.conf.local裡面，以下是zone的基本格式</p>
```
zone "your domain-name" {
		type ; // master or slave
		file ; // your file path
	}
```
以下是真實檔案的範例:
```
zone "nphw2-0410817.nctucs.net" {
        type master;
        file "/etc/bind/zones/internal/db.sub.nphw2-0410817.nctucs.net";
        allow-transfer { trust; };         # ns2 private IP address - secondary
        notify yes;
    };
```
type 分為master與slave，master就是這個zone紀錄檔的擁有者，zone file由他編寫，而slave則可以同步master更新後的檔案，allow-transfer 為了要紀錄哪個ip可以直接拿到你dns的紀錄檔，通常設定自己信任的伺服器或是可以設為 any。


## Zone file
接下來是zone file的編寫，首先要知道zone file裡面的紀錄分成很多種type，最基本的分別是

* SOA record

用來紀錄zone file的資訊，例如擁有者是誰、版本等等，基本格式如下:
```
@       IN      SOA     owner dns server domain 	owner email (
                        ; Serial
                        ; Refresh
                        ; Retry
                        ; Expire
                        ; Negative Cache TTL
                        )
```


* NS record

用來紀錄哪些dns server在管理這個zone file，因為一個zone file不只會存在一台伺服器中，因此提供管理的server才能找到此zone file，基本格式如下：
```
@       IN      NS		dns server domain

```



* A record

用來紀錄domain name對應到的IP address，可以說是dns最重要的功能了吧？基本格式如下：
```
Domain name		IN      A       IP address
```

* MX record

用於紀錄信箱domain的形式，用法跟A record一樣這邊就不贅述了

* CNAME record

通常打網址的時候常常會多打或打錯，最常見的就是多打了www了，CNAME record可以將兩個網址指到同一個A record上面，以處理這種錯誤，基本格式如下：
```
domain    IN      CNAME   domain which your already record
```
還有一些額外的用法可以縮減你打相同domain例如：

```
$ORIGIN nphw2-0410817.nctucs.net.
```

可以讓你在下面的record都不用再打後面一長串，像是sub.nphw2-0410817.nctucs.net.就只要打sub就好了

以下是zone file的真實檔案範例：

```
$TTL    3600
@       IN      SOA     ns.nphw2-0410817.nctucs.net. admin.nphw2-0410817.nctucs.net. (
                        2018042502      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
; name servers - NS records
@       IN      NS      ns.nphw2-0412241.nctucs.net.
@       IN      NS      ns.nphw2-0410817.nctucs.net.

$ORIGIN nphw2-0412241.nctucs.net.
; name servers - A records
@       IN      A       167.99.71.71
ws      IN      A       140.113.235.151
demo    IN      CNAME   ws
```

這樣一來就建立好最基本DNS server的功能。


## Master and Slave
因為zone file 不能只有一台dns server 有（避免server 掛掉就連不上了），但每一台server 都要重打一份一模一樣的zone file 既浪費時間又很容易每一台打的有些為不同，而且要修改資訊就要每一台都改，因此可以使用master and slave 的方法，這種方法就是zone file 由master 編寫，slave 則在zone file 有修改的時候將其下載回來，那又如何實做呢？在master 與slave 的zone config 分別是這樣寫：

* Master zone config:

```
one "your domain-name" {
        type master;
        file "your zone file location";
        allow-transfer { "your slave server ip" };         # ns2 private IP address - secondary
        notify "yes or no";
    };
```

* Slave zone config:

```
zone "your domain-name" {
        type slave;
        file "location your want to put zone file";
        masters { your master server ip };
    };
```

如此設定下slave 會在一段時間之後自動去檢查master 那邊的 zone file 是否有更新（利用serial number），若是master 有設定notify yes 的話，在檔案修改之後master 會主動通知slave 要更新zone file。


## View
當你希望不同IP 連進你的dns server 會獲得不同的資訊，例如有些公司內部的資訊不希望外界的人可以獲得，就會以使用view 這個功能來區分不同 IP 所能看到的資料，其實使用起來也是蠻簡單的，主要架構如下：
```
view "view's name" {
    match-clients { "Which ip you want to access" };
    /*
    	Put zone config here
    */
};
```
這麼一來只有你希望的match-client 可以讀到這個view 裡面的東西，藉由放不同的zone config 就可以使不同IP 得到不同的zone file 資訊，值得注意的是一旦使用了view ，所有config file 都要在view 的包裹之下才能正常運行Bind 。 所以不只named.config.local 要如此，named.config.default 也要如此，不然Bind 是跑不起來的。


## DNSSEC
一開始看到DNSSEC這個名稱還以為這是用來確認USER 與DNS server 之間連線安全性的，結果DNSSEC 實際上的功能卻是讓USER 確認你問到的DNS server 是否確實是由parent server delegate 的，也可以防止DNS record 被串改，以確保USER 不會收到假的DNS record。

首先我們必須產生公鑰，公鑰是由ZSK, KSK 所組成，產生ZSK, KSK 的方法如下:

```
dnssec-keygen -a NSEC3RSASHA1 -b 2048 -n ZONE example.com
dnssec-keygen -f KSK -a NSEC3RSASHA1 -b 2048 -n ZONE example.com
```

這邊使用的加密法是NSEC3RSASHA1，長度則是2048，要鎖的domain name 則是打在ZONE 的後面，在執行完這兩行後就可以於原目錄中找到四個新檔案，分別是KSK(公鑰, 私鑰), ZSK(公鑰, 私鑰)，大概會像”Kexample.com.+007+40400.key” 的結構，結尾.key 的為公鑰 .private 則為私鑰，請記好在產生時ZSK 與KSK 的編號以免打反了，接著輸入：

```
cp example.com.zone example.com.dnssec 					＃ 複製一份準備要簽署的Zone file
cat Kexample.com.+007+40400.key > example.com.dnssec	＃ 加入KSK公鑰
cat Kexample.com.+007+60468.key > example.com.dnssec 	＃ 加入ZSK公鑰
```

將公鑰檔案一順序貼到要簽署的Zone file 之後就可以進行簽署：

```
dnssec-signzone -3 123456 -o example.com -s 20130728000000 -e 20130902000000 -k Kexample.com.+007+40400.key example.com.dnssec Kexample.com.+007+60468.key
```

簽署時大概是要設定domain name, 簽署時間, 到期時間, KSK公鑰, 準備要簽署的Zone file, 以及ZSK公鑰，更詳細的指令這邊就不多細講，簽署完後會出現兩個檔案大概會像下面兩個名字：

```
example.com.dnssec.signed	# 簽署後的Zone file檔
dsset-example.com			＃ DS record
```

仔細看簽署後的Zone file 裡面會發現中間多了兩種record，首先DNSKEY record 是紀錄了KSK與ZSK公鑰，而RRSIG則是將原本Zone file 裡面的record用私鑰進行簽署，因此USER 拿到的Zone file 一定是無法被別人隨便更改的，而DS record 會交給parent server 擺在他的zone config 裡面，USER 便可以確認他詢問的DNS server 是否有被parent server 授權。


## 後記
第一次打網誌發現其實花蠻多時間的，這篇我從學期初打到暑假才打完zz，不過其實收穫也蠻多的，除了可以將自己所作的記下來以防自己忘記，還有許多在做作業時沒有用懂的地方因為要寫成教學因此又上網重新用懂之後才敢放上來，不過這樣做真的有點花時間，所以之後關於NA 的作業可能不會做太多的解釋，只會將大概的作法放上來，而會花更多時間做自己比較了解的主題。

