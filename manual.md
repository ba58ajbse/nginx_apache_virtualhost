## 設定手順

1. WebServerのバーチャルホスト設定
2. ApplicationServerのバーチャルホスト設定
3. Webserverでの証明書設定
4. 証明書の自動更新設定

### 1. WebServerのバーチャルホスト設定

バーチャルホストを設定するため、`/etc/nginx/conf.d/example_1.conf`と`/etc/nginx/conf.d/example_2.conf`を作成する

ApplicationServerへの接続先をドメイン毎にポート番号で振り分ける

- `/etc/nginx/conf.d/example_1.conf`

```bash
# /etc/nginx/conf.d/example_1.conf
server {
    listen       80;
    server_name  example_1.com;

	# ログの設定
    access_log  /var/log/example_1.com/access_log;
    error_log   /var/log/example_1.com/error_log;
		
    # ローカルIPで接続されたApplicationServerへの接続設定
    location / {
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_pass  http://192.168.0.2:8080;
    }
}
```

- `/etc/nginx/conf.d/example_2.conf`

```bash
# /etc/nginx/conf.d/example_2.conf
server {
    listen       80;
    server_name  example_2.com;

	# ログの設定
    access_log  /var/log/example_2.com/access_log;
    error_log   /var/log/example_2.com/error_log;
		
	# ローカルIPで接続されたApplicationServerへの接続設定
    location / {
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_pass  http://192.168.0.2:8081;
    }
}
```

### 2. ApplicationServerのバーチャルホスト設定

WebServerからの接続のみ受け付けるためfirewallで制限する

```bash
firewall-cmd --add-rich-rule='rule family=ipv4 source address="192.168.0.1" port port=8080 protocol=tcp accept' --zone=public --permanent
firewall-cmd --add-rich-rule='rule family=ipv4 source address="192.168.0.1" port port=8081 protocol=tcp accept' --zone=public --permanent

firewall-cmd --reload

firewall-cmd --list-rich-rules
rule family="ipv4" source address="192.168.0.1" port port="8080" protocol="tcp" accept
rule family="ipv4" source address="192.168.0.1" port port="8081" protocol="tcp" accept
```

受け付けるポートの設定を修正する

```bash
# /etc/httpd/conf/httpd.con

#Listen 80
Listen 8080
Listen 8081
```

バーチャルホストを設定するため`/etc/httpd/conf.d/vhost.conf`を作成する

```bash
# /etc/httpd/conf.d/vhost.conf
<VirtualHost *:8080>
  DocumentRoot /var/www/html/example_1
  ServerName example_1.com
</VirtualHost>

<VirtualHost *:8081>
  DocumentRoot /var/www/html/example_2
  ServerName example_2.com
</VirtualHost>
```

WebServerから受け付けた通信ログのIPが、ローカルIPのアドレスになるためWebServerが受け付けたIPを取得できるよう修正する

```bash
# /etc/httpd/conf/httpd.con
~~~

<IfModule remoteip_module>
    RemoteIPHeader X-Forwarded-For
    RemoteIPInternalProxy 192.168.0.1
</IfModule>

~~~

<IfModule log_config_module>
    #
    # The following directives define some format nicknames for use with
    # a CustomLog directive (see below).
    #
    LogFormat "%a %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
    LogFormat "%a %l %u %t \"%r\" %>s %b" common

    <IfModule logio_module>
      # You need to enable mod_logio.c to use %I and %O
      LogFormat "%a %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio
    </IfModule>

    CustomLog "logs/access_log" combined
</IfModule>
```

### 3. Webserverでの証明書設定

`example_2.com`のドメインにLet’s Encryptで取得した証明書を適用する

- `certbot`インストール

Let’s Encryptから証明書を取得するためにインストールする

```bash
yum install -y certbot
```

- バーチャルホストの設定修正

> certbotで証明書の設定を行う時、`<domain>/.well-known/acme-challenge/~`というディレクトリを参照しに行く
この時バーチャルホストの設定でアプリケーションサーバーにアクセスしてしまうと参照するディレクトリが存在しないためエラーとなってしまうため、`<domain>/.well-known/acme-challenge/~`にアクセスした場合はウェブサーバーのディレクトリを参照するよう設定する
> 

```bash
# /etc/nginx/conf.d/example_2.conf
server {
    listen       80;
    server_name  example_2.com;

    access_log  /var/log/example_2/access_log;
    error_log   /var/log/example_2/error_log;

    location ^~ /.well-known/acme-challenge/ {
        root    /var/www/example;
        index   index.html index.htm;
    }

    location / {
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_pass  http://192.168.0.2:8080;
    }
}
```

- certbotから証明書取得

```bash
certbot certonly --webroot -w /var/www/example -d example_2.com

# 証明書の取得に成功すると下記が表示される
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example_2/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example_2/privkey.pem
   Your certificate will expire on 2022-09-12. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
```

- 上記で作成された証明書とキーからSSLの設定と80ポートへの接続を443ポートにリダイレクトして常時SSL化する

```bash
# /etc/nginx/conf.d/example_2.conf
server {
    listen       80;
    server_name  example_2;

    location ^~ /.well-known/acme-challenge/ {
        root    /var/www/example;
        index   index.html index.htm;
    }

    location / {
        return https://example_2.com
    }
}

server {
    listen 443 ssl;
    ssl_certificate      /etc/letsencrypt/live/example_2.com/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/example_2.com/privkey.pem;
    server_name example_2;

    access_log  /var/log/example_2/access_log;
    error_log   /var/log/example_2/error_log;

    location / {
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_pass  http://192.168.0.2:8080;
    }
}
```

### 4. 証明書の更新

下記コマンドで自動的に更新可能

```bash
certbot renew
```

また、下記コマンドで更新を試すことができる（実際には更新しない）

```bash
certbot renew --dry-run
```

Let’s Encryptの証明書の期限が90日なので、定期的に更新する運用が必要になる
cronで自動で更新するよう設定する

例）毎月1日の3:00に証明書を更新し、nginxを再起動する

```bash
crontab -e

00 03 1 * * certbot renew 2>&1 > ~/certbot_renew_result && systemctl restart nginx
```

上記のように更新結果を`certbot_renew_result`ファイルに出力した場合下記のように出力される

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/example_2.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Simulating renewal of an existing certificate for example_2.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
new certificate deployed without reload, fullchain is
/etc/letsencrypt/live/example_2.com/fullchain.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/example_2.com/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

> ※上記について実際運用する場合、更新結果の通知方法やnginxの再起動のスケジュールや再起動の成功、失敗の通知についてのスクリプト等用意したほうがいいかもしれない
