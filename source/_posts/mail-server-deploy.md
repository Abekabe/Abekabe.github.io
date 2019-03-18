---
title: 佈署郵件伺服器
date: 2019-03-20 19:08:35
tags: NA
---

# 用SMTP佈署mail server
這份紀錄也是我在上NA時的一份作業紀錄，主要目的事要架起一個mail server，並且擁有spf,dkim,dmarc等保護協定的認證以及很多其餘的小功能，以完成一個安全的mail server，至於這幾個協定的功能我這邊就不再贅述，只敘述過程。

## 佈署SMTP
send port gcp ban
smtp_tls_security_level = encrypt
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
header_size_limit = 4096000
smtp_sasl_security_options = noanonymous


openssl req -new -x509 -days 365 -nodes -out /etc/postfix/ssl_key/smtp.key -keyout /etc/postfix/ssl_key/smtp.pam
postconf -e 'smtpd_tls_cert_file = /etc/postfix/ssl_key/smtp.pam'
postconf -e 'smtpd_tls_key_file = /etc/postfix/ssl_key/smtp.key'

sudo apt install dovecot-common dovecot-imapd dovecot-pop3d

### /etc/dovecot/conf.d/10-auth.conf
disable_plaintext_auth = yes
...
auth_mechanisms = plain login
http://lomu.me/post/SPF-DKIM-DMARC-PTR

### /etc/dovecot/dovecot.conf
protocols = pop3 imap

### ~/forward
hw4user@naphw4.nctucs.net
ms0229643@gmail.com
\hw4user



### /etc/postfix/main.cf
virtual_alias_maps = regexp:/etc/postfix/virtual

----------------------
### /etc/postfix/virtual
/^.*demo.*@mail.nphw2-0410817.nctucs.net$/ hw4user@mail.nphw2-0410817.nctucs.net
/^(.*)+.*@mail.nphw2-0410817.nctucs.net$/ $1@mail.nphw2-0410817.nctucs.net



## Sender Rewriting Schema
sudo apt install postsrsd
sudo postconf -e "sender_canonical_maps = tcp:127.0.0.1:10001"
sudo postconf -e "sender_canonical_classes = envelope_sender"
sudo postconf -e "recipient_canonical_maps = tcp:127.0.0.1:10002"
sudo postconf -e "recipient_canonical_classes = envelope_recipient,header_recipient"


## Postgrey
sudo apt-get install postgrey
sudo vim /etc/default/postgrey
POSTGREY_OPTS="--inet=10023  --delay=60"


## SMTP reject
smtpd_recipient_restrictions = permit_sasl_authenticated,
        permit_mynetworks,
        reject_unauth_destination,
        reject_rbl_client zen.spamhaus.org,
        check_policy_service inet:127.0.0.1:10023

## SPF
sudo vim /etc/bind/zones/db.nphw2-0410817.nctucs.net
;SPF record
mail    IN      TXT     "v=spf1 a mx -all"
mail    IN      SPF     "v=spf1 a mx -all"
https://serverfault.com/questions/126828/configure-a-spf-rule-on-ubuntu?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa

sudo vim /etc/opendkim.conf
Domain    mail.nphw2-0410817.nctucs.net
KeyFile    /etc/postfix/dkim.key
Selector    dkim
SOCKET    inet:8891@localhost

sudo vim /etc/default/opendkim
SOCKET="inet:8891@localhost"

sudo vim /etc/postfix/main.cf

## DKIM
milter_default_action = accept
milter_protocol = 2
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891


### genkey
opendkim-genkey -t -s dkim -d mail.nphw2-0410817.nctucs.net
dkim._domainkey IN      TXT     ( "v=DKIM1; k=rsa; t=y; "
          "p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCq8fM4NIW8DvJ5VW6YbRM+bkW/uKoWBpVdBpCgECQq2m95ho7ipTYkdzRQ7FeMbXE7E/2QsfagvjWkdU7H9lwW4RaptPHZ9FO5fqOD1HWVLQS0d4astoyT75Kxe+damLwp2VrlDCC2W4StYJ8FPSr95mYZuOzwLWmN8azjSyRYpwIDAQAB" )  ; ----- DKIM key dkim for mail.nphw2-0410817.nctucs.net

dkim._domainkey IN      TXT     ( "v=DKIM1; k=rsa; t=y; "
    "p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDLZTmRvIwPU7t4X45Ox0aG/+qIE7ejcemQd6BVvSEOV//Fbbnc/8xf/QI+yYEyYEBAAhsb/CeXOrVbKLLl+8qZGIeT8AVhuqXYy+k6G1JIg8E306z/LYJjS3YleOFJ1Z54A0Vqk7LfqaUFtFcOv+sn5RpLwSlTk0ffI9rV55040wIDAQAB" )  ; ----- DKIM key dkim for nphw2-0410817.nctucs.net

dkim._domainkey　TXT　v=DKIM1; k=rsa;s=email;p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwsf4PKUbGHLFkXYmER8W
0jOdokHeahe26fBY0oB/qKnTxBuIkb62JcGWn97wYEoyRIX22YHvB9k34rN2dDOY
relR8c0KOLT57RXFhd3BhnREz631AEuNmIWv0avxQWM5rAZBT1CIGQD67ucnoiM4
dVT3Nyn4fGD3c4fiosGkxxcg8TjB5f60vJq7CLe+FenqchCHgC82NyN/1yGU2abI
bmiFENAgLfwyME0OcD3HTq+8NU1pibwRmIu993Xj27gHMktYfw8K9FghNZGsuGaz
6d/xfUCL2jKKGclkXAW+SD3OGqqCl+kPf7kFNpCASZfzbsCpY9OZw6THvnQVvM7H
nwIDAQAB



### add to bind record
sudo vim /etc/bind/zones/db.nphw2-0410817.nctucs.net
;DKIM record
dkim._domainkey.mail     IN     TXT     "v=DKIM1; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCq8fM4NIW8DvJ5VW6YbRM+bkW/uKoWBpVdBpCgECQq2m95ho7ipTYkdzRQ7FeMbXE7E/2QsfagvjWkdU7H9lwW4RaptPHZ9FO5fqOD1HWVLQS0d4astoyT75Kxe+damLwp2VrlDCC2W4StYJ8FPSr95mYZuOzwLWmN8azjSyRYpwIDAQAB"



https://www.exratione.com/2014/07/setting-up-spf-and-dkim-for-an-ubuntu-1404-mail-server/

## DMARC
sudo vim /etc/bind/zones/db.nphw2-0410817.nctucs.net
;DMARC record
_dmarc.mail 1800 IN TXT "v=DMARC1; p=reject; rua=hw4user@mail.nphw2-0410817.nctucs.net; ruf=hw4user@mail.nphw2-0410817.nctucs.net"
https://petermolnar.net/howto-spf-dkim-dmarc-postfix/


## Ngnix
https://blog.gtwang.org/linux/nginx-create-and-install-ssl-certificate-on-ubuntu-linux/
