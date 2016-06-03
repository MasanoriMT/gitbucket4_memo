gitbucket4_memo
===============

GitBucket 4.0を試験的にMacにインストールした際のメモ。  

# 背景

GitBucket 4.0では外部データベースとしてMySQLとPostgreSQLが使用可能になりました。

http://takezoe.hatenablog.com/entry/2016/05/01/004350

上記に記載されているように、従来のH2データベースには課題があるので、PostgreSQLを利用するようにセットアップします。

# GitBucketインストール

Homebrewで4.0がインストール可能でした。

    $ brew install gitbucket

launchctlで起動するようにします。  
ポート番号は、デフォルトでは8080になっていますので、環境に合わせて変更してください。

    $ cp /usr/local/opt/gitbucket/homebrew.mxcl.gitbucket.plist ~/Library/LaunchAgents
    （ポート番号を編集）
    $ launchctl load ~/Library/LaunchAgents/homebrew.mxcl.gitbucket.plist

この時点ですでにGitBucketは起動していますが、データベース設定を変更していきます。

# PostgreSQLインストール

こちらもHomebrewでインストールしました。
（9.5.3がインストールされました）

    $ brew install postgresql

同様にlaunchctlで起動するようにします。

    $ cp /usr/local/opt/postgresql/homebrew.mxcl.postgresql.plist ~/Library/LaunchAgents
    $ launchctl load ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist

# database.confを編集

https://github.com/gitbucket/gitbucket/wiki/External-database-configuration

上記に記載されているように、database.confを編集します。
database.confは、`${HOME}/.gitbucket`配下にあります。  
（gitbucketの実行ユーザのホームディレクトリ配下ということです。）

ここでは以下のように設定したとして以降の手順を進めます。

```
db {
  url = "jdbc:postgresql://localhost/gitbucket"
  user = "gitbucket"
  password = "gitbucket"
}
```

# データベース作成

以下のようにして、データベースおよびユーザを作成します。

```
$ psql -d postgres
psql (9.5.3)
Type "help" for help.

postgres=# create user gitbucket;
CREATE ROLE
postgres=# \password gitbucket
Enter new password: 
Enter it again: 
postgres=# create database gitbucket;
CREATE DATABASE
postgres=# \q
```

テーブルについては作成しておく必要はないようです。GitBucketを再起動すると勝手に作成してくれます。

    $ launchctl stop gitbucket
    $ launchctl start gitbucket

# java.net.UnknownHostException

Service HookでSlackと連携するように設定したところ、UnknownHostExceptionが発生しました。
GitBucketにプロキシ設定を認識させる必要がありました。

## VMパラメータ（JAVA_TOOL_OPTIONS）

VMパラメータでプロキシ指定ができるようです。  
homebrew.mxcl.gitbucket.plistにパラメータを追記します。

```
<string>-Dhttp.proxyHost=172.16.248.253</string>
<string>-Dhttp.proxyPort=8080</string>
<string>-Dhttps.proxyHost=172.16.248.253</string>
<string>-Dhttps.proxyPort=8080</string>
<string>-Dhttp.nonProxyHosts=172.16.*</string>
```
      
直接コンソールからGitBucketを起動するのであれば、JAVA_TOOL_OPTIONS環境変数を設定します。

    export "JAVA_TOOL_OPTIONS=-Dhttp.proxyHost=172.16.248.253 -Dhttp.proxyPort=8080 -Dhttps.proxyHost=172.16.248.253 -Dhttps.proxyPort=8080 -Dhttp.nonProxyHosts=172.16.*"

## WebHookService.scala

残念なことにVMパラメータを設定しても、その設定が有効にならないようでした。

```
（追記）  
以下の修正のプルリクエストを出したところマージされました。  
4.1ではこの問題は解消されていると思いますので、以下の手順は不要になるはずです。
```

http://qiita.com/uzresk/items/bc7c4a9dc764390cd5ce

GitBucketのソースコードを見る限り、HttpClientを利用しているようなのですが、useSystemPropertiesされていません。
そこで試しにコード修正してみることにしました。

```scala
val httpClient = HttpClientBuilder.create.useSystemProperties.addInterceptorLast(itcp).build
```

ビルドは、sbt.shを利用します。

    $ ./sbt.sh
    (中略)
    > compile
    [info] Updating {file:/Users/matoh/Desktop/gitbucket-master/}root...
    (中略)
    > packageBin
    [info] Packaging /Users/matoh/Desktop/gitbucket-master/target/scala-2.11/gitbucket_2.11-4.0.0.jar ...
    [info] Done packaging.
    [success] Total time: 1 s, completed 2016/05/25 16:42:46
    > exit

出来上がったjarで差し替えたgitbucket.warを作成します。

1. /usr/local/opt/gitbucket/libexec配下のgitbucket.warを展開（作業ディレクトリに展開）
1. 作成したjarをWEB-INF/lib配下にコピー（上書き）
1. 作業ディレクトリで以下のコマンドを実行
```
    $ jar cvfm ../gitbucket.war META-INF/MANIFEST.MF .
```

# GitHub Pull Request Builder

プルリクエストされたタイミングで検証用のJenkinsジョブを実行するために、このプラグインを利用してみました。
設定項目が多いこともあり、なかなかうまく動作しなかったのですが、GitBucketがGitHubのWeb APIを完全に網羅していないこともあって、一部機能しない点もありました。
少なくとも、ジョブ設定において、以下の項目をチェックしてしまうと、JenkinsからGitBucketへのAPI呼び出しが失敗して、思ったようにジョブが起動しなくなってしまうようでした。

Use github hooks for build triggering

プルリクエストがあってすぐにジョブが動きだすわけではなく、cronで定期的にJenkins側から問い合わせすることでビルドが走るようになります。

JenkinsからGitBucketへのリクエストは、Charlesで確認することができます。
Jenkinsのプロキシ設定をCharlesに向けることで、どのようなAPIを呼び出しているかをCharlesで検知します。
