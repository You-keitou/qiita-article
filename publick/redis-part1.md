# そうだ Redisを作ろう 🚀

こんにちは。今年の4月からWeb DeveloperとしてキャリアをスタートしたJinyangです！
記事を書こう、書こうと思いながら半年経ってしまいましたが、ここから継続的に頑張りたいと思っています。

さて、この記事の目標は **Redisを自作すること**！
業務でも使うRedisについて、ただ使うだけではなく、自作を通して理解を深めていきます。
エンジニアなら一度は車輪の再発明をしてみたいですよね？笑

それでは、やっていきましょう！

---

## 目次 📚

1. [Redisとは？](#redis)
2. [CodeCrafterで学ぼう](#codecrafterで学ぼう)
3. [準備：Ruby環境を整える](#準備ruby環境を整える)
4. [基本機能を実装してみる](#基本機能を実装してみる)
5. [学んだこと・まとめ](#学んだことまとめ)

---

## Redisとは？ 🤔 <a id="redis"></a>

Redisはオープンソースのインメモリデータベースで、高速なキー・バリューストアとして知られています。
キャッシュ、セッション管理、メッセージキューなど、さまざまな用途で使われています。

とまあ、その辺のDocumentから引っ張ってきたかのような説明文ですが、私は業務ではまだRedis Cliを使って何かコマンドを叩いたりしたことがないので、あんまりわかっていないのです。

最近、同期の方が会社でredisについて調査書を書いていたので、もっともっと深ぼっていきたいですね。

**TODO:** Redisについて勉強し、記事を書く〠

## CodeCrafterで学ぼう 🧑‍💻 <a id="codecrafterで学ぼう"></a>

さて、最初からredisを作るのはなかなか大変です。何かしら補助輪が欲しいと思うことでしょう。そこで、[CodeCrafter](https://app.codecrafters.io/catalog)というサービスを強くお勧めします。

CodeCrafterは、システムプログラミングを学ぶための素晴らしいプラットフォームです。
Redisコースを解くことで、徐々に自作Redisを完成させていきます。

また、さらなる知識を追い求めいたい方はぜひ[Working with ...](https://workingwithruby.com/)というオンラインebookもお勧めです。(私はまだ読んでいません😅)

---

## 準備：Ruby環境を整える 🛠 <a id="準備ruby環境を整える"></a>

業務でも使うRubyでRedisを自作していきます。以下の手順で環境を整えましょう。

1. Rubyのインストール
   - rbenvを使ってRubyをインストールします。バージョン管理をしてくれるので、複数のプロジェクトで異なるバージョンのRubyを使う場合に便利です。
   - [rbenvのインストール](https://qiita.com/kiharito/items/240911cc43bb9a1f4356)という記事が参考になるかと思います。
2. 必要なライブラリのインストール
   - [redis-cliだけをインストールする（EC2, Mac）](https://qiita.com/ajitama/items/ad37d9795e5cfdb840d2)という記事が参考になるかと思います。
   - 今回はredisのサーバーを立てるので、自作redisに接続しコマンドを実行するためのredis-cliをインストールします。

---

## 基本機能を実装してみる 💾 <a id="基本機能を実装してみる"></a>

では、CodeCrafterのコースに沿って、基本機能を実装していきます。

1. Portをbindする

CodeCrafterでデフォルトで提供されたコードを元に、以下のように実装します。

```ruby:server.rb
require "socket"

class YourRedisServer
  def initialize(port)
    @port = port
  end

  def start
    server = TCPServer.new(@port)
    client = server.accept
  end
end

YourRedisServer.new(6379).start
```

すでに、rubyではプロセス外部との通信を実現するためのSocketライブラリーが提供されています。その中にあるTCPServer classを使って、簡単にソケットを利用したサーバのプログラミングができます。また、acceptというinstanceメソッドを使って、クライアントからの接続を受け付け、TCPSocketのインスタンスを返します。また、これはブロッキングメソッドであり、クライアントからの接続があるまで待機します。


2. PINGコマンドに対しての応答を返す

redisサーバーがローカルで起動している場合、redis-cliを使って、`PING`コマンドを実行すると、`PONG`という文字列が返ってきます。これを実装してみましょう。

どうやら、clientからのデータを受け取るためには、TCPSocket#recvを使うようです。以下のように実装します。

```diff_ruby:server.rb
require "socket"

class YourRedisServer
  def initialize(port)
    @port = port
  end

  def start
    server = TCPServer.new(@port)
    client = server.accept
+   listen(client)
  end

+  private
+
+  def listen(client)
+    client.recv(1024).split("\r\n").each_with_index do |s, i|
+      client.sendmsg("+PONG\r\n") if s == 'PING'
+    end
+  end
end

YourRedisServer.new(6379).start

```

新たに、privateメソッドlistenを追加し、clientからのデータを受け取り、`PING`コマンドに対して`PONG`という文字列を返すようにしました。
redisでは、コマンドと引数はCRLF（\r\n）で区切られるため、splitメソッドを使って、それぞれのコマンドを取り出しています。あとは単純にコマンドが`PING`の場合に`PONG`を返すようにしています。

3. 並列接続をサポートする

次に、複数のクライアントからの接続をサポートするように実装していきます。
現在の実装では、一つのクライアントからの接続しか受け付けられません。acceptメソッドでクライアントからの接続があるまでずっと待機し、クライアントからの接続があると、socketを通じてデータのやり取りを行います。

```bash
$ redis-cli PING
$ redis-cli PING
```

このように、２行に分けて`PING`コマンドを実行すると、そもそも二つ目のコマンドが実行されるのはまた別のプロセスとなり、serverはこれを受信できません。なんとなく、acceptはブロッキングするからクライアントからの接続を常に待機するようにも感じますが、一度
`redis-cli PING`を実行すると、処理が進んでしまうため、serverはさらなるクライアントからの接続を受け付けることができません。

調べてみたところ、多数の実装方針がありました。rubyはマルチスレッドをサポートしているため、Threadを使って実装することができます。具体的に、server.acceptをloopで回し、クライアントからの接続があるたびにThreadを生成し、それぞれのThreadでlistenメソッドを実行することで、並列接続をサポートすることができます。

しかし、ただ単にThreadを作るだけでは少しつまらないので、本家のredisで実装されているevent loopを使って実装してみたいと思います。

週末くらいはここに追記したいとおもいます~~~~