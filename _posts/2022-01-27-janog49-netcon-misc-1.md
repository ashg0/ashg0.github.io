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

- **問題**

  `curl https://internetserver.netcon`が成功するようにしてください。
- **期待する結果**
```
janog49@Client:~$ curl https://internetserver.netcon
Succeeded!
```

- **制限事項**

  InternetServerとProviderDNSの設定は変更してはいけない

  InternetRouterとProviderRouterにはログインできない

# **構成**
![arch](https://ashg0.github.io/assets/images/20220127_janog49netcon.PNG)

# **解説**
この問題はPath MTU black holeとTCP MSS Clampingを題材にしています。

作問の発端は、自宅NWでフレッツ回線にLinux刺してルーティングしていた時に、同じように嵌ったことです。


**切り分け**

port 80へのcurlは通っているのでL3の疎通はあると考えられます。

port 443がタイムアウトするため、ACL等の設定が予想されますが、HomeRouterからのcurlは通ることから否定されます。
```
janog49@HomeRouter:~$ curl https://internetserver.netcon
Succeeded!
```
Clientからのcurlを詳しく見てみます。
```
janog49@Client:~$ curl -v https://internetserver.netcon
*   Trying 10.0.0.100:443...
* TCP_NODELAY set
* Connected to internetserver.netcon (10.0.0.100) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
^C
janog49@Client:~$
```
Client helloの後、Server helloが返ってきていないことがわかります。戻りのどこかでパケットが破棄されているようです。

InternetServer側でtcpdumpを取り、ClientとHomeRouterのリクエストを比較してみます。
- Client
```
14:55:21.933009 IP 172.16.0.2.38524 > InternetServer.https: Flags [S], seq 2562719857, win 64240, options [mss 1460,sackOK,TS val 850820184 ecr 0,nop,wscale 7],length 0
14:55:21.933338 IP InternetServer.https > 172.16.0.2.38524: Flags [S.], seq 1536893779, ack 2562719858, win 28960, options [mss 1460,sackOK,TS val 13818108 ecr850820184,nop,wscale 7], length 0
14:55:21.939751 IP 172.16.0.2.38524 > InternetServer.https: Flags [.], ack 1, win 502, options [nop,nop,TS val 850820191 ecr 13818108], length 0
```
- HomeRouter
```
14:55:40.200944 IP 172.16.0.2.47162 > InternetServer.https: Flags [S], seq 298814695, win 65340, options [mss 1452,sackOK,TS val 2507845421 ecr 0,nop,wscale 6], length 0
14:55:40.201124 IP InternetServer.https > 172.16.0.2.47162: Flags [S.], seq 3987956370, ack 2988814696, win 28960, options [mss 1460,sackOK,TS val 13836376 ecr2507845421,nop,wscale 7], length 0
14:55:40.206723 IP 172.16.0.2.47162 > InternetServer.https: Flags [.], ack 1, win 1021, options [nop,nop,TS val 2507845426 ecr 13836376], length 0
```

Synパケットを比較すると、HomeRouterではmss 1452, Clientではmss 1460 となっています。これが原因のようです。

HomeRouterの設定を確認すると、ProviderへはPPPoEで接続されています。
```
set interfaces pppoe pppoe0 authentication password 'netcon49'
set interfaces pppoe pppoe0 authentication user 'janog49'
set interfaces pppoe pppoe0 default-route 'auto'
set interfaces pppoe pppoe0 source-interface 'eth0'
```
PPPoEではPPPoEヘッダ、PPPヘッダに8byteが必要です。そのためpppoeインターフェスのmtuは1500 - 8 = 1492byteとなります。

これを上回るサイズのパケット(DFフラグがセット)は破棄され、 ICMP Type:3 / Code:4(too big)が返されます。

通常、too bigを受け取った場合は、Server側でpacketサイズを下げて再送することで、到達不能を回避します。(Path MTU Discovery)しかし、この問題環境では何故かtoo bigがServer側に返らないようになっているため、Black holeが発生していました。
※InternteRouterでProviderRouterからのICMPをDropしていました。本文末show runのaccess-list参照

本問題では、TLSのhandshakeでサーバ証明書を送る際にmtuを超過してしまい、破棄されています。

TCPでは3WayHandshakeの際に使用するMSSサイズを交換し、小さい方を採用します。TCPヘッダのMSSサイズはエンドポイントのほかに、経路上でも変更することが可能です。（TCP MSS Clamping）本問題ではこれをHomeRouterで実施することを回答として想定しています。


**所感**

Client,Serverでの切り分けをしてみましたが、PPPoEでProviderRouterに接続されている構成を見た時点で、mtuの問題と予想がついた方もおられるかと思います。

自分が自宅で切り分けた際は、mtuの観点が無かったので嵌りました。



**解答例**
- HomeRouter(VyOS)でtcp mss clamping

  `set firewall options interface pppoe0 adjust-mss '1452'`など

Client側でIFのmtuを下げることでもcurlは通りますが、HomeRouter配下LAN内の全てのノードで設定必要になります。(自分の自宅の例だとClientがWindowsで、WSL上でもmtu設定しないといけない…)

採点自体は正答にしました。

また、dhcpのoptionでmtuも配布できるらしいです。(windows10では設定できないようですが)


**採点方法**

この問題では自動採点を実施していました。

回答者の問題環境にログインし、Clinetからcurlを実行して期待する出力が得られるかテストするプログラムを実行しました。

単純にClientからのcurl確認のみだと、Clientの/etc/hostsに`127.0.0.1 localhost internetserver.netcon`を記載してローカルにWeb server立てる等の不正が可能なため、InternetServer側のログも確認しました。
- 実際の採点ログ ※`H1A9K0G4`はランダム文字列を都度生成
```
  "Client: curl -A \"H1A9K0G4\" -m 5 https://internetserver.netcon": "Succeeded!",
  "Server: grep H1A9K0G4 /var/log/nginx/access.log | wc -l": "1"
```
- 採点基準

  Succeeded!が返ってきていたら50%

  wc -l が 1の場合は100%

**その他**

証明書のサイズが小さいとTLSのハンドシェイクは通ります。`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.crt`だと1300byteくらいのパケットになったので、問題環境では`rsa:4096`で作成しました。

DNSが構成内に存在していますが、問題内容とは無関係になっていました。切り分けポイントを増やしたい意図で配置しました。


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

