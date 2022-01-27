---
layout: post
title:  "ブログ作った"
date:   2022-01-27 17:00:00
---
JANOG49 NETCONで問題作成をしました。
カテゴリ：MISC(その他)
問題：MISC-1
配点：30点

# 問題文
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
- 問題
`curl https://internetserver.netcon`が成功するようにしてください。
- 期待する結果
```
janog49@Client:~$ curl https://internetserver.netcon
Succeeded!
```
- 制限事項
InternetServerとProviderDNSの設定は変更してはいけない
InternetRouterとProviderRouterにはログインできない

# 構成
![arch](https://ashg0.github.io/assets/images/20220127_janog49netcon.PNG)



### 環境構築の設定
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

