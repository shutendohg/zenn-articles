---
title: "あなたのSSHをOpen Quauntum Safe(非Hybrid)"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [security, 暗号化]
published: true
---

これは[FUJITSU Advent Calendar 2024 - Qiita](https://qiita.com/advent-calendar/2024/fujitsu)24日目の記事になります。

記事は全て個人の見解です。会社・組織を代表するものではありません。

## TL;DR
- OpenSSH9.0からはHybridな耐量子鍵交換アルゴリズムがデフォルトでサポートされている
- Open Quautum Safe ProjectのOpenSSHを使えばHybridではない耐量子鍵交換アルゴリズムのみを使うことも可能
- 上記をUbuntu 24.04.1 LTS上でトライ(localhostにSSH接続したのみで、デバイス間では試してない)

## OpenSSH9.xにおける鍵交換
このアドベントカレンダーを書こうと決めた当初はSSHも耐量子な鍵交換できるんですよ〜まあ本家OpenSSHではサポートされていないんですけど的な文脈で記事を書こうと思っていましたが、少し調べたところ、OpenSSH9.0からハイブリッド方式での鍵交換がデフォルトでサポートされているようです(X25519+Sreamlined NTRU Prime)。

[量子コンピュータ時代に向けた暗号通信 #pqc - Qiita](https://qiita.com/saitomst/items/bc8ee7820d044898d271#ssh)

また、以下の記事によるとOpenSSH 9.9では昨日NISTで標準化されたML-KEMを用いたハイブリッド鍵交換のサポートも開始されているようです。

[OpenSSH 9.9 new features: Enhanced security with post-quantum key exchange (mlkem768x25519-sha256) and DSA removal - 4sysops](https://4sysops.com/archives/openssh-99-new-features-enhanced-security-with-post-quantum-key-exchange-mlkem768x25519-sha256-and-dsa-removal/#rtoc-2)

## 耐量子鍵交換のみを使ってSSHする
そうなるとこれでこの記事は終わりになってしまうのですが、本家OpenSSHでサポートしているのはあくまでハイブリッド方式。硬派な(?)な方は「ハイブリッドなんて半端は嫌だ！俺はガチ(?)の耐量子鍵交換を使わせてもらう！」という方もいると思います。

そんなあなたにはOpen Quantum Safe(OQS) Projectで開発されているOpenSSHがあります。

[GitHub - open-quantum-safe/openssh: Fork of OpenSSH that includes prototype quantum-resistant key exchange and authentication in SSH based on liboqs. PROJECT INACTIVE. CONTRIBUTORS WANTED.](https://github.com/open-quantum-safe/openssh)

OQSプロジェクトについては手前味噌ですが以下の記事の章も参考にしてください。
[耐量子計算機暗号に対応したBoringSSLをお試しする](https://zenn.dev/shutendohg/articles/231220-advent-pqc#open-quantum-safe-project%E3%81%A8%E3%81%AF)

本題に戻りますが、ここではOQSで開発されているOpenSSHを用いてlocalhost向けにではありますがML-KEMとFalconを利用したSSH接続を実現します。

:::message alert
[open-quantum-safe/openssh](https://github.com/open-quantum-safe/openssh)はREADMEの

> WE DO NOT RECOMMEND RELYING ON THIS FORK TO PROTECT SENSITIVE DATA.

とあるように、実験的なもので実環境での利用は推奨されません。これを利用して生じた損害等には責任を負えませんのでご注意ください。
:::

### 環境と事前設定
以下、実行しているOSはUbuntu 24.04.1 LTSになります。

まず、必要なツールをインストールします。

``` sh
sudo apt update
sudo apt install -y autoconf automake cmake gcc libtool libssl-dev make ninja-build zlib1g-dev python3
```
ソースコードのダウンロード

``` sh
git clone -b OQS-v9 https://github.com/open-quantum-safe/openssh.git
``` 

次にliboqsのcloneとビルドを行います
``` sh
./oqs-scripts/clone_liboqs.sh
./oqs-scripts/build_liboqs.sh
``` 

最後にOQS-OpenSSHのビルドを行います
``` sh
env OPENSSL_SYS_DIR=/usr ./oqs-scripts/build_openssh.sh
```
OQS-OpenSSHが利用できるかのテストを実行。
``` sh
./oqs-test/run_tests.sh # 完了に数分かかります
```

### OQS-OpenSSHの設定
以下の内容でサーバー設定ファイルを作成します。
``` sh
cp regress/sshd_config oqs_sshd_config
```
viコマンドなどで以下を設定します(port 2222にしているのは他の設定とのコンフリクトを避けるため)
``` sh
Port 2222
KexAlgorithms ml-kem-512-sha256
HostKeyAlgorithms ssh-falcon512
PasswordAuthentication yes
PermitRootLogin no
```

- Port: サーバーがリッスンするポート番号。
- 	KexAlgorithms: 鍵交換アルゴリズム（ml-kem-sha256 を追加）。
- HostKeyAlgorithms: ホスト鍵アルゴリズム（ssh-falcon512 を追加）。
- PasswordAuthentication: パスワード認証を有効化。

そして以下のコマンドでサーバーを起動します。

``` sh
sudo $HOME/openssh/sshd -f $HOME/openssh/oqs_sshd_config -D -d
``` 
- -f: 設定ファイルを指定。
- -D: フォアグラウンド実行。
- -d: デバッグモード。

以下のような出力が得られれば準備完了です。
``` sh
Server listening on 0.0.0.0 port 2222.
debug1: Bind to port 2222 on ::.
Server listening on :: port 2222.
```

### localhostに向けてSSH接続
別ウィンドウを開き、opensshのディレクトリに移動して以下のコマンドでサーバーに接続します。

``` sh
$HOME/openssh/bin/ssh -vvv -p 2222 \ -o KexAlgorithms=ml-kem-sha256 \ -o HostKeyAlgorithms=ssh-falcon512 \ localhost
```
- -p: サーバーのポート番号（2222）。
- 	-o: KexAlgorithms: 鍵交換アルゴリズムを指定（ml-kem-sha256）。
- 	-o: HostKeyAlgorithms: ホスト鍵アルゴリズムを指定（ssh-falcon512）。
- 	-vvv: 詳細ログを表示。

パスワードを正しく入力し、いつものプロンプトが出てくれば成功です！```-vvv```によって詳細なログが出力されていて、以下2つが確認できれば耐量子鍵交換が利用されていると言えるでしょう。

``` sh
debug1: kex: algorithm: ml-kem-512-sha256 #鍵交換にML-KEMが利用されている
```
``` sh
debug1: kex: host key algorithm: ssh-falcon512 # ホスト鍵アルゴリズムにFalconが利用されている
```

## 終わりに
非ハイブリッド方式の耐量子鍵交換のトライ方法について述べました。今回はお手軽にlocalhostへの接続のトライでしたが、次回は異なるデバイス間で試してみたいですね。
