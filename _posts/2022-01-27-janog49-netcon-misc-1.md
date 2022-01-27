---
layout: post
title:  "JANOG49 NETCON問題解説"
date:   2022-01-27 17:00:00
---
JANOG49 NETCONで問題作成をしました。
カテゴリ：MISC(その他)
問題：MISC-1
配点：30点

# **問題文**
友人に自宅のインターネットが使えないと相談されました。
詳しく話を聞くと、一部のサイトにアクセスできないようです。
ひとまずClientからInternetServerに対してcurlしてみます。
```
janog49@Client:~$ curl internetserver.netcon
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.20.1</center>
</body>
</html>
```
リダイレクトされました。では、`https://internetserver.netcon` はどうでしょうか？
```
janog49@Client:~$ curl --max-time 30 https://internetserver.netcon
curl: (28) Operation timed out after 30001 milliseconds with 0 out of 0 bytes received
```
タイムアウトしてしまいました。

**問題**

`curl https://internetserver.netcon`が成功するようにしてください。
- 期待する結果
```
janog49@Client:~$ curl https://internetserver.netcon
Succeeded!
```
**制限事項**
- InternetServerとProviderDNSの設定は変更してはいけない
- InternetRouterとProviderRouterにはログインできない

# **構成**
![arch](https://ashg0.github.io/assets/images/20220127_janog49netcon.PNG)



### **環境設定**
<details>
  <summary>InternetServer</summary>

```
echo InternetServer > /etc/hostname
yum install -y nginx
systemctl enable nginx
systemctl disable firewalld
nmcli c mod eth0 ipv4.addresses "10.0.0.100/24"
nmcli c mod eth0 ipv4.method "manual"
nmcli c mod eth0 ipv4.gateway "10.0.0.1"
echo 'Succeeded!' > /usr/share/nginx/html/index.html
openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout /etc/ssl/certs/nginx_server.key -out /etc/ssl/certs/nginx_server.crt
cp /etc/ssl/certs/nginx_server.crt /etc/pki/ca-trust/source/anchors
update-ca-trust
```
</details>

<details>
  <summary>nginx.conf</summary>

```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
 
events {
    worker_connections 1024;
}
 
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
 
    access_log  /var/log/nginx/access.log  main;
 
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;
 
    server {
        listen       80;
        server_name  _;
        return 301 https://$host$request_uri;
        }
 
    server {
        listen       443 ssl http2;
        server_name  internetserver.netcon;
        root         /usr/share/nginx/html;
 
        ssl_certificate "/etc/ssl/certs/nginx_server.crt";
        ssl_certificate_key "/etc/ssl/certs/nginx_server.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
 
        error_page 404 /404.html;
            location = /40x.html {
        }
 
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
```
</details>

<details>
  <summary>ProviderDNS</summary>

```
echo ProviderDNS > /etc/hostname
nmcli c mod eth0 ipv4.addresses "172.16.2.100/24"
nmcli c mod eth0 ipv4.method "manual"
nmcli c mod eth0 ipv4.gateway "172.16.2.1"
yum install -y dnsmasq
systemctl enable dnsmasq
echo '10.0.0.100 internetserver.netcon' >> /etc/hosts
echo local=/netcon/ > /etc/dnsmasq.conf
systemctl disable firewalld
```
</details>

<details>
  <summary>InternetRouter</summary>

```
!
! Last configuration change at 02:14:10 UTC Sun Jan 16 2022
!
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname InternetRouter
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 ip address 172.16.1.2 255.255.255.0
 ip access-group 100 in
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip route 172.16.0.0 255.255.0.0 172.16.1.1
!
ipv6 ioam timestamp
!
!
access-list 100 deny   icmp host 172.16.1.1 any
access-list 100 permit ip any any
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
line aux 0
line vty 0 4
 login
 transport input none
!
no scheduler allocate
!
end
```
</details>

<details>
  <summary>ProviderRouter</summary>

```
!
! Last configuration change at 15:17:29 UTC Sat Jan 15 2022
!
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname ProviderRouter
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
ip name-server 172.16.2.100
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
username janog49 password 0 netcon49
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
bba-group pppoe bba1
 virtual-template 1
!
!
interface Loopback1
 ip address 172.16.0.1 255.255.255.0
!
interface GigabitEthernet0/0
 no ip address
 duplex auto
 speed auto
 media-type rj45
 pppoe enable group bba1
!
interface GigabitEthernet0/1
 ip address 172.16.1.1 255.255.255.0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 ip address 172.16.2.1 255.255.255.0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
 pppoe enable group bba1
!
interface Virtual-Template1
 description pppoe bba1
 mtu 1492
 ip unnumbered Loopback1
 peer default ip address pool pool1
 ppp authentication pap
!
ip local pool pool1 172.16.0.2
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip route 10.0.0.0 255.0.0.0 172.16.1.2
!
ipv6 ioam timestamp
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
line aux 0
line vty 0 4
 login
 transport input none
!
no scheduler allocate
!
end
```
</details>

<details>
  <summary>HomeRouter</summary>

```
set system host-name HomeRouter
set interfaces pppoe pppoe0 default-route 'auto'
set interfaces pppoe pppoe0 mtu 1492
set interfaces pppoe pppoe0 authentication user 'janog49'
set interfaces pppoe pppoe0 authentication password 'netcon49'
set interfaces pppoe pppoe0 source-interface 'eth0'
set nat source rule 100 outbound-interface 'pppoe0'
set nat source rule 100 source address '192.168.0.0/24'
set nat source rule 100 translation address 'masquerade'
set service dhcp-server shared-network-name LAN subnet 192.168.0.0/24 default-router '192.168.0.1'
set service dhcp-server shared-network-name LAN subnet 192.168.0.0/24 dns-server '192.168.0.1'
set service dhcp-server shared-network-name LAN subnet 192.168.0.0/24 lease '86400'
set service dhcp-server shared-network-name LAN subnet 192.168.0.0/24 range 0 start 192.168.0.2
set service dhcp-server shared-network-name LAN subnet 192.168.0.0/24 range 0 stop '192.168.0.10'
set service dns forwarding cache-size '0'
set service dns forwarding listen-address '192.168.0.1'
set service dns forwarding allow-from '192.168.0.0/24'
set service dns forwarding name-server 172.16.2.100
commit
scp janog49@internetserver.netcon:/etc/ssl/certs/nginx_server.crt /etc/ssl/certs/
c_rehash
scp /etc/ssl/certs/nginx_server.crt janog49@192.168.0.2:
```
</details>

<details>
  <summary>Client</summary>

```
echo Client > /etc/hostname
cp nginx_server.crt /etc/ssl/certs/
c_rehash
```
</details>

