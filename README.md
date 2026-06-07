# noriki-me-ansible

このリポジトリは、Zabbix Server、Zabbix Proxy、Bastion サーバ（踏み台）、および WireGuard VPN を構築するための Ansible プレイブックとロールのセットです。

---

## 本レポジトリの構成

リポジトリは以下のように構成されています。

### 1. Playbooks (プレイブック)
リポジトリ直下には、各サーバーを構築するためのメインのプレイブックが配置されています。

* **[bastion.yml](file:///home/norik/Project/noriki-me-ansible/bastion.yml)**: Bastion (踏み台) サーバのセットアップ用プレイブック。
  * **呼び出すロール**: `linux_common`, `bastion`, `wireguard`, `zabbix_proxy`
* **[zabbix_server.yml](file:///home/norik/Project/noriki-me-ansible/zabbix_server.yml)**: Zabbix Server のセットアップ用プレイブック。
  * **呼び出すロール**: `linux_common`, `disable_firewalld`, `wireguard_client`, `zabbix_server`, `zabbix_server_https`

### 2. Roles (ロール)
`roles/` ディレクトリ配下に定義されている各ロールの用途は以下の通りです。

* **[linux_common](file:///home/norik/Project/noriki-me-ansible/roles/linux_common)**: Linux サーバの共通設定を行います。具体的には、ホスト名 (`common_hostname`) の設定を適用します。
* **[disable_firewalld](file:///home/norik/Project/noriki-me-ansible/roles/disable_firewalld)**: OS標準の `firewalld` サービスを停止し、自動起動を無効化します。
* **[bastion](file:///home/norik/Project/noriki-me-ansible/roles/bastion)**: Bastion サーバとして動作させるために、IPv4/IPv6 パケット転送 (forwarding) を有効化し、iptables による NAT (MASQUERADE) の設定を行います。
* **[wireguard](file:///home/norik/Project/noriki-me-ansible/roles/wireguard)**: WireGuard VPN サーバーの構築を行います。パッケージのインストール、パケット転送設定、サーバーおよび各クライアント鍵の生成、`wg0.conf` インターフェース設定、クライアント用接続設定ファイル（全トラフィック用/スプリットトンネル用）の自動生成、およびサービスの起動を行います。
* **[wireguard_client](file:///home/norik/Project/noriki-me-ansible/roles/wireguard_client)**: WireGuard クライアントとして動作させるためのパッケージインストールと、サービスの自動起動設定を行います。
* **[setup_https](file:///home/norik/Project/noriki-me-ansible/roles/setup_https)**: Let's Encrypt による HTTPS 化に必要なツール (Certbot, python3-certbot-dns-route53, mod_ssl) のインストール、Route53 連携用の AWS 認証情報の配置、および証明書の自動更新タイマーの有効化を行います。
* **[zabbix_proxy](file:///home/norik/Project/noriki-me-ansible/roles/zabbix_proxy)**: Zabbix Proxy (SQLite3版) および Zabbix Agent2 のインストールと設定を行います。Zabbix Server と安全に通信するための PSK 暗号化（TLS）も設定します。
* **[zabbix_server](file:///home/norik/Project/noriki-me-ansible/roles/zabbix_server)**: MIRACLE ZBX 7.0 (Zabbix Server) とそれに付随する MariaDB, Apache, PHP のインストール・構築を行います。データベースの初期化や基本構成ファイルの設定を行います。
* **[zabbix_server_https](file:///home/norik/Project/noriki-me-ansible/roles/zabbix_server_https)**: Route53 を用いた DNS 認証による Let's Encrypt 証明書の取得、Apache SSL の設定、および HTTP から HTTPS へのリダイレクト設定を行います (`setup_https` ロールに依存)。

---

## 変数の定義

本リポジトリでは、秘匿情報や環境固有のパラメータを以下の変数ファイルで管理します。
セキュリティ保護のため、実ファイルは `ansible-vault` による暗号化が行われているか、`.gitignore` で除外対象になっています。適宜 `.example` ファイルをコピーして作成してください。

### 1. グループ変数: `group_vars/all/vault.yml`
すべてのホストに適用される共通の秘匿情報やパラメータです。([vault.yml.example](file:///home/norik/Project/noriki-me-ansible/group_vars/all/vault.yml.example) をコピーして作成します)

| 変数名 | 説明 | 例 / 設定値 |
| :--- | :--- | :--- |
| `vault_host_bastion` | Bastion ホストの接続先 IP または AWS インスタンスID | `"your-bastion-instance-id-here"` |
| `vault_host_zabbix_server` | Zabbix Server の接続先 IP または ドメイン名 | `"monitor.noriki.me"` |
| `vault_wireguard_endpoint` | WireGuard サーバーの接続先となる外部 IP または ドメイン名 | `"your-vpn-server-ip-or-domain-here"` |
| `vault_wireguard_dns_server` | VPNクライアントに配布する内部 DNS サーバーの IP | `"your-internal-dns-ip-here"` |
| `vault_zabbix_server_ipv4` | Zabbix Proxy から見た Zabbix Server の接続用 IPv4 アドレス | `"your-zabbix-server-ipv4-here"` |
| `vault_zabbix_server_ipv6` | Zabbix Server の IPv6 アドレス | `"your-zabbix-server-ipv6-here"` |
| `vault_zabbix_proxy_psk_identity` | Zabbix Proxy - Server 間で TLS 接続に使用する PSK 識別子 | `"your-zabbix-proxy-psk-identity-here"` |
| `vault_zabbix_proxy_psk_key` | Zabbix Proxy - Server 間で TLS 接続に使用する PSK キー (16進数文字列) | `"your-zabbix-proxy-psk-key-here"` |
| `vault_cert_email` | Let's Encrypt の証明書登録および更新通知用メールアドレス | `"your-email-address"` |

### 2. ホスト変数: `host_vars/zabbix_server.yml`
Zabbix Server ホスト固有の設定変数です。([zabbix_server.yml.example](file:///home/norik/Project/noriki-me-ansible/host_vars/zabbix_server.yml.example) をコピーして作成し、`ansible-vault` で暗号化します)

| 変数名 | 説明 | 例 / 設定値 |
| :--- | :--- | :--- |
| `zabbix_domain` | Zabbix Web インターフェースに割り当てるドメイン名 | `"your-zabbix-domain"` |
| `aws_access_key_id` | Let's Encrypt DNS認証 (Route53) に使用する AWS アクセスキー ID | `"your-aws-accesskey"` |
| `aws_secret_access_key` | Let's Encrypt DNS認証 (Route53) に使用する AWS シークレットアクセスキー | `"your-aws-secretkey"` |
| `zabbix_db_name` | Zabbix Server 用のデータベース名 (通常はデフォルト) | `zabbix` |
| `zabbix_db_user` | Zabbix Server 用のデータベースユーザー名 (通常はデフォルト) | `zabbix` |
| `zabbix_db_password` | Zabbix Server 用のデータベースパスワード (暗号化を推奨) | `(任意のパスワード文字列)` |
