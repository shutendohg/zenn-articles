---
title: "耐量子計算機暗号に対応したBoringSSLをお試しする"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [security, 暗号化]
published: true
---

これは[富士通アドベントカレンダー](https://qiita.com/advent-calendar/2023/fujitsu)17日目の記事になります。

## TL;DR
- 耐量子計算機暗号対応のBoringSSLがOQS Projectで開発されている
- この記事ではビルドとTLSデモ(+speedオプションの実施)を実際に試してみた
- ライブラリの詳説や各種性能比較/考察はしていなくて、簡単なOQS-BoringSSLの説明とデモをトライした的な内容になっています

## Open Quantum Safe projectとは
[Open Quantum Safe project](https://openquantumsafe.org/)(以下OQS)は現在NISTなどで検討されている耐量子計算機暗号(PQC)アルゴリズムを既存のライブラリに組み込むプロジェクトです。
[liboqs](https://github.com/open-quantum-safe/liboqs)というC言語のPQCライブラリをプロトタイプ開発し、OpenSSLや今回紹介するBoringSSLなどへのインテグレーションを行っています。 具体的な開発ライブラリは[github](https://github.com/open-quantum-safe)のページをご参照ください。ざっと見ると
- OpenSSL 1.1.1(※本家OpenSSL 1.1.1のサポート終了とOpenSSL 3への移行に伴いPQC対応のOpenSSL 1.1.1もサポートは終了した模様です)
- OpenSSL 3 Provider
- OpenSSH
- BoringSSL

などへの対応をしているようです。

上記ライブラリ以外にもcurlやWiresharkなどのアプリケーションへの対応も行っており、[こちら](https://github.com/open-quantum-safe/oqs-demos)にリストがあります。dockerなどで簡単に動作確認をすることもできます。

本記事ではliboqs対応されている[BoringSSL(OQS-BoringSSL)](https://github.com/open-quantum-safe/boringssl)の簡単なデモを実施するのが目的になります。ざっと調べたところ(少なくとも日本語では)OQS-OpenSSLをトライしている記事はチラホラみるものの、 BoringSSLに関しては意外とないな...と思ったのがこの記事を書いた動機になります。

## BoringSSLとは 
そもそもBoringSSLとは何か、とても平たく言うとOpenSSLのGoogleによるforkです。
forkの動機については[ImperialViolet](https://www.imperialviolet.org/2014/06/20/boringssl.html)に書かれています。もともとGoogleとしてもOpenSSLにコントリビュートしてきたものの、今後Google側の用途向けにメンテナンスしていくにはforkするのが最善という結論に至ったと読めます(2014年の話です)。
[Wikipedia](https://ja.wikipedia.org/wiki/OpenSSL#BoringSSL)情報で恐縮ですが、双方今後も協力していくという方針になっているようです。

そしてさらにPQCアルゴリズムに対応するためにOQS ProjectによってさらにforkされたBoringSSLが[OQS-BoringSSL](https://github.com/open-quantum-safe/boringssl)になります。
READMEによると以下のTLS 1.3をサポートしています。残念ながらハイブリッド証明アルゴリズムについては未サポートのようです。
> - quantum-safe key exchange to TLS 1.3
> - hybrid (quantum-safe + elliptic curve) key exchange to TLS 1.3
> - quantum-safe digital signatures to TLS 1.3

:::message alert
READMEには
WE DO NOT RECOMMEND RELYING ON THIS FORK IN A PRODUCTION ENVIRONMENT OR TO PROTECT ANY SENSITIVE DATA.
とあり、他のOQS Projectによってforkされたライブラリと同じく、まだ試験的な段階であるため, 本番環境での利用は推奨されていません
:::

## READMEのTLSデモの実行
では、実際にビルドして動かしていきたいと思います
### 実行環境
``` sh
$ uname -a
Linux ubuntu 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:29:11 UTC 2023 aarch64 aarch64 aarch64 GNU/Linux
```
M1 Mac(OSはmacOS Sonoma 14.1.2)で、UTM上に仮想マシンとして上記のubuntuを実行しています。メモリは4G, CPU数は4に設定しています。

### BoringSSLビルド
基本的にはREADMEに従います。`<BORINGSSL_DIR>=~/advent`として必要なものをinstallしていきます。
``` bash
# ~/上にいるとします
$ sudo apt install cmake gcc ninja-build libunwind-dev pkg-config python3 python3-psutil golang-go
$ mkdir advent && cd advent # adventディレクトリを作成
$ git clone --branch master https://github.com/open-quantum-safe/boringssl.git ~/advent
$ mkdir oqs && cd oqs # oqs ディレクトリをadvent内に作成, 移動
$ git clone --branch main --single-branch --depth 1 https://github.com/open-quantum-safe/liboqs.git
$ cd liboqs
$ mkdir build && cd build
$ cmake -G"Ninja" -DCMAKE_INSTALL_PREFIX=<BORINGSSL_DIR>/oqs -DOQS_USE_OPENSSL=OFF ..
$ ninja
$ ninja install
$ cd ~/advent # adventディレクトリに戻る
$ mkdir build # buildディレクトリをadvent内に作成
$ cd build
$ cmake -GNinja ..
$ ninja
```
基本的には大きな引っかかりはなくビルド完了できます。ディレクトリをあっちこっちいくのでご注意ください

最後に各種テストを実行して、正しく動作することを確認します。
``` sh
$ ninja run_tests # BoringSSLのホワイトボックステストとブラックボックステスト、およびOQS鍵交換とデジタル署名アルゴリズムのテストを実行
```

test結果
``` sh
ok  	boringssl.googlesource.com/boringssl/ssl/test/runner/hpke	(cached)
ok  	boringssl.googlesource.com/boringssl/util/ar	(cached)
ok  	boringssl.googlesource.com/boringssl/util/fipstools/acvp/acvptool/testmodulewrapper	(cached)
ok  	boringssl.googlesource.com/boringssl/util/fipstools/delocate	(cached)
crypto_test (for CPU "crypto")
crypto_test (for CPU "none")
crypto_test
crypto_test (for CPU "neon")
crypto_test --gtest_also_run_disabled_tests --gtest_filter=BNTest.DISABLED_WycheproofPrimality
crypto_test --gtest_also_run_disabled_tests --gtest_filter=RSATest.DISABLED_BlindingCacheConcurrency
crypto_test --gtest_also_run_disabled_tests --gtest_filter=RSATest.DISABLED_BlindingCacheConcurrency (for CPU "none")
crypto_test --gtest_also_run_disabled_tests --gtest_filter=RSATest.DISABLED_BlindingCacheConcurrency (for CPU "neon")
crypto_test --gtest_also_run_disabled_tests --gtest_filter=RSATest.DISABLED_BlindingCacheConcurrency (for CPU "crypto")
urandom_test
~~~~~省略~~~~~~
pki_test (for CPU "none")
pki_test (for CPU "neon")
pki_test (for CPU "crypto")
ssl_test (for CPU "crypto")
ssl_test (for CPU "neon")

All tests passed!
0/0/8599/8599/8599
PASS
ok  	boringssl.googlesource.com/boringssl/ssl/test/runner	39.280s
```
上記のような出力が出ればOKです。テスト完了には今回の仮想環境上では数分かかりました。

### TLSデモの実行
手っ取り早いデモは`python3 oqs_scripts/try_handshake.py`を実行することです。
``` sh
$ python3 oqs_scripts/try_handshake.py
---bssl server output---
Connected.
  Version: TLSv1.3
  Resumed session: no
  Cipher: TLS_AES_128_GCM_SHA256
  ECDHE group: p256_kyber512 # Kyber
  Secure renegotiation: yes
  Extended master secret: yes
  Next protocol negotiated:
  ALPN protocol:
  Early data: no
  Encrypted ClientHello: no

---bssl client output---
Connecting to 127.0.0.1:36299
Connected.
  Version: TLSv1.3
  Resumed session: no
  Cipher: TLS_AES_128_GCM_SHA256
  ECDHE group: p256_kyber512 # Kyber
  Signature algorithm: sphincsshake256ssimple #SPHINCS
  Secure renegotiation: yes
  Extended master secret: yes
  Next protocol negotiated:
  ALPN protocol:
  OCSP staple: no
  SCT list: no
  Early data: no
  Encrypted ClientHello: no
  Cert subject: C = US, O = BoringSSL
  Cert issuer: C = US, O = BoringSSL
```
ただ、このスクリプトではKEMとSIGのアルゴリズムがランダムに選択されてしまいます。アルゴリズムを自分で指定したい場合は`tool/bssl `を直接実行します。
* サーバー側
``` sh
tool/bssl server -accept 4433 -sig-alg <SIG> -loop #~advent/build上で実行
```
* クライアント側
```sh
tool/bssl client -curves <KEX> -connect localhost:4433
```
`<SIG>`, `<KEX>`はそれぞれ署名アルゴリズム、鍵交換アルゴリズムが入ります。[README](https://github.com/open-quantum-safe/boringssl?tab=readme-ov-file#supported-algorithms)にも記載されていますので、適宜入力してください。

ここでは例としてKEX `kyber1024`は, SIG `falcon1024` を選択します。
サーバー側は
``` sh
$ tool/bssl server -accept 4433 -sig-alg  falcon1024  -loop
# client側実行後に以下が出力されるため、初回実行時は待ち状態になります
Connected.
  Version: TLSv1.3
  Resumed session: no
  Cipher: TLS_AES_128_GCM_SHA256
  ECDHE group: kyber1024 # kyber
  Secure renegotiation: yes
  Extended master secret: yes
  Next protocol negotiated:
  ALPN protocol:
  Early data: no
  Encrypted ClientHello: no
```
次に別のターミナルを立ち上げ、クライアント側を実行します。

``` sh
$ tool/bssl client -curves kyber1024 -connect localhost:4433
Connecting to 127.0.0.1:4433
Connected.
  Version: TLSv1.3
  Resumed session: no
  Cipher: TLS_AES_128_GCM_SHA256
  ECDHE group: kyber1024 # kyber
  Signature algorithm: falcon1024 #falcon
  Secure renegotiation: yes
  Extended master secret: yes
  Next protocol negotiated:
  ALPN protocol:
  OCSP staple: no
  SCT list: no
  Early data: no
  Encrypted ClientHello: no
  Cert subject: C = US, O = BoringSSL
  Cert issuer: C = US, O = BoringSSL
```
以上でREADMEにあるデモを実行できました。

## speedオプション
`speed`オプションというものがあるらしいです。これで性能を測ることができます。
``` sh
$ tool/bssl 
Usage: tool/bssl COMMAND

Available commands:
    ciphers
    client
    isfips
    generate-ech
    generate-ed25519
    genrsa
    md5sum
    pkcs12
    rand
    s_client
    s_server
    server
    sha1sum
    sha224sum
    sha256sum
    sha384sum
    sha512sum
    sha512256sum
    sign
    speed
```

speedの引数は以下を取ることができます。
``` sh
tool/bssl speed  -
Unknown argument: -
-filter	A filter on the speed tests to run
-timeout	The number of seconds to run each test for (default is 1)
-chunks	A comma-separated list of input sizes to run tests at (default is 16,256,1350,8192,16384)
-json	If this flag is set, speed will print the output of each benchmark in JSON format as follows: "{"description": "descriptionOfOperation", "numCalls": 1234, "timeInMicroseconds": 1234567, "bytesPerCall": 1234}". When there is no information about the bytes per call for an  operation, the JSON field for bytesPerCall will be omitted.
-threads	The number of threads to benchmark in parallel (default is 1)
```

`speed`オプションでkyberを測定してみます。
``` sh
$ tool/bssl speed  -json -filter Kyber
[
{"description": "Kyber generate + decap", "numCalls": 1372, "microseconds": 1002253},
{"description": "Kyber parse + encap", "numCalls": 2270, "microseconds": 1007782}
]
```

## おわりに
BoringSSLにおける各アルゴリズムのベンチマークを手元の環境で取って比較してみる、気になる部分のコードにdeep diveするなどしたかったですが、今回はデモのお試しに終わってしまい、調べてみましたがいかかでしたか？的な記事になってしまいました。機会があればちゃんとした調査を行いたいですね。また、これをきっかけに耐量子計算機暗号に対応したライブラリについて興味をもっていただける方が出てくれば幸いです。

量子計算機が実際に既存暗号を破る未来はもう少し先な気もしますが、今から各種ライブラリの使い勝手や仕組みを把握しておくことで対応できるようにしておきたいですね。

## 参考リンク
* [Home | Open Quantum Safe](https://openquantumsafe.org/)
* [Open Quantum Safe · GitHub](https://github.com/open-quantum-safe)
* [GitHub - open-quantum-safe/boringssl: Fork of BoringSSL that includes prototype quantum-resistant key exchange and authentication in the TLS handshake based on liboqs](https://github.com/open-quantum-safe/boringssl)
* [GitHub - open-quantum-safe/liboqs: C library for prototyping and experimenting with quantum-resistant cryptography](https://github.com/open-quantum-safe/liboqs)
* [GitHub - open-quantum-safe/oqs-demos: Instructions for enabling the use of quantum-safe cryptography in assorted software using the OQS suite](https://github.com/open-quantum-safe/oqs-demos)
* [ImperialViolet - BoringSSL](https://www.imperialviolet.org/2014/06/20/boringssl.html)
* [OpenSSL - Wikipedia](https://ja.wikipedia.org/wiki/OpenSSL#BoringSSL)