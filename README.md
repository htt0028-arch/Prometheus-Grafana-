# Prometheus-Grafana-
Linux監視環境構築（Prometheus + Grafana + Node Exporter）

# 🖥️ Linux監視環境構築（Prometheus + Grafana + Node Exporter）

## 📌 プロジェクト概要
このプロジェクトは、仮想環境（VirtualBox / WSL2）上の Ubuntu サーバに  
**Prometheus・Node Exporter・Grafana** を構築し、  
Linuxサーバのリソースを可視化・監視する環境を実装したものです。

バックエンドエンジニアとしてLinux運用や監視の基礎を理解する目的で作成しました。

---

## 🧰 使用技術
| 種類 | 使用ツール・技術 |
|------|----------------|
| OS | Ubuntu 22.04 LTS |
| 監視 | Prometheus / Node Exporter |
| 可視化 | Grafana |
| Webサーバ | Nginx（静的HTML確認用） |
| セキュリティ | UFW / Fail2ban |
| 管理 | systemctl（systemd） |

---

## ⚙️ 構成図
[Node Exporter] → [Prometheus] → [Grafana Dashboard]


---

## 🧱 セットアップ手順（概要）

### 1️⃣ Node Exporter の導入
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvf node_exporter-*.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
sudo useradd -rs /bin/false nodeusr
sudo tee /etc/systemd/system/node_exporter.service <<EOF
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

EOF
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
2️⃣ Prometheus の導入
wget https://github.com/prometheus/prometheus/releases/download/v2.55.1/prometheus-2.55.1.linux-amd64.tar.gz
tar xvf prometheus-*.tar.gz
sudo mv prometheus-*/prometheus /usr/local/bin/
sudo mv prometheus-*/promtool /usr/local/bin/
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r prometheus-*/{consoles,console_libraries} /etc/prometheus/

sudo tee /etc/prometheus/prometheus.yml <<EOF
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
EOF

sudo tee /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus
After=network.target

[Service]
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=:9090
Restart=always

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable prometheus
sudo systemctl start prometheus
3️⃣ Grafana の導入
sudo apt install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
ブラウザでアクセス： ➡ http://localhost:3000 ログイン：admin / admin（初回はパスワード変更）

📊 Grafana ダッシュボード設定
Prometheus をデータソースとして追加

URL: http://localhost:9090
ダッシュボードのインポート

Node Exporter 用公式ID: 1860
🔐 セキュリティ設定（任意）
# ファイアウォール有効化
sudo ufw enable
sudo ufw allow 22/tcp
sudo ufw allow 3000/tcp
sudo ufw allow 9090/tcp

# SSHブルートフォース対策
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
🧩 学んだこと
systemctlによるサービス管理
ufw/fail2banを使った基本的なサーバ防御
Prometheusの設定構成（scrape targetの概念）
Grafanaダッシュボードでの監視可視化
🧑‍💻 作者
htt0028-archi AWS CLF保有 / SAA学習中 Linux運用・監視の基礎を独学中


---

## 🧠 Part ②：Linux運用で理解しておくべきコマンド解説集


| コマンド | 用途 | 具体例 | 説明 |
|-----------|------|--------|------|
| `systemctl` | サービス管理 | `sudo systemctl status nginx` | サービスの起動・停止・自動起動設定などを制御する。systemdのフロントエンド。 |
| `journalctl` | ログ閲覧 | `sudo journalctl -u prometheus` | systemdが管理するログの閲覧。トラブルシュートで重要。 |
| `ufw` | ファイアウォール設定 | `sudo ufw allow 22/tcp` | Ubuntuの簡易ファイアウォール。ポート許可・拒否の設定を行う。 |
| `fail2ban` | 不正アクセス防御 | `sudo fail2ban-client status sshd` | ログを監視して、繰り返しログイン失敗するIPを自動BANする。 |
| `tar` | アーカイブ展開 | `tar xvf file.tar.gz` | ソースやバイナリ配布でよく使う圧縮ファイルの展開。 |
| `wget / curl` | ファイルダウンロード | `wget https://example.com/file` | ソフトウェアを直接ダウンロード。curlはパイプ接続にも使える。 |
| `chmod / chown` | 権限設定 | `sudo chown -R www-data:www-data /var/www` | 所有者や実行権限を調整。サービスが動かない時の原因になりやすい。 |
| `netstat / ss` | ポート確認 | `sudo ss -tuln` | どのポートが開いているか、サービスがリッスンしているか確認。 |
| `ps / top` | プロセス確認 | `ps aux | grep nginx` | 稼働中のプロセス確認・リソース監視。 |
| `df / du` | ディスク容量確認 | `df -h` | ディスク容量や使用率を確認。Prometheusの監視項目にも重要。 |

---
