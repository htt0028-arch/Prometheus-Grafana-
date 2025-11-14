# 🚀 Linuxサーバー構築と監視の基礎

この資料では、Webサーバー「**Nginx**（エンジンエックス）」の環境設定と、システム監視ツールである「**Prometheus**（プロメテウス）」と「**Grafana**（グラファナ）」のセットアップ手順を解説します。

Linuxの基本的なコマンド操作を学びながら、サーバーの「見える化」に挑戦しましょう！

-----

## Chapter 1: ユーザー作成と権限設定 🧑‍💻

サーバーを安全に運用するために、サービス専用のユーザーを作成します。

### 1\. サービス専用ユーザーの作成

#### コマンド

```bash
useradd -rs /bin/false nodeusr
```

| オプション | 意味 | 補足 (なぜ必要？) |
| :--- | :--- | :--- |
| **`useradd`** | 新しいユーザーを作成するコマンド。 | |
| **`-r`** | **システムユーザー**として作成します。 | ログインして使う通常のユーザーではなく、**サービスやデーモン**（裏側で動くプログラム）専用のユーザーであることを示します。UID（ユーザーID）が100未満になることが多いです。 |
| **`-s /bin/false`** | ログインシェルに **`/bin/false`** を指定します。 | このシェルは「何もしないで即終了する」ため、**このユーザーでサーバーにログインすることができなくなります**。サービス専用ユーザーのセキュリティを高めるための設定です。 |
| **`nodeusr`** | 作成するユーザー名。 | 今回は監視ツール `Node Exporter` 用のユーザーです。 |

-----

#### コマンド

```bash
useradd -r -d /var/www/myproject -s /bin/false webusr
```

| オプション | 意味 | 補足 (なぜ必要？) |
| :--- | :--- | :--- |
| **`-d /var/www/myproject`** | **ホームディレクトリ**を `/var/www/myproject` に設定します。 | 通常は `/home/webusr` になりますが、ここではWebサイトのファイルを置く場所を直接ホームディレクトリとして指定しています。 |
| **`webusr`** | 作成するユーザー名。 | Webサーバー（Nginx）でWebコンテンツを扱うための専用ユーザーです。 |

### 2\. Webコンテンツ用ディレクトリの作成

#### コマンド

```bash
mkdir -p /var/www/myproject
```

| コマンド | 意味 | 補足 |
| :--- | :--- | :--- |
| **`mkdir`** | ディレクトリ（フォルダ）を作成するコマンド。 | |
| **`-p`** | **途中のディレクトリが存在しなくても自動で作成**します。 | 今回は `/var/www` がなくても、`/var`、`/var/www`、`/var/www/myproject` をすべて作ってくれます。 |
| **`/var/www/myproject`** | 作成したいディレクトリの完全なパス。 | **`/var/www`** はWebサーバーが公開するWebページやアプリのファイルを置くための標準的な場所です。 |

### 3\. 所有権の変更

作成したディレクトリを `webusr` が使えるように所有権を変更します。

#### コマンド

```bash
chown -R webusr:webusr /var/www/myproject
```

| コマンド | 意味 | 補足 (所有者とグループ) |
| :--- | :--- | :--- |
| **`chown`** | ファイルやディレクトリの**所有ユーザー**と**所有グループ**を変更するコマンド。 | `change owner` の略。 |
| **`-R`** | **再帰的**（Recursive）に変更を適用します。 | 指定したディレクトリ（`/var/www/myproject`）の中にあるすべてのサブフォルダやファイルにも、まとめて変更を適用します。 |
| **`webusr:webusr`** | **所有ユーザー**を `webusr`、**所有グループ**も `webusr` に設定します。 | |
| **`/var/www/myproject`** | 所有権を変更したいディレクトリのパス。 | |

-----

## Chapter 2: Nginxインストールと仮想ホスト設定 🌐

Webサーバーソフトウェアである **Nginx** をインストールし、Webサイトを公開するための設定（仮想ホスト設定）を行います。

### 1\. Nginxのインストールと起動

#### パッケージリストの更新

```bash
sudo apt update
```

  * **`apt`**: LinuxのDebian/Ubuntu系OSで使う**パッケージ管理コマンド**。ソフトウェアのインストールや更新、削除を行います。
  * **`update`**: インストールできる**パッケージのリストを最新の状態に更新**します。

#### Nginxのインストール

```bash
sudo apt install nginx -y
```

  * **`install nginx`**: Nginxというソフトウェアをインストールします。
  * **`-y`**: インストール時の「本当に実行しますか？(Yes/No)」の質問に自動で「**Yes**」と答えます。

#### Nginxサービスの有効化と起動

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

  * **`systemctl`**: OSのシステム管理ツールである **systemd** を操作するコマンド。
  * **`enable`**: **OS起動時にNginxサービスが自動で立ち上がる**ように設定します（自動起動の有効化）。
  * **`start`**: **今すぐ** Nginxサービスを起動します。

### 2\. Webページの作成（サンプル）

ルート権限が必要な場所にファイルを作成するテクニックです。

#### コマンド

```bash
echo "<h1>Hello from my Linux Server\!</h1>" | sudo tee /var/www/myproject/index.html
```

| コマンド/記号 | 意味 | 補足 (なぜ `sudo tee` を使う？) |
| :--- | :--- | :--- |
| **`echo "..."`** | 文字列を標準出力（画面など）に出力します。 | `<h1>` はHTMLで一番大きな見出しを表すタグです。**`\!`** は、コマンド履歴の展開を防ぐための記号です（`!` はシェルで特殊な意味を持つため）。 |
| **`|`** | **パイプ**（Vertical Bar）。前のコマンドの**出力**を、次のコマンドの**入力**に渡します。 | |
| **`sudo tee ...`** | 標準入力を受け取り、それを**ファイルに書き出す**コマンド。 | ファイルを置く `/var/www/myproject` は **root権限** でしか書き込めません。**`tee` コマンドに `sudo` をつける**ことで、ファイルへの書き込み部分だけを管理者権限で行えます。 |
| **`/var/www/myproject/index.html`** | WebサイトのトップページとなるHTMLファイルをこの場所に作成します。 | |

> **ポイント💡**: 通常のリダイレクト（`>`)を使うと、`echo`自体は一般ユーザーで実行されるため、書き込み権限がなく失敗します。`| sudo tee` とすることで、書き込み処理（`tee`）だけを管理者権限（`sudo`）で実行できます。

### 3\. 仮想ホストの設定と有効化

複数のWebサイトを1つのNginxで管理するために、サイトごとの設定ファイルを作成します。

#### 設定ファイルの作成

```bash
sudo nano /etc/nginx/sites-available/myproject
```

  * **`nano`**: テキストエディタを起動するコマンド。
  * **`/etc/nginx/sites-available/`**: Nginxの「**利用可能なサイト設定**」を置くディレクトリです。

#### 設定ファイル（`/etc/nginx/sites-available/myproject`）の例

`nano`でファイルを開いたら、以下の内容をコピー＆ペーストして保存しましょう。

```nginx
server {
    listen 80;
    server_name _;
    
    root /var/www/myproject;
    index index.html;
    
    access_log /var/log/nginx/myproject_access.log;
    error_log /var/log/nginx/myproject_error.log;
}
```

| ディレクティブ | 意味 | 補足 |
| :--- | :--- | :--- |
| **`server { ... }`** | **サーバブロック**。1つのWebサイトの設定をまとめる単位。 | |
| **`listen 80;`** | HTTP通信の標準ポートである **80番ポート** でリクエストを待ち受けます。 | |
| **`server_name _;`** | サイトのドメイン名。**`_`** は「**どのドメインからのアクセスでも受け入れる**」というワイルドカードのような意味です（テスト環境で便利）。 | |
| **`root /var/www/myproject;`** | **ドキュメントルート**。ブラウザからアクセスされたときに、Nginxがファイルを探し始めるベースディレクトリ。 | |
| **`index index.html;`** | ディレクトリにアクセスされたときに、デフォルトで返すファイル名（トップページ）。 | |
| **`access_log ...;`** | **アクセスログ**（誰がいつアクセスしたか）の保存場所。 | |
| **`error_log ...;`** | **エラーログ**（サーバーのエラーや設定ミス）の保存場所。 | |

#### 設定の有効化（シンボリックリンクの作成）

`sites-available`に置いただけでは有効になりません。Nginxが読み込む **`sites-enabled`** ディレクトリに**シンボリックリンク**を作成します。

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
```

  * **`ln`**: リンクを作成するコマンド（`link`の略）。
  * **`-s`**: **シンボリックリンク**（Symbolic Link）を作成します。これはWindowsでいう**ショートカット**のようなもので、元ファイル（実体）を参照します。
  * **`/etc/nginx/sites-available/myproject`**: **元ファイル**（実体）。
  * **`/etc/nginx/sites-enabled/`**: **リンク先**（ショートカットを置く場所）。
      * Nginxは `/etc/nginx/sites-enabled/` 配下のファイルを読み込むため、リンクを置くことで設定が「**有効化**」されます。元ファイルを編集すれば、リンク先の設定も自動で反映されます。

#### 設定ファイルのテストとNginxの再起動

```bash
nginx -t
sudo systemctl restart nginx
```

  * **`nginx -t`**: Nginxの設定ファイルを読み込み、**構文エラーやパスのエラーがないか**をテスト（`test`）します。
  * **`restart`**: Nginxサービスを再起動し、新しい設定を反映させます。

-----

## Chapter 3: Node Exporterのインストール 📊

サーバーのCPU使用率、メモリ、ディスクといった\*\*システムの状態（メトリクス）\*\*を取得するためのプログラム「**Node Exporter**」をインストールします。

### 1\. プログラムのダウンロードと展開

#### 作業ディレクトリへの移動

```bash
cd /opt
```

  * **`/opt`**: サーバーに手動でインストールしたサードパーティ製のアプリケーションなどを置く場所としてよく使われます。

#### Node Exporterのダウンロード

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
```

  * **`wget`**: 指定したURLからファイルをダウンロードするコマンド。
  * **`.tar.gz`**: 複数のファイルを1つにまとめた（**tar**）後、圧縮（**gz**）したアーカイブ形式のファイル。

#### ファイルの展開

```bash
tar xvf node_exporter-1.10.2.linux-amd64.tar.gz
```

  * **`tar`**: アーカイブファイルの作成・展開コマンド。
  * **`x`**: **展開**（e**x**tract）する。
  * **`v`**: **処理内容を表示**（**v**erbose）する。
  * **`f`**: **ファイル**（**f**ile）を指定する。
  * 展開後、`node_exporter-1.10.2.linux-amd64/` というディレクトリができます。

### 2\. 実行ファイルの移動と権限変更

#### 実行ファイル（バイナリ）の移動

```bash
mv node_exporter-1.10.2.linux-amd64/node_exporter /usr/local/bin/
```

  * **`mv`**: ファイルやディレクトリを**移動**（move）または**名前変更**するコマンド。
  * **`/usr/local/bin/`**: **PATHが通っている**（コマンドを実行するときに**ファイル名だけで場所を特定できる**）実行ファイルを置く場所です。ここに移動させることで、コマンドとして実行しやすくなります。

#### 所有権の変更

Node Exporterを専用ユーザー（`nodeusr`）で実行できるように設定します。

```bash
chown nodeusr:nodeusr /usr/local/bin/node_exporter
```

  * 実行ファイルの所有者と所有グループを、Chapter 1で作成した\*\*`nodeusr`**に変更します。これにより、サービスを**セキュリティの高い専用ユーザー\*\*で実行できるようになります。

-----

## Chapter 4: Node Exporterのsystemdサービス登録 ⚙️

Node Exporterをサーバー起動時に自動で立ち上がり、常に裏側で動く**サービス**として登録します。

### 1\. サービス定義ファイルの作成

`/etc/systemd/system/node_exporter.service` に以下の内容でファイルを作成します。

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

| セクション | ディレクティブ | 意味 | 補足 |
| :--- | :--- | :--- | :--- |
| **`[Unit]`** | **`After=network.target`** | **ネットワークの起動完了後**にこのサービスを起動する。 | ネットワーク経由でデータを提供するサービスのため必要です。 |
| **`[Service]`** | **`User=nodeusr`** | サービスを **`nodeusr`** という専用ユーザーで実行する。 | セキュリティを高めるために、管理者（root）以外の専用ユーザーで実行します。 |
| | **`ExecStart=...`** | 実際に実行する**コマンドのフルパス**。 | |
| | **`Restart=always`** | プロセスが予期せず終了（クラッシュ）した場合、**自動で再起動**する。 | |
| **`[Install]`** | **`WantedBy=multi-user.target`** | **通常のマルチユーザー環境**（サーバーが完全に立ち上がった状態）になったときに、このサービスを自動起動する。 | OS起動時に自動実行するための設定です。 |

### 2\. systemdへの反映とサービスの操作

#### systemdの設定読み込み

```bash
sudo systemctl daemon-reload
```

  * **`daemon-reload`**: systemdに、新しいサービス設定ファイル（`.service`）が追加されたことを教え、**設定を読み直させる**ためのコマンド。

#### サービスの有効化、起動、状態確認

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

  * **`enable`**: OS起動時の**自動起動を有効化**します。
  * **`start`**: **今すぐサービスを起動**します。
  * **`status`**: サービスの\*\*状態（稼働中か、エラーはないか）\*\*を表示します。

-----

## Chapter 5: Prometheusのインストールと設定 📈

**Prometheus** は、Node Exporterなどから送られてくる**サーバーのメトリクス（数値データ）を収集し、時系列データベースとして保存**するメインの監視システムです。

### 1\. Prometheusのインストール

手順はNode Exporterとほぼ同じです。

#### ダウンロード、展開、名前変更

```bash
cd /opt
wget https://github.com/prometheus/prometheus/releases/download/v2.55.1/prometheus-2.55.1.linux-amd64.tar.gz
tar xvf prometheus-2.55.1.linux-amd64.tar.gz
mv prometheus-2.55.1.linux-amd64 prometheus
```

  * ダウンロードしたディレクトリ名が長いので、`mv`で\*\*`prometheus`\*\*という短い名前に変更しています。

#### 設定ファイルの準備

```bash
cp -r prometheus/{consoles,console_libraries} /etc/prometheus/
mkdir -p /var/lib/prometheus
```

  * `cp -r`: 設定に必要なフォルダを `/etc/prometheus` に**再帰的にコピー**します。
  * `mkdir -p`: Prometheusが**収集した時系列データ**を保存するディレクトリ `/var/lib/prometheus` を作成します。

### 2\. Prometheusの設定ファイル（`prometheus.yml`）

`/etc/prometheus/prometheus.yml` に監視対象（ターゲット）を定義する設定ファイルを作成します。

```yaml
global:
  scrape_interval: 15s # データを取得する間隔を15秒に設定

scrape_configs:
  - job_name: 'prometheus' # Prometheus自身の状態を監視するジョブ
    static_configs:
      - targets: ['localhost:9090'] # Prometheusが自身のメトリクスを公開しているアドレス

  - job_name: 'node_exporter' # サーバーの状態（Node Exporter）を監視するジョブ
    static_configs:
      - targets: ['localhost:9100'] # Node Exporterがメトリクスを公開しているアドレス
```

  * **`scrape_interval: 15s`**: デフォルトで、**15秒ごと**に監視ターゲットからメトリクスを\*\*スクレイピング（取得）\*\*します。
  * **`job_name`**: 監視の単位（ジョブ）を定義します。
  * **`targets`**: **監視対象のアドレスとポート番号**を指定します。
      * **`localhost:9090`**: Prometheus自身のメトリクス用。
      * **`localhost:9100`**: Node Exporter（サーバーの状態）のメトリクス用。

### 3\. systemdサービス登録

Prometheusをサービスとして登録します。（Node Exporterと異なり、Prometheusは今回は**root**ユーザーで実行しています。）

#### サービス定義ファイル（`/etc/systemd/system/prometheus.service`）

```ini
[Unit]
Description=Prometheus Monitoring
After=network.target

[Service]
User=root # ★今回はrootで実行
ExecStart=/opt/prometheus/prometheus \ # ★実行ファイルを/opt/prometheus/prometheusに変更
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
Restart=always

[Install]
WantedBy=multi-user.target
```

  * **`ExecStart`** の行では、**Prometheusの実行コマンド**と、**各種設定ファイルの場所**をオプション（`--config.file`など）で指定しています。
　
 * --config.file：設定ファイル場所
　
 * --storage.tsdb.path：データ保存ディレクトリ
　
 * --web.console.templates：コンソールテンプレートの場所
　
 * --web.console.libraries：コンソールライブラリーの場所
　
 * Restart=always：プロセスを自動再起動


#### systemdへの反映とサービスの操作

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

-----

## Chapter 6: Grafanaのインストール 🎨

**Grafana** は、Prometheusが収集・保存したデータ（メトリクス）を読み込み、**グラフやダッシュボード**にして**分かりやすく表示**するためのツールです。

### 1\. Grafanaのリポジトリの追加

Grafanaの公式サイトからパッケージをインストールできるように、APT（パッケージ管理）の設定にGrafanaの情報を追加します。

#### 鍵のダウンロードと登録

以下の3つのコマンドで、パッケージの**署名検証用**のGPG公開鍵を取得し、APTが使える場所に保存します。

```bash
mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```

  * **`wget ... | gpg ... | sudo tee ...`**:
    1.  **`wget`** で鍵をダウンロード。
    2.  **`| gpg --dearmor`** で、ダウンロードした鍵をAPTで使える**バイナリ形式に変換**。
    3.  **`| sudo tee /etc/apt/keyrings/grafana.gpg`** で、変換した鍵ファイルを**所定の場所**に管理者権限で書き込みます。

#### リポジトリ情報の登録

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

  * このコマンドは、**「Grafanaのパッケージは、このURLから取ってきてね」** という情報をAPTに教える設定ファイル（`grafana.list`）を作成しています。
  * `[signed-by=...]` の部分で、先ほど登録した鍵で署名を検証することを指定しています。
  * echo：GrafanaのAPTリポジトリ情報を文字列として出力
  * deb [オプション] リポジトリURL ディストリビューション コンポーネント➡APTにどこからどのパッケージをとってきていいかを教える宣言
  * [オプション]：署名検証キー、アーキテクチャ制限などを指定（省略可）
  * リポジトリURL：パッケージを置いているサーバーのURL
  * ディストリビューション：OSのバージョンやリリース名(例：stable,focal,buster)
  * コンポ－ネント：パッケージの分類(例：main,contrib,non-free)

  *今回のコマンドに落とし込むと*
  * deb：バイナリパッケージを使う
  * [signed-by=/etc/apt/keyrings/grafana.gpg]：先ほど登録したGPGキーで署名を確認
  * sudo tee /etc/apt/sources.list.d/grafana.list：ファイルとして保存(APTのリポジトリ情報として登録)
  * https://apt.grafana.com：Frafana公式リポジトリ
  * stable：安定版リリース
  * main：主要パッケージのカテゴリ

### 2\. Grafanaのインストールと起動

#### パッケージリストの更新とインストール

```bash
sudo apt update
sudo apt install grafana -y
```

  * 新しいリポジトリを追加したので、もう一度 **`apt update`** でパッケージリストを更新します。
  * **`apt install grafana -y`** でGrafanaをインストールします。

#### サービスの有効化と起動

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

  * Grafanaのサービス名は **`grafana-server`** です。
  * `enable` で自動起動を有効にし、`start` で起動します。


これで、Webサーバー、監視データ収集、データ可視化の主要なツールがすべてインストール・設定されました！

