---
title: "キャッシュDNSサーバ構築（Ubuntu 20.04）"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["DNS", "Bind", "Ubuntu"]
published: false
---

# はじめに

研究室のプロジェクトで内部向け DNS サーバを構築する必要があり，DNS とは何か勉強を行った上で，実際に構築まで行いました．本記事ではその際に学んだことを自分用のメモとして残しておりますが，誰かの役に立てば幸いです！

# DNS とは

大前提として，インターネットの世界では IP という仕組みを使って目的のホストまでパケットが届きます．しかし，私たちが Web サイトを閲覧する際はドメイン(例：`google.co.jp`)を入力するだけでページが閲覧できるようになりますよね．そこで登場するのが DNS です．DNS とは Domain Name System の略でドメインと IP アドレスを関連づけて管理するためのシステムです．簡単に言えば，ドメインと IP アドレスを紐付ける電話帳のようなものです．例えば，私たちはよく「ググる」と口にしますが，検索する際に入力する`google.co.jp`は私たちが覚えやすい形で表現されており，実際は`142.250.206.195`にアクセスしています．ブラウザで`142.250.206.195`と入力すると，google の検索画面に遷移することが確認できると思います．このように，私たちは DNS という仕組みによってアクセスしたい Web サイトの IP アドレスを覚えなくても，ドメインを入力すればサイトにアクセスできます．参考までに下記コマンドでドメインと IP アドレスを見ることができます．

```bash
$ nslookup google.co.jp
// 実行例
Server:		192.168.10.1
Address:	192.168.10.1#53

Non-authoritative answer:
Name:	google.co.jp
Address: 216.58.220.99
```

# ドメインの構造

ここまでで DNS の役割は大体理解できましたね．これからは DNS サーバでどのような処理が行われているのか理解するためにもドメインの構造について説明します．
例えば`kyushu-u.ac.jp`というドメインを考えてみましょう．`kyushu-u`は固有のドメインを示しています．また`ac`は「academic（学術的な）」という意味で大学や高専などの高等教育機関を，`jp`は「japan（日本）」を示しております．
他にも`com`は「commercial（商業組織用）」や`go`は「government（政府）」などがあります．
このドメインの構造が DNS での名前解決の流れに大きく関わってきます．
![](/images/9ebfefda4f70eb_image1.png)

# DNS の構造

DNS サーバには大きく分けて DNS キャッシュサーバ（リゾルバ）と権威 DNS サーバ（ネームサーバ）です．下図を見ながら説明していきます．

![](/images/9ebfefda4f70eb_image3.png)

１．まずクライアントがキャッシュサーバに問い合わせを行います．（ここでは`google.com`とします）
２．キャッシュサーバはルートサーバへ`google.com`の IP アドレスを問い合わせます．
３．ルートサーバから次のレベル(`com`)の IP アドレスに問い合わせるように回答します．
４．キャッシュサーバは`com`の DNS サーバに問い合わせます．
５．`com`の DNS サーバから次のレベル(`google.com`)の IP アドレスに問い合わせるように回答します．
６．キャッシュサーバは`google.com`の DNS サーバに問い合わせます．
７．` google.com``のDNSサーバはキャッシュサーバにIPアドレスを回答します． ８．キャッシュサーバはクライアントに `google.com`の IP アドレスを回答します．

もし，キャッシュサーバでキャッシュしている場合はその情報を再利用して名前解決を行うためルートサーバへの問い合わせは行いません．
このように，ドメインの階層化によって DNS の名前解決は`分散的`に行われていることがわかります．

# Bind で DNS キャッシュサーバ構築

ここまでで DNS とは何か説明してきました．まだまだ説明できていないところもありますが，早速サーバ構築してみようと思います．まずは bind をインストールしましょう．ターミナルを開いて下記コマンドを実行してください．

```bash
$ sudo apt update
$ sudo apt install bind9 bind9utils
```

bind がきちんとインストールできたか下記コマンドで確認してください．

```bash
$ systemctl status named

## 出力例
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-06-07 14:42:52 UTC; 2min 47s ago
       Docs: man:named(8)
    Process: 24733 ExecStart=/usr/sbin/named $OPTIONS (code=exited, status=0/SUCCESS)
   Main PID: 24734 (named)
      Tasks: 4 (limit: 4538)
     Memory: 7.3M
        CPU: 35ms
     CGroup: /system.slice/named.service
             mq24734 /usr/sbin/named -u bind
```

次に DNS サーバの設定ファイルを編集していきます．`/etc/bind/named.conf.options`を開く．

```bash
$ cd /etc/bind
$ sudo vim named.conf.options
```

```dns-zone-file
acl internal-network {
	"172.16.0.0/21";
};

options {
	directory "var/cache/bind";

	forwarder {
		8.8.8.8; // パブリックDNSサーバにforwarding
	};

	allow-query {
		localhost;
		internal-network; //アクセスを許可するネットワーク
	};

	recursion yes; //リゾルバとして設定

        dnssec-validation auto;

	listen-on-v6 { any; };
};
```

# ログの出力方法

私の場合 DNS のクエリログを蓄積する必要がありました．その設定も Bind 上で行うことができます．下記設定を先ほど記述した内容に追記します．

```dns-zone-file
logging {
        channel "queries-log" {
                file "/var/log/named/queries.log" versions 5 size 10M;
                severity debug;
                print-time yes;
                print-severity yes;
                print-category yes;
        };

        category queries { "queries-log"; };
};
```

ここで一度設定に間違いないか確認します．下記コマンドを実行して何にも表示されなければきちんと設定ができています．

```bash
$ named-checkcon
```

後は named.service を restart させてみてください．

```
$ sudo systemctl restart named
```

そしてきちんと動作しているか確認してください．

```
$ sudo systemctl status named
```

自分の場合はここで失敗しており，エラーログを確認すると，ログファイルを生成するディレクトリに権限がなかったようです，もしエラーが出た場合は`chmod`で権限を付与してください．
そして権限を付与したらもう一度 service を restart してみてください．

```
$ sudo systemctl restart named
$ sudo systemctl status named
```

active と表示されたら成功です！
それでは クライアント PC から DNS キャッシュサーバにアクセスしてみましょう．
その前に構築した DNS サーバの IP アドレスを確認(`ifconfig`or`ip address`)．私の場合は`172.16.4.103`だったので macbook で指定してみます．
環境設定のネットワークから WiFi(もしくわ有線)の詳細を選択すると下記画面が表示される．左下の`+`ボタンから`172.16.4.103`を入力し適用．
![](/images/9ebfefda4f70eb_image2.png)

そうするとクライアント PC から構築した DNS キャッシュサーバへ向けることができます．
念の為，macbook で`nslookup`を実行しておきます．

```bash
$ nslookup google.co.jp

// 実行例
Server:		172.16.4.103
Address:	172.16.4.103#53

Non-authoritative answer:
Name:	google.co.jp
Address: 172.217.161.227
```

きちんと`172.16.4.103`で名前解決できていることが確認できます．
これで Web サイトを閲覧できたりアプリを利用できます．またクエリログは DNS サーバの`/var/log/named/queries.log`に記録されていると思うので確認してみてください，

```bash
$ tail -f /var/log/named/queries.log
```

# まとめ

本記事では DNS とは何か説明した上で，実際に DNS キャッシュサーバの構築まで行いました．
面白そうと思った方は自宅に DNS サーバを構築してみてください！
