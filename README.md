# 🚀 Linux監視＆Web環境構築（Prometheus + Node Exporter + Grafana + Nginx）

![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-DA4B2A?logo=prometheus\&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-Dashboard-F46800?logo=grafana\&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Server-333?logo=linux\&logoColor=white)
![Systemd](https://img.shields.io/badge/Systemd-Service-blue?logo=systemd\&logoColor=white)
![UFW](https://img.shields.io/badge/Security-UFW%2Ffail2ban-green)
![Nginx](https://img.shields.io/badge/Nginx-WebServer-009639?logo=nginx\&logoColor=white)

---

## 📋 プロジェクト概要

仮想Linux環境（Lubuntu）上に以下を構築：

* **Prometheus + Node Exporter + Grafana**：システム監視
* **Nginx**：Webサーバの構築・運用

CPU・メモリ・ディスクなどのメトリクスをリアルタイムで監視し、
Linux運用・監視・Web構築の基礎を実践。

---

## ⚙️ 環境情報

| 項目     | 内容                                                             |
| ------ | -------------------------------------------------------------- |
| OS     | Ubuntu / Lubuntu                                               |
| 仮想環境   | VirtualBox / WSL2                                              |
| 監視     | Prometheus / Node Exporter / Grafana                           |
| Web    | Nginx                                                          |
| サービス管理 | systemd                                                        |
| 使用ポート  | Prometheus:9090 / Node Exporter:9100 / Grafana:3000 / Nginx:80 |

---

## 🧩 構築手順（概要）

### 1️⃣ ユーザー作成

```bash
# 一般ユーザを追加（root直操作を避ける）
sudo apt update && sudo apt upgrade -y

# Node Exporter専用
sudo useradd -rs /bin/false nodeusr

# Prometheus ユーザー作成
sudo useradd --no-create-home --shell /bin/false prometheus

# Nginx用（任意）
sudo useradd -r -d /var/www/myproject -s /bin/false webusr
sudo mkdir -p /var/www/myproject
sudo chown -R webusr:webusr /var/www/myproject
```

---

### 2️⃣ Nginxインストール & 仮想ホスト設定

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx

# サンプルコンテンツ配置
echo "<h1>Hello from my Linux Server!</h1>" | sudo tee /var/www/myproject/index.html

# 仮想ホスト設定
sudo nano /etc/nginx/sites-available/myproject
```

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

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

### 2-1セキュリティ設定

#### UFW(Firewall)

```bash
sudo apt install ufw -y
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
````

#### Fail2Ban(不正ログイン)

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status
```

---

### 3️⃣ Node Exporter インストール

```bash
cd /opt
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
sudo tar xvf node_exporter-1.10.2.linux-amd64.tar.gz
sudo mv node_exporter-1.10.2.linux-amd64/node_exporter /usr/local/bin/
sudo chown nodeusr:nodeusr /usr/local/bin/node_exporter
```

#### systemdサービス登録
```bash
sudo nano /etc/systemd/system/node_exporter.service
```
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

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

✅ 動作確認

```bash
curl http://localhost:9100/metrics
```

---

### 4️⃣ Prometheus インストール

```bash
cd /opt
sudo wget https://github.com/prometheus/prometheus/releases/download/v2.55.1/prometheus-2.55.1.linux-amd64.tar.gz
sudo tar xvf prometheus-2.55.1.linux-amd64.tar.gz
sudo mv prometheus-2.55.1.linux-amd64 prometheus
mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r prometheus/{consoles,console_libraries} /etc/prometheus/
```

#### 設定ファイル `/etc/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

#### systemdサービス `/etc/systemd/system/prometheus.service`

```ini
[Unit]
Description=Prometheus Monitoring
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

✅ ブラウザ確認

```
http://localhost:9090
```

---

### 5️⃣ Grafana インストール

```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

---

### 6️⃣ Grafana 設定

* `http://<サーバIP>:3000` にアクセス
* データソース追加 → Prometheus（URL: `http://localhost:9090`）
* ダッシュボード ID **1860** を Import（Node Exporter Full）

## 📊 ダッシュボード設定

1. Grafana にログイン
2. 左メニュー → ⚙️ Data sources → Add data source
3. Prometheus を選択
4. URL に http://localhost:9090 を入力 → 「Save & test」  
　→ ✅ Successfully queried the Prometheus API. が表示されたらOK

5. 左メニュー ➕ Create → Import Dashboard  
　→ ID に 1860 を入力（Node Exporter Full）  
　→ 「Load」→ Data source に Prometheus を選択 → Import  
|監視項目|内容|
|---|---|
|CPU Usage|CPU使用率の推移|  
|Memory Usage|メモリ利用率|  
|Disk Spaceディスク容量の使用状況|  
|Network I/O|ネットワーク入出力量|  
⸻

🔍 動作確認（監視可視化）

Grafana の Dashboards → Node Exporter Full を開くと以下が確認できます。
---

## 📚 使用技術

* OS: Lubuntu 22.04 LTS
* 監視: Prometheus v2.55.1 / Node Exporter v1.10.2
* 可視化: Grafana OSS v11
* Web: Nginx
* サービス管理: systemd
* セキュリティ: ufw, fail2ban

---

## 🧩 理解しておくと良い Linux コマンド集（監視・運用向け）

この構築を通じて利用・理解しておくべき主要コマンドを整理しました。  
システム管理、ネットワーク確認、セキュリティ設定、ログ解析などの基本を押さえています。

---

### 🧠 サービス管理（systemd 関連）

| コマンド | 説明 |
|-----------|------|
| `systemctl status <サービス名>` | サービスの状態確認（active, failed など） |
| `systemctl start <サービス名>` | サービスの起動 |
| `systemctl stop <サービス名>` | サービスの停止 |
| `systemctl restart <サービス名>` | 再起動（設定変更時に使用） |
| `systemctl enable <サービス名>` | 自動起動の有効化（再起動後も起動） |
| `systemctl disable <サービス名>` | 自動起動の無効化 |
| `journalctl -u <サービス名>` | サービスのログ出力を確認 |
| `systemctl daemon-reload` | 新しいUnitファイルを読み込む（変更後に必須） |

> 🧩 例：`journalctl -u node_exporter -f`  
> リアルタイムで Node Exporter のログを追跡（トラブル解析に便利）

---

### 🌐 ネットワーク関連

| コマンド | 説明 |
|-----------|------|
| `ss -tulnp` | 現在リッスンしているポートを表示（旧 netstat） |
| `curl http://localhost:9100/metrics` | HTTP通信確認（Node Exporter動作確認） |
| `ping <ホスト名 or IP>` | 通信可否の確認 |
| `hostname -I` | 自サーバのIPアドレス確認 |
| `ufw status` | ファイアウォールの許可ルール確認 |
| `ufw allow 9090/tcp` | Prometheusポートを開放 |
| `ufw enable` | ufw（Firewall）有効化 |
| `ufw disable` | ufw無効化 |

> 💡 ufw（Uncomplicated Firewall）はUbuntu標準のシンプルなFirewall。  
> 外部アクセスを制御してPrometheusやGrafanaの公開範囲を管理できます。

---

### 🛡️ セキュリティ（fail2ban など）

| コマンド | 説明 |
|-----------|------|
| `sudo apt install fail2ban -y` | SSHブルートフォース対策ツールの導入 |
| `sudo systemctl enable fail2ban` | 自動起動設定 |
| `sudo fail2ban-client status` | 保護対象サービスと検知状況を確認 |
| `/etc/fail2ban/jail.local` | 設定ファイルの変更場所（例: SSH保護ポリシー） |

> 🔐 fail2ban は不正ログインを自動検知してIPをブロック。  
> サーバを公開運用する場合のセキュリティ基礎ツールです。

---

### 🧾 システム状態・監視

| コマンド | 説明 |
|-----------|------|
| `top` / `htop` | CPU・メモリ使用率のリアルタイム確認 |
| `free -h` | メモリ使用量の確認 |
| `df -h` | ディスク使用量確認 |
| `du -sh /var/log` | ログディレクトリの容量確認 |
| `ps aux | grep <プロセス名>` | 特定プロセスの状態確認 |
| `uptime` | 稼働時間とロードアベレージ確認 |

> 💡 Grafanaで可視化されるメトリクス（CPU、メモリ、ロード）は、  
> 実際にはこれらのコマンドが自動で数値化されたものです。

---

### 🧰 ファイル・パーミッション操作

| コマンド | 説明 |
|-----------|------|
| `ls -l` | 権限・所有者の確認 |
| `chmod +x <ファイル>` | 実行権限の付与（例: node_exporter実行ファイル） |
| `chown user:group <ファイル>` | 所有者の変更 |
| `nano <ファイル>` | テキスト編集 |
| `cat /etc/passwd` | システムユーザー一覧表示 |
| `useradd -rs /bin/false nodeusr` | ログイン不可ユーザーの作成（セキュリティ目的） |

---

### 🧹 トラブルシューティングの基本流れ

| ステップ | 目的 | コマンド例 |
|-----------|------|-------------|
| ① サービスの状態確認 | 起動しているか | `systemctl status node_exporter` |
| ② ログ確認 | エラー詳細を調べる | `journalctl -xeu node_exporter` |
| ③ ポート確認 | リッスン状態か | `ss -tulnp | grep 9100` |
| ④ 手動起動テスト | 実行可能か確認 | `/usr/local/bin/node_exporter` |
| ⑤ 設定修正 & 再読み込み | 設定変更反映 | `sudo systemctl daemon-reload && sudo systemctl restart node_exporter` |

### 🧩　トラブルシューティングの例

| 症状                                   | 原因                      | 対応                               |
| ------------------------------------ | ----------------------- | -------------------------------- |
| `Active: failed (exit-code)`         | ExecStartのパスミス or 権限不足  | 実行権確認・再設定                        |
| `Start request repeated too quickly` | 起動失敗を繰り返し               | `journalctl -xeu <サービス>`で詳細確認    |
| GrafanaでPrometheus見つからない             | URL設定誤り or Prometheus停止 | `systemctl status prometheus`で確認 |
| ダッシュボードが真っ白                          | node_exporter未起動        | `curl localhost:9100/metrics`で確認 |


---

## 🧭 まとめ

Linuxサーバ監視構築の過程で、以下のスキル領域を体系的に理解できました。

| 分野 | スキル要素 |
|------|-------------|
| サービス管理 | systemd, journalctl |
| 監視基盤 | Prometheus, Node Exporter, Grafana |
| ネットワーク | ポート通信, ufw設定 |
| セキュリティ | fail2ban, 非ログインユーザー作成 |
| 運用実務 | トラブル対応・ログ解析・永続化設定 |

---

## 🧭 今後の展開

* 複数サーバ監視への拡張
* Docker化 / IaC化
* Slack通知による自動アラート

---
