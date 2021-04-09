# バージョン一覧

|使用ツール|バージョン|
|---|---|
|PHP|7.3.27|
|Nginx|1.19.9|
|MySQL|5.7.33|
|Laravel|6.20.22|
|OS|Windows10|

# 環境構築の流れ
## ホストOSでのvagrant準備
- 下記コマンドを実行し、boxダウンロード(コマンド実行はどのディレクトでも問題ない)
```
vagrant box add centos/7
```
&emsp;&emsp;実行後、使用ソフト番号入力(virtual box使用時は '3')

- 作業用ディレクトリにvagrant_newディレクトリ作成,ディレクトリ初期化
~~~
mkdir vagrant_test
cd vagrant_new
vagrant init centos/7
~~~
- Vagrantfileを編集
~~~
変更点① config.vm.network "forwarded_port", guest: 80, host: 8080⇒コメントイン
変更点② config.vm.network "private_network", ip: "192.168.33.19"⇒コメントイン、IP編集
変更点③ config.vm.synced_folder "../data", "/vagrant_data" ⇒コメントイン、下記へ編集
config.vm.synced_folder "./", "/vagrant",type:"virtualbox"
~~~
- Vagrantプラグインインストール、Vagrant実行
~~~
vagrant plugin install vagrant-vbguest --plugin-version 0.21
vagrant up
~~~
- RloginでゲストOSへログイン
## ゲストOSでのツール準備
- 開発に必要なパッケージをインストール
~~~
sudo yum -y groupinstall "development tools"
~~~
- PHPインストール、バージョン確認の為、下記コマンド実行
~~~
sudo yum -y install epel-release wget
sudo wget http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
sudo rpm -Uvh remi-release-7.rpm
sudo yum -y install--enablerepo=remi-php**73** php php-pdo php-mysqlnd php-mbstring php-xml php-fpm php-common php-devel php-mysql unzip
php -v
~~~
- composerインストール、バージョン確認の為、下記実行
~~~
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer  #どのディレクトリでもcomposerコマンド利用する為file移動)
composer -v
~~~
## Laravelインストール
- vagrant_newディレクトリへ移動し、Laravel6.0インストール
~~~
composer create-project laravel/laravel --prefer-dist laravel_app 6.0
~~~
## データベースのインストール、データベース作成
- MySQLインストール
~~~
sudo wget https://dev.mysql.com/get/mysql57-community-release-el7-7.noarch.rpm
sudo rpm -Uvh mysql57-community-release-el7-7.noarch.rpm
sudo yum install -y mysql-community-server
mysql --version
~~~
- MySQL起動,PW変更
~~~
sudo systemctl start mysqld
sudo cat /var/log/mysqld.log | grep 'temporary password'
~~~
- 上記実行後、root@localhost:以下のPW確認
- 確認したPWにてMySQLへログイン
~~~
mysql -u root -p
mysql > set password = "新たなpassword";
~~~
- ファイルを開く
~~~
sudo vi /etc/my.cnf
~~~
- ファイル内追記
~~~
validate-password=OF
~~~
- データベース作成
~~~
sudo systemctl restart mysqld  #MySQL再起動
mysql > create database laravel_app;
sudo vi .env  #設定ファイルを開く
DB_PASSWORD= ⇒ DB_PASSWORD=登録したパスワード  # 左記編集
php artisan migrate  #マイグレーション  
~~~
## Nginxインストール、起動
- Nginx用ファイル作成
~~~
sudo vi /etc/yum.repos.d/nginx.repo #ファイル作成、以下書き込み

[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/mainline/centos/\$releasever/\$basearch/  
gpgcheck=0
enabled=1
~~~
- Nginxインストール
~~~
sudo yum install -y nginx
nginx -v
sudo systemctl start nginx  #Nginx起動
~~~
- ブラウザでhttp://192.168.33.19ヘアクセス ⇒ Nginxのページが表示される
- nginxの設定ファイル編集
~~~
sudo vi /etc/nginx/conf.d/default.conf

#以下内容編集
server {
    listen       80;
    server_name  192.168.33.19;
    root /vagrant/laravel_app/public; # 追記
    index  index.html index.htm index.php; # 追記

    #charset koi8-r;
    #access_log /var/log/nginx/host.access.log main;

location / {
    #root   /usr/share/nginx/html; # コメントアウト
    #index  index.html index.htm;  # コメントアウト
    try_files $uri $uri/ /index.php$is_args$args;  # 追記
}

#下記は root を除いたlocation { } までのコメント解除
location ~ \.php$ {
    #root          html;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME/$document_root/$fastcgi_script_name;
    include        fastcgi_params;
}
~~~
- php-fpm設定ファイル編集
~~~
sudo vi /etc/php-fpm.d/www.conf

#ファイル内、下記編集
user = apache ⇒ user = nginx
group = apache ⇒ group = nginx
sudo systemctl restart nginx # nginx再起動
sudo systemctl start php-fpm  #php-fpm起動 
~~~
- http://192.168.33.19 をブラウザで表示し、laravelが表示される事を確認
## Laravelログイン機能実装
- laravel/uiライブラリインストール
~~~
composer require laravel/ui:^1.0 --dev
sudo yum install nodejs npm  #Node.js、npmインストール
npm install && npm run dev  #フロントエンドに必要なパッケージをインストール、CSS/JSを作成

#以下、laravel_appの.envファイル編集
DB_DATABASE = laravel ⇒ DB=DATABASE = laravel_app
~~~
# 環境構築の所感
- プラグインやパッケージのバージョンが正しくないと起動できず、自分がどのバージョンをインストールしたのか、逐次確認するのが大事だと思った。
- Laravelがブラウザ表示できず、直前に行った設定ファイルの編集で誤りがあると思い、見直していた。しかしselinux設定ファイルの編集が必要だった。直前の変更に固執して解決に時間がかかってしまった。エラーの原因を速く見つけられるようにしたい。
- 用語やツール、コマンドが数多く出てきたので、きちんと整理して覚えたい。

# 参考サイト
[vagrantを導入しよう](https://qiita.com/ohuron/items/057b74a42b182b200ae6)
[VagrantのCentOS7のカーネルを更新(yum)](https://qiita.com/reflet/items/b1d9f169dfdad69c4d35)
[Laravel MySQL SQLSTATE[HY000] [1049] Unknown database が出た時に確認すること](https://qiita.com/miriwo/items/7c87f1053da413e739f7)
[Laravelで「Base table or view not found: 1146 Table」エラーが出るときの対処法](https://qiita.com/igz0/items/d14fdff610dccadb169e)
[Laravel6 ログイン機能を実装する](https://qiita.com/ucan-lab/items/bd0d6f6449602072cb87)
[Laravel 6.x 認証 - ReaDouble](https://readouble.com/laravel/6.x/ja/authentication.html)
[MarkdownでTable(表テーブル)を書く](https://notepm.jp/help/markdown-table)

