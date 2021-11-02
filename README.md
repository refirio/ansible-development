# Ansible Playbook: development

## 概要

コーダーさんのローカル開発環境を作るためのPlaybook<br>
環境の構築や調整はプログラマが行う想定

細かな挙動の差異があっても、そもそもコーダーさん用なので問題にならないはず<br>
`php72` や `composer1` といった見慣れないコマンドを実行することがあっても、手順として共有しておけば問題にならならはず<br>
という、ある程度割り切った環境<br>
特徴は以下のとおり

* Vagrant+CentOS7+Ansibleで Apache + PHP7 + DB の環境を作成する<br>原則「Apache + PHP7.4 + MariaDB」のための環境
* バーチャルホストで複数案件を管理する
* PHPはバーチャルホスト単位で7.0～7.4を切り替えられる<br>デフォルトは7.4とする
* MySQLはMariaDBで代用する
* PostgreSQLは一緒にインストールしておくので、必要なら使える
* Composerは一緒にインストールしておくので、必要なら使える<br>バージョン1も `composer1` という名前でインストールしておくので、必要なら使える
* LaravelはApacheで稼働させる（仮に本番環境がnginxだとしても、この環境はApacheとする）

## Vagrantを準備

VagrantでCentOS7を新規にインストールし、そこに開発環境を構築する<br>
作業フォルダは、ここでは `C:\Users\refirio\Vagrant\development` とする<br>
VagrantのIPアドレスは、ここでは `192.168.33.10` とする

### ボックスの追加

以下のコマンドで公式のCentOS7を追加

```
>vagrant box add centos/7
```

以下のコマンドで確認すると `centos/7 (virtualbox, 1905.1)` が追加されている

```
>vagrant box list
```

### Vagrantfile

```
Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.box_check_update = false
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.synced_folder "./code", "/var/www"
end
```

Vagrantfile と同階層に、同期用フォルダとして `code` を作成しておく<br>
Playbook は `code/ansible` に配置するものとする（つまりサーバ内の `/var/www/ansible` に配置される）

### 起動

```
>cd C:\Users\refirio\Vagrant\development
>vagrant up
```

### 終了する場合

```
>cd C:\Users\refirio\Vagrant\development
>vagrant halt
```

### 破棄する場合

```
>cd C:\Users\refirio\Vagrant\development
>vagrant destroy
```

### 初期起動時にエラーになった場合

Guest Additions を自動でアップデートしてくれるプラグインを導入する

```
>vagrant plugin install vagrant-vbguest
>vagrant halt
>vagrant up
```

解消されなければ、さらにサーバ内でカーネルのアップデートを行う

```
>vagrant ssh
$ sudo yum install -y kernel kernel-devel gcc
$ exit
>vagrant reload
```

## Ansibleで環境構築

### 初期hostsを設定

Vagrant に `dev.local` でアクセスできるようにする

`C:\Windows\System32\drivers\etc\hosts`

```
192.168.33.10   dev.local
```

`code\ansible\role-web\templates\vhosts.conf.j2` を作成し、以下の内容を記述しておく

```
<VirtualHost *:80>
    ServerName any
    DocumentRoot /var/www/main/html
</VirtualHost>
```

※`vhosts.conf.j2` は各々の環境（参加プロジェクト）に依存した内容になるので、PlaybookをGit管理する場合は管理対象外のファイルとしておく

### Ansibleをインストール

```bash
$ sudo su -
# yum -y install epel-release
# yum -y install ansible
```

### Ansibleを実行

Ansibleのhostsファイルの最後に設定を追加

```bash
# vi /etc/ansible/hosts
- - - - - - - - - - - - - - - - - - - - - - - - -
[localhost]
127.0.0.1
- - - - - - - - - - - - - - - - - - - - - - - - -
# exit
```

接続をテスト

```bash
$ ansible localhost -m ping --connection=local
```

Playbookの場所へ移動してから以下を実行

```bash
$ ansible-playbook site.yml --connection=local
```

ブラウザから以下にアクセスして確認<br>
http://dev.local/

### データベースと接続ユーザを作成（MariaDBの場合）

データベースと接続ユーザを作成

```bash
$ sudo mysql -u root
（パスワードなし）
mysql> GRANT ALL PRIVILEGES ON main.* TO webmaster@localhost IDENTIFIED BY '1234';
mysql> FLUSH PRIVILEGES;
mysql> CREATE DATABASE main DEFAULT CHARACTER SET utf8mb4;
mysql> QUIT;
```

### データベースへの接続をテスト（MariaDBの場合）

```bash
$ mysql -u webmaster -p
1234
mysql> QUIT;
```

以下でPHPからアクセスできる

```php
<?php

try {
    $pdo = new PDO(
        'mysql:dbname=main;host=localhost',
        'webmaster',
        '1234'
    );

    $stmt = $pdo->query('SELECT NOW() AS now;');
    $data = $stmt->fetch(PDO::FETCH_ASSOC);
    echo "<p>" . $data['now'] . "</p>\n";

    $pdo = null;
} catch (PDOException $e) {
    exit($e->getMessage());
}
```

### データベースと接続ユーザを作成（PostgreSQLの場合）

PostgreSQLの仕様上、同名のLinuxユーザも作成しておく必要がある

```bash
$ sudo su -
# useradd webmaster
# passwd webmaster
1234
```

引き続き、PostgreSQLのデータベースと接続ユーザを作成

```bash
# su - postgres

-bash-4.2$ createuser webmaster
-bash-4.2$ createdb main
-bash-4.2$ exit
```

### データベースへの接続をテスト（PostgreSQLの場合）

```bash
# su - webmaster

$ psql -l
$ psql main
main=> \q
```

以下でPHPからアクセスできる

```php
<?php

try {
    $pdo = new PDO(
        'pgsql:dbname=main;host=localhost',
        'webmaster',
        '1234'
    );

    $stmt = $pdo->query('SELECT NOW() AS now;');
    $data = $stmt->fetch(PDO::FETCH_ASSOC);
    echo "<p>" . $data['now'] . "</p>\n";

    $pdo = null;
} catch (PDOException $e) {
    exit($e->getMessage());
}
```

## Vagrantfileの調整

### Apacheを自動で起動させる

環境構築が完了した後は、Vagrantfile 内の最後にある `end` の前の行に、以下を追加しておく<br>
これが無いと、Vagrant起動時にApacheが自動で起動されないので注意

* 参考: [Vagrantのup時、httpdが自動起動しないとき - Qiita](https://qiita.com/ooba1192/items/96b7ab25d2bda1676aaa)

```
  config.vm.provision :shell, run: "always", :inline => <<-EOT
    sudo systemctl restart httpd
  EOT
```

## バーチャルホストの設定例

`test.dev.local` という領域を作成する例をもとに、バーチャルホストの設定方法を紹介する

* 原則 `/var/www/main` は使用せず、バーチャルホストで案件用の領域を作成する
* 設定例では dev.local のサブドメインとして設定しているが、これにこだわる必要は無い<br>「同じ環境なら同じドメイン」としておくと数が増えたときに管理しやすい可能性がある…という程度のもの
* Laravel用に作成する場合、公開ディレクトリは `html` ではなく `public` にしておく<br>関連するパス名も変更する

Vagrant に `dev.local` でアクセスできるようにする

C:\Windows\System32\drivers\etc\hosts

```
192.168.33.10   test.dev.local
```

`code\vhosts\test\html\index.php` を作成する
（つまり `/var/www/vhosts/test/html/index.php` を作成したことになる）

`code\ansible\role-web\templates\vhosts.conf.j2` に以下のように追記する

```
<VirtualHost *:80>
    ServerName test.dev.local
    DocumentRoot /var/www/vhosts/test/html
</VirtualHost>
```

PHPのバージョンを指定する場合、以下のように設定する（例はPHP7.2を指定する場合。PHP7.0～7.4を、ポート9070～9074に割り当てている）

```
<VirtualHost *:80>
    ServerName test.dev.local
    DocumentRoot /var/www/vhosts/test/html
    <FilesMatch \.php$>
        SetHandler "proxy:fcgi://127.0.0.1:9072"
    </FilesMatch>
</VirtualHost>
```

Playbookの場所へ移動してから以下を実行する

```bash
$ ansible-playbook site.yml --connection=local
```

必要に応じてデータベースを作成する

```
$ sudo mysql -u root
（パスワードなし）
mysql> GRANT ALL PRIVILEGES ON `test`.* TO webmaster@localhost IDENTIFIED BY '1234';
mysql> FLUSH PRIVILEGES;
mysql> CREATE DATABASE `test` DEFAULT CHARACTER SET utf8mb4;
mysql> QUIT;

$ mysql -u webmaster -p
1234
```

## コマンド実行例

### apacheユーザで作業する場合のコマンド

```bash
$ sudo su -s /bin/bash - apache
$ cd /var/www/vhosts/test
```

### Laravelを使用する場合のコマンド

```bash
$ sudo su -s /bin/bash - apache
$ cd /var/www/vhosts/test
$ composer install
$ php artisan migrate
$ php artisan db:seed
$ php artisan config:clear
$ php artisan view:clear
```

### PHP7.4を実行する場合のコマンド

```bash
$ php74 -v
PHP 7.4.24 (cli) (built: Sep 21 2021 11:23:11) ( NTS )
```

※PHP7.4をデフォルトとしているため、「$ php -v」でも実行できる

### PHP7.3を実行する場合のコマンド

```bash
$ php73 -v
PHP 7.3.31 (cli) (built: Sep 21 2021 10:24:03) ( NTS )
```

### PHP7.2を実行する場合のコマンド

```bash
$ php72 -v
PHP 7.2.34 (cli) (built: Aug 25 2021 15:54:25) ( NTS )
```

### PHP7.1を実行する場合のコマンド

```bash
$ php71 -v
PHP 7.1.33 (cli) (built: Aug 25 2021 16:39:50) ( NTS )
```

### PHP7.0を実行する場合のコマンド

```bash
$ php70 -v
PHP 7.0.33 (cli) (built: Aug 26 2021 08:10:14) ( NTS )
```

### php-fpmを再起動する場合のコマンド

PHPの設定ファイルは共通の内容を配置している<br>PHPバージョンごとに異なる設定内容にしたい場合、Playbookの調整が必要

php-fpmを個別に再起動する場合、一例だが以下のようにする

```bash
systemctl restart php73-php-fpm
systemctl restart php74-php-fpm
```

### Composerのバージョン2（インストール時点の最新版）を実行する場合のコマンド

```bash
$ composer
Composer version 2.1.9 2021-10-05 09:47:38
```

### Composerのバージョン1を実行する場合のコマンド

```bash
$ composer1
Composer version 1.10.15 2020-10-13 15:59:09
```

## 懸念点

案件数が増えると同期が重くなる可能性がある。ECCubeだと一つでも重いかもしれない<br>
vendorを同期対象から外したり対策できるか…と思ったが、何故か反映されなかった<br>
要調査
