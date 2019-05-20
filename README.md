# tanks-management-screen
Codeigniterでゲームの管理画面を作成する

## VagrantでCentOS7環境を作成
https://qiita.com/sudachi808/items/3614fd90f9025973de4b

ログイン`vagrant ssh`

## 共有ファイル
- Vagrantfileを編集  
ホスト側のVagrantfileと同じ階層のdataファイルと、  
ゲスト側の/vagrant_dataが共有される  
```
Vagrant.configure("2") do |config|
 config.vm.box = "centos/7"
 config.vm.network "private_network", ip: "192.168.33.10"
 # ローカルマシンの Vagrantfile があるディレクトリ内の data フォルダと
 # 仮想マシン内の /vagrant_data ディレクトリを共有する
 # config.vm.synced_folder {host_path}, {guest_path}, option...
 config.vm.synced_folder "./data", "/var/www/html/"
end
```
- vagrant再起動 `vagrant reload`

## Apachインストール
- `sudo su`
- Apacheのインストール`yum -y install httpd`
- Apacheが自動で起動するように設定 `systemctl enable httpd.service`
- Apacheを起動 `systemctl start httpd.service`
- http用に80番ポートを開放 `firewall-cmd --add-service=http --permanent`

`FirewallD is not running`というエラーが出た  
->`systemctl status firewalld`で確認すると死んでる
```
# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
  Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
  Active: inactive (dead)
    Docs: man:firewalld(1)
```
->`yum updatte`  
->再起動`systemctl restart firewalld`
```
# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
  Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
  Active: active (running) since 月 2019-05-20 07:56:16 UTC; 2s ago
    Docs: man:firewalld(1)
Main PID: 26372 (firewalld)
  CGroup: /system.slice/firewalld.service
          └─26372 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid
```
- 設定の再読み込み `firewall-cmd --reload`

- ホストOSで適当なブラウザを開きhttp://192.168.33.10/にアクセスし，テストページが表示されればOK

- Apache用にユーザを追加する `useradd -s /sbin/nologin www`

- パスワードを設定 `passwd www`

- httpd.confを変更

* カレントディレクトリを変更し，オリジナルの設定ファイルをバックアップ  

`cd /etc/httpd/conf; cp -p httpd.conf httpd.conf.org`

* エディタで設定ファイルを編集 `vi httpd.conf`
```
User Apache
Group Apache
```
を
```
User www
Group www
```
に変更
** `Options Indexes FollowSymLinks`から`Indexes`を削除（これはしなくても良かった）
* `httpd -t`として`Syntax OK`と表示されれば設定ファイルを正しく書けている
- 確認用のページを作成する `vi /var/www/html/index.html`
```
 <html>
    <body>
    Test Page
    </body>
 </html>
```
`http://192.168.33.10/`で確認

## PHP 7.2インストール
* PHP7.2を入手するためのリポジトリを追加  
`yum -y install http://rpms.famillecollet.com/enterprise/remi-release-7.rpm`
* `yum list installed | grep php`として，古い(バージョン5.4)phpが残っていないことを確認
  - 残っていた場合は`yum remove php*`で削除
* PHP7.2のインストール  
`yum -y install --enablerepo=remi,remi-php72 php php-mbstring php-xml php-xmlrpc php-gd php-pdo php-pecl-mcrypt php-mysqlnd php-pecl-mysql`
  - `php -v`としてバージョン7.2のphpがインストールされたことを確認
* タイムゾーンの設定
  - 設定ファイルをバックアップ `cp -p /etc/php.ini /etc/php.ini.org`
  - `vi /etc/php.ini`として設定ファイルを開き，  
`;date.timezone =` を`;date.timezone = "Asia/Tokyo"`に変更
  - Apacheを再起動`systemctl restart httpd`
* テストページを作成
  - `vi /var/www/html/index.php`に`<?php phpinfo() ?>`と記述して保存
  - ホストOS側で`http://192.168.33.10/index.php`にアクセスしてPHPの情報が表示されればOK

## MySQLインストールと設定
* `yum remove mariadb-libs` でmariaDB関連を削除
* `yum list installed | grep mariadb`でなにもリストされなければOK
* MySQL公式リポジトリを追加  
`yum -y install http://rpms.famillecollet.com/enterprise/remi-release-7.rpm`
  - [公式](https://dev.mysql.com/downloads/repo/yum/)の中央下方にある  
Red Hat Enterprise Linux 7 / Oracle Linux 7 (Architecture Independent),   
RPM Packageの真下の括弧内に記述されたリポジトリが、2019年5月時点で  
`mysql80-community-release-el7-3.noarch.rpm`であるためこれを追加した。
＊ インストール `yum -y install mysql-community-server`
* 起動時にMySQLサーバも自動で起動するように設定 `systemctl enable mysqld.service`
* MySQLサーバ起動  `systemctl start mysqld.service`
* `/var/log/mysqld.log`に初期パスワードが記載されているのでgrepして調べる  
`grep password /var/log/mysqld.log`
* `mysql_secure_installation`を実行して初期設定
  - rootのパスワードを聞かれるので先ほどgrepして調べたものを入力
  - それ以降はすべてyesで解答
* `mysql -u root -p`を実行して先ほど設定したパスワードを入力してMySQLサーバにログイン
* `ALTER USER root@localhost IDENTIFIED WITH mysql_native_password BY 'YOUR_ROOT_PASSWORD'`  
を実行してパスワードの形式を変えておく
  - これをやっておかないとCodeIgniterからのアクセスでエラーが出る
* 動作確認のため，簡単なテーブルを作成
  - `CREATE DATABASE test_db` で新しいデータベースを作成
  - `use test_db` で使うデータベースを変更
  - `CREATE TABLE name(id INTEGER, name VARCHAR(100));` としてテーブルを作成
  - `INSERT INTO name VALUES (0, "Alice"), (1, "Bob"), (2, "Charlie") `としてテーブルnameに行を追加
  - `SELECT * FROM name`で追加されていることを確認
* `exit` で終了

## CodeIgniter動作確認
### 接続確認
- ダウンロードして中身を共有ファイルの中に入れる
- `cd /var/www/html`
- `vi application/controllers/Hello.php`
```
class Test extends CI_Controller {
              public function index()
              {
                      $this->load->database();
                      $query = $this->db->query('SELECT id, name FROM name');
                      foreach($query->result() as $row){
                              echo "{$row->id}, {$row->name}";
                              echo nl2br("\n");
                      }
              }
      }
```
- `http://192.168.33.10/index.php/hello`にアクセス
- Forbiddenになっちゃうので`sudo setenforce 0`

### MySQL確認
- DB接続設定ファイルを記述する
     `vi application/config/database.php`  として設定ファイルを開く．
     `$db['default'] = array(`行以降に各項目の設定が記述されているので，3項目を以下のように変更する
 
     - `'username' => 'root',`
     - `'password' => 'YOUR_ROOT_PASSWORD',`
     - `'database' => 'test_db',`

- `vi application/controllers/Test.php` として以下のコードを保存
```
class Test extends CI_Controller {
              public function index()
              {
                      $this->load->database();
                      $query = $this->db->query('SELECT id, name FROM name');
                      foreach($query->result() as $row){
                              echo "{$row->id}, {$row->name}";
                              echo nl2br("\n");
                      }
              }
      }
```
- `http://192.168.33.10/index.php/test`にアクセスして，以下のような内容が表示されればOK

## メモ
- index.phpを省く
https://qiita.com/saken649/items/a678836df65ebe24b606
