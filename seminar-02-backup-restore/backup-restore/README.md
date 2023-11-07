# バックアップ・リストア
今回はVM上のGrafanaをバックアップし、別のVMにリストアするデモを行う。
実際の運用でのバックアップリストアは、データが失われた場合の業務影響が大きい場合、テスト環境または開発環境で正しく行われるかをテストすることが望ましい。

## 環境準備
環境準備として具体的に、バックアップ用、リストア用のVMそれぞれに対して以下の操作を行った。
1. **デモに利用するためのVMの作成**
2. **Grafanaのインストール**
3. **PostgreSQLのインストール**
4. **PostgreSQLでGrafana用のユーザーとDBの作成**

以下に、各操作の詳細な手順を示す。

### デモに利用するためのVMの作成
バックアップ・リストア対象のGrafanaの環境は、Ubuntu上に作成した。
構成は、一つのUbuntuの上に、PostgreSQLとGrafanaが建っていて、GrafanaがそのPostgreSQLを使用する構成である。

その構成のため、以下の条件を満たすマシンを用意した。
- バックアップ・リストア対象のマシン
  - CPU → 2コア
  - メモリ → 8GB
  - ストレージ → 16GiB
  - OS → Ubuntu 22.04

今回は、再現しやすいようにazure cliを使ってVMを作成した。
azure cliを使ったVMの操作手順は[こちらのページ](../azurevm-commands/README.md)に示す。
azure以外の方法でVMを建てても、以下の手順は利用可能。

### Grafanaのインストール
次に、作成したVMの両方に対して、Grafana、PostgreSQLをインストールする。
1. 前提条件のインストール
```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
```

2. GPGキーをインポートする
```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

3. 安定版リリースのリポジトリを追加する
```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

4. パッケージリストの更新を行う
```bash
sudo apt-get update
```

5. Grafanaをバージョンを指定してインストールする
エンタープライズ版のバージョン9をインストールする
（Enterprise版のイメージはOSS版の機能を全て内包していて、Enterprise版へのアップグレード機能を持つ）
```bash
sudo apt-get install grafana-enterprise=9.5.13
```

6. Grafanaを有効化し、起動する
```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

### postgreSQLのインストール
GrafanaのDBはデフォルトでは 組み込み型のDBであるsqlite3 を使用している。しかし、DBの要件によっては、DBを複数のクライアントで共有したい場合や処理を分散して行いたい場合があり、そのような場合はクライアントサーバー型のDBを使用する。

Grafanaで利用可能なクライアントサーバー型のDBはPostgreSQLとMySQLの2つである。
今回はバックアップの説明のために、GrafanaのDBとしてPostgreSQL(version 15)を使用する。

※今回は、Grafanaと同じVMにPostgreSQLを導入するが、性能のためにDBをPostgresSQLに変更する場合、別のVMに分けるケースが多いと考えられる。

PostgreSQLのインストールには、[UbuntuでのPostgreSQL公式のインストール手順](https://www.postgresql.org/download/linux/ubuntu/)をを参考にした。

PostgreSQLは、特定のバージョンであればUbuntuのデフォルトのパッケージに含まれているが、今回のような任意のバージョンや最新版のインストールを行うためには、リポジトリ設定、パッケージリストの更新を行う必要がある。

1. ファイルリポジトリ設定を作成する
```bash
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

2. リポジトリ署名キーをインポートする
```bash
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

3. パッケージリストの更新を行う
```bash
sudo apt-get update
```

4. PostgreSQL 15  をインストールする
```bash
sudo apt-get -y install postgresql-15
```

5. postgresql が起動していることを確認する
```bash
sudo systemctl status postgresql
```

### PostgreSQLでGrafana用のユーザーとDBの作成
Grafanaの初期設定と、PostgresSQLを使う設定を行う。postgresql関連のコマンドを利用するためには、postgresqlに対する操作権限を持つユーザーで操作を行う必要がある。

大まかな流れとしては、PostgreSQL標準コマンドでGrafana用のユーザーを作成し、そのユーザーでGrafana用のDBを作成する

こちらのブログを参考にした：[https://qiita.com/kkk777/items/dfb2ffb103a5f5ab2cdb#dbユーザ作成からテーブル作成まで](https://qiita.com/kkk777/items/dfb2ffb103a5f5ab2cdb#db%E3%83%A6%E3%83%BC%E3%82%B6%E4%BD%9C%E6%88%90%E3%81%8B%E3%82%89%E3%83%86%E3%83%BC%E3%83%96%E3%83%AB%E4%BD%9C%E6%88%90%E3%81%BE%E3%81%A7)

1. Linuxのユーザーをpostgresql用のユーザーであるpostgresに切り替える
```bash
sudo -i -u postgres
```

2. postgresqlに接続（postgresユーザーで入る）
```
psql
```

3. Grafana用のユーザーを作成する
```
CREATE USER grafanauser WITH PASSWORD 'password' CREATEDB CREATEROLE;
```

4. ユーザーの一覧を確認する
```
\du
```

5. 一旦、postgresqlとの接続を切断
```
\q
```

6. デフォルトで作成されたpostgresというDBを使ってgrafanauserでpostgresqlに接続
```
psql -h localhost -U grafanauser -d postgres
```
ちなみに、`psql -U grafanauser -d postgres;`とDBを指定せずにコマンドを実行すると、`psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "grafanauser"`というエラーが出る、DBの指定も必須。

7. grafana用のDBユーザーで、Grafana用のDBを作成する
```
CREATE DATABASE grafana_db;
```

8. データベースのリストを確認する
```
\l
```

9. postgresqlとの接続を切断
```
\q
```

10. ユーザーをもとに戻す
```
exit
```

## バックアップ対象のGrafanaに変更を加える
環境準備が完了したら、バックアップリストアが終わった後で、それが正しく行われたかを確認することができるように、バックアップ対象のGrafanaに以下の3つの変更を加える。
1. **設定ファイルの変更**
  バックアップ対象のGrafanaでPostgreSQLを利用するよう設定ファイルを更新する。    
2. **ダッシュボードの変更**
  Grafanaのダッシュボードやユーザーの情報はDBに保存される。今回は、DBに変更を加えるため、ダッシュボードを作成する。
3. **プラグインのインストール**
  Grafanaプラグインのバックアップについても、プラグインをインストールしておく。

以下に、各操作の手順を示す。

### GrafanaでPostgreSQLを使うように設定ファイルを変更する
GrafanaでPostgresSQLを使用する設定を行う。
1. Grafanaの設定ファイルを編集する
```bash
sudo vi /etc/grafana/grafana.ini
```
2. DBにpostgreSQLを使用する設定を行うため、ファイルの中で [database] のセクションを探し、以下のように設定する。
```
[database]
type = postgres
host = localhost
name = grafana_db
user = grafanauser
password = password
```
3.  Grafanaを再起動する
```
sudo systemctl restart grafana-server
```
ここまでの手順で、Grafanaが正常に起動すれば、Grafanaで使用するDBをpostgresqlに変更することが完了する。

### ダッシュボードに変更を加える
1. Grafanaのデフォルトのデータソースで、ランダムの時系列データが生成されるグラフのパネルを置いた「Test Dashboard」という名前のダッシュボードを作成しておく。
  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/24dea527-e29b-45f2-85fc-c23ea5e5b8f1/13fddc64-b88e-42b1-8b45-9414a9b7a61e/Untitled.png)
    
### プラグインをインストールする
パネル操作により、以下のプラグインをインストールする。
  - Apache EChartsプラグイン - パネルの表現を増やすために利用することができる

## Grafanaのバックアップ
ここまで、バックアップ対象とリストア対象のVMにともにGrafanとPostgreSQLのインストール、PostgreSQLのユーザーとDB作成を行った上で、バックアップ対象のGrafanaをPostgreSQLに接続し、いくつかの変更を加えた。
ここから、バックアップ対象のGrafanaに対してバックアップを行っていく。

Grafanaのバックアップ対象は、主に以下の2つである。
1. OS上のGrafana関連ファイル
  → 構成ファイル、プラグイン等を含む
2. Grafana用のDB
  → Grafanaのダッシュボードの情報や、ユーザーの情報を保管している

特に **1. の OS上のGrafana関連ファイル** に関しては、バックアップするべき範囲が設定やGrafanaの使用法等によって異なるが、これについては、[GrafanaのOS上のフォルダ構成説明ブログ]()を参照し、各自で判断を行って欲しい。

今回は、一般的な構成の場合にバックアップするべき情報を対象として、バックアップのデモを行う。
おおまかな手順は、3つのバックアップ対象 ファイル/フォルダ を、`/work` ディレクトリにまとめて保存する。以下のような手順である。
1. **Grafana構成ファイルのバックアップ**
2. **プラグインデータのバックアップ**
3. **データベースのバックアップ**
リストアの際にはそれらを圧縮して送信し、リストア用VM内で展開していく。

※`/work`のようなバックアップは実際はVMの外部に安全に保存しておくことが望ましいが、今回はデモの簡単化のために同VM内にバックアップを保存する。

ちなみに、Grafanaは基本的にダッシュボードに表示するデータを外部から取得する（Prometheus、PostgresSQL、influxDB、etc）が、それらのバックアップは今回のデモの対象範囲外である。

以下に、バックアップの詳細な手順を示す。

### Grafana構成ファイルのバックアップ
Grafanaを建てる際に変更した可能性があるGrafana構成ファイルをバックアップする。今回だと、`/etc/grafana/grafana.ini` が対象となる。他にも、例えばdatasourceのプロビジョニング設定を行っていた場合だと `/etc/grafana/povisioning/datasources` のバックアップが必要となる。

今回はファイルはコピー、フォルダはtarとかで固めて、リストア先へはscpコマンドとかでデータを転送する。

1. まず、バックアップ用のディレクトリ `/work` ディレクトリを作成し、権限を変更する。
```
sudo mkdir /work
sudo chmod 777 /work
sudo chown azureuser:azureuser /work
```

2. 構成ファイルをコピーし、 `/work` ディレクトリにバックアップを作成する
```
sudo cp /etc/grafana/grafana.ini /work/grafana.ini.bak
```

### プラグインデータのバックアップ
Grafanaを利用する際にインストールしたプラグインのフォルダをバックアップする。今回だと、`/var/lib/grafana/plugins` が対象となる。

1. プラグインフォルダをコピーし、、`/work` ディレクトリにバックアップを作成する

```
sudo cp -rp /var/lib/grafana/plugins /work/
```

### データベースのバックアップ
1. ユーザーをpostgresユーザーに変更する
```
sudo -i -u postgres
```

2. DBのバックアップを作成する（今回は `/home/azureuser` 直下で作成した）

```
pg_dump grafana_db > /work/grafana_db.bak
```

3. postgresユーザーをログアウトする

```
exit
```

※今回は、PostgreSQLを使った方法について紹介しているが、Grafanaはデフォルトでは組み込みDBであるsqlite3を使用している。その場合は、データはディスク上の単一ファイルに保存されるため、そのファイルをバックアップしておけばよい。その場合、データの整合性を確保するには、SQLiteデータベースをバックアップするためにGrafanaサービスをシャットダウンする。

## Grafanaのリストア手順
ここで、リストア先のVMは、本ドキュメントの[環境準備]()の方法で作成し、GrafanaとPostgreSQLのインストール、Grafana用のDBユーザーとDBの作成まで済んでいると想定している。
そのため、ここでは以下の操作を行うことで、リストアが完了する。
1. ファイルの転送と展開
2. DBの復元

以下、リストアの詳細な手順を示す。

### ファイルの転送と展開
1. バックアップ用のVMに、環境準備の[azureの操作]()で作成した秘密鍵を転送
  バックアップ用VMからリストア用VMへscpコマンドを使ってバックフォルダ```work```を転送するために準備しておく。
!!!!コマンドでやる方法を書く!!!!

2. バックアップ用VMの /work ディレクトリを tar.gz 形式に圧縮し、権限を変更する。
```
sudo tar zcvfp /work/work.tar.gz /work
sudo chmod 777 /work/work.tar.gz
```

3. リストア用のVMにも work ディレクトリを作成し、外部から書き込みができるように権限を変更しておく。
```
sudo mkdir /work
sudo chmod 777 /work
sudo chown azureuser:azureuser /work
```

4. バックアップ用VM側からscpコマンドを実行し、workディレクトリに固めたバックアップをまとめてリストア用VMに送信する
```
sudo scp -i ~/.ssh/rg-grafana-seminar-2_key  /work/work.tar.gz azureuser@20.169.141.155:/work/
```

5. リストア用VMでログインし、送信したバックアップファイルを権限を保持して解凍する
```
sudo tar zxvfp /work/work.tar.gz -C /
```

6. リストア先VMの設定ファイルを上書きする
```
sudo cp /work/grafana.ini.bak /etc/grafana/grafana.ini
```

7. リストア先VMのプラグインフォルダを上書きする
```
sudo cp -rp /work/plugins /var/lib/grafana/
```

### DBの復元
1. ユーザーをpostgresユーザーに変更する
```
sudo -i -u postgres
```

2. grafana_dbにダンプしたデータを復元する
```
	psql grafana_db < /work/grafana_db.bak
```

3. postgresユーザーをログアウトする
```
exit
```

4. Grafanaサーバーを再起動する
```
sudo systemctl restart grafana-server
```
ここまでの操作で、リストアが完了するはずである。

## リストアの確認
最後に、リストアが正しく行われたかどうかを確認する。
1. リストア用VMのGrafanaへアクセス
2. バックアップ用VMに加えたGrafanaへの以下の3つの変更が復元されていることを確認する。
  ①GrafanaがPostgreSQLのDBを使用する設定
  ②ダッシュボードの作成
  ③プラグインのインストール
  ※①と②については、作成したダッシュボードがリストア済のGrafanaで確認できれば、復元されていると確認できる。

ここまでできれば、バックアップリストアのデモは完了！