# chef-centos-setup

CentOS6.x のサーバー環境を構築するための Chef レシピ集

* 本番環境： さくらVPS CentOS6.7 64bit（標準OS）を想定
* テスト環境： Vagrant CentOS6.7 64bit on MacOSX

## 適用されるレシピ

|No|クックブック名|本番環境  (レシピ名)|テスト環境  （レシピ名）|備考|
|:---|:---|:---:|:---:|:---|
|1|sakuravps_tunig|default|default|不要なデーモン、パッケージの削除|
|2|users|default|default|Root 権限のある一般ユーザーの作成|
|3|yum|default|default|各種レポジトリなどの追加|
|4|apache2|default|default|Apache 2.2.15 のインストール|
|5|mysql56|default|default|Mysql 5.6.28 のインストール|
|6|php56|default|default|PHP 5.6.17 のインストール|
|7|tools|default|default|vim(+lua), git などのインストール|
|8|ssh|default||sshd の設定（root 接続禁止など）
|9|iptables|default|flush|iptables の設定|
|10|webapp-dev-env|default|default|Nodejs, npm, bower, Gulp, Composer, SASS のインストール|

（注意）テスト環境に ``SSH`` レシピ、または ``iptables`` レシピを適用すると SSH 接続ができなくなる

* 空欄はレシピの適用なし
* 本番環境の詳細は ``node/sakuraroot.json`` の ``run_list`` 参照
* テスト環境の詳細は ``node/centos.json`` の ``run_list`` 参照


## 事前準備

``users::default`` レシピでは Data bag を利用するため、

1. Data bag 暗号化のための秘密鍵の生成
2. ユーザーパスワードの生成

が必要

``ssh::default`` レシピでは 鍵認証のみの設定にしてあるため、

1. SSH 接続用の RSA 鍵ペアの生成

が必要

```bash
# data bag を暗号化する秘密鍵の生成
$ openssl rand -base64 512 > .chef/data_bag_key

# 環境変数の設定
$ export EDITOR=vim

# ユーザー（nobita）のパスワード生成
# PASSWORD の部分を実際のパスワードにすること
$ php -r 'echo crypt("PASSWORD","$6$".substr(uniqid(),0,8));' >> password.txt

# SSH 接続用の RSA 鍵ペアを生成する（パスフレーズはなし（空欄）でOK）
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/username/.ssh/id_rsa):  ← 作成される場所。問題なければEnter。
Enter passphrase (empty for no passphrase):  ← パスフレーズの入力。省略する場合はそのままEnter。
Enter same passphrase again:  ← パスフレーズの再入力。省略した場合はそのままEnter。

# ユーザー（nobita）の作成
$ knife data bag create --secret-file .chef/data_bag_key --local users nobita

# vim が起動したら、

{
	"id": "nobita",
	"shell": "/bin/bash",
	"password": "$6$5696018b$EPgCN6vj.jocDrBiax0HpIfAbI.24Dwov8K6ri45OAsiG1SxmFItFzlLEwp7eiwFjUvxDI0S/I/
	"sshkey": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDL+mG+DQcUGn2iwmtt13dBlyWbOk0d063uz6HrShDm3S+6g7WYR
	"wheel": true
}

# として、vim を終了すると ``data_bags/users/nobita.json`` が作成される
# password は上記で生成した暗号化されたパスワード文字列
# sshkey は上記で生成した ~/.ssh/id_rsa.pub の文字列をペースト
# root 権限（sudo, su-）を付与する場合は wheel に true を指定する
# ユーザーは複数作成することが可能
```

## テスト環境（Vagrant）での実行

```bash
# SSH 接続設定
$ vagrant ssh-config --host centos >> ~/.ssh/config

# インスタンス起動
$ vagrant up

# vagrant へ chef のインストール
$ knife solo prepare centos

# vagrant で chef 実行
$ knife solo cook centos

# iptables のリセット
$ ssh centos
$ sudo iptables --flush
```

## 本番環境（さくら VPS）での実行

```bash
# SSH 接続の設定
# iptables レシピ適用によって Root ユーザーの接続禁止、SSH 接続ポートの変更が行われるため、レシピ適用前と適用後それぞれに接続設定をする
$ vim ~/.ssh/config

# SSH レシピ適用前の接続用
Host sakuraroot
	HostName xxx.xxx.xxx.xxx
	Port 22
	User root

# SSH レシピ適用後の接続用
Host sakura
	HostName 160.16.229.167
	Port 10022
	User nobita
 
# さくらVPS へ chef のインストール
$ knife solo prepare sakuraroot

# さくらVPS で chef 実行
$ knife solo cook sakuraroot

# 接続確認
$ ssh sakura
```

## 4. サーバー構成後の確認

```bash
# Apache のバージョン確認
$ httpd -v

# MySQL のバージョン確認
$ mysql --version

# php のバージョン確認
$ php -v

# git のバージョン確認
$ git --version

# vim のバージョン確認
$ vim --version | egrep 'IMproved|Huge|lua'

# node のバージョン確認
$ node -v

# npm のバージョン確認
$ npm -v

# bower のバージョン確認
$ bower -v

# gulp のバージョン確認
$ node -v

# composer のバージョン確認
$ node --version

# sass のバージョン確認
$ sass -v

# iptables の設定内容確認
$ sudo iptables -L

# 待ち受けポートの確認
$ lsof -i | grep LISTEN
```

## 参考

### テスト環境構成

* サーバーOS CentOS 6.5 64bit   
	* （BOX）: [nrel/CentOS-6.7-x86_64](https://vagrantcloud.com/nrel/boxes/CentOS-6.7-x86_64)
* ホスト名： centos
* IP アドレス: 192.168.33.10
* httpd: #80（#8080 からフォワード）
* ユーザー名/パスワード： vagrant/vagrant
* Rootパスワード： vagrant
