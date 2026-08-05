# Ansible セットアップ

Raspberry Pi をNASサーバー・クライアントとして構成する Ansible Playbook です。

## Ansibleとは

Ansible は、サーバーやPCなどの構成管理・自動化を行うオープンソースのツールです。エージェントレスかつ宣言的な設定（Playbook）、冪等性が特徴で、`ansible-playbook` コマンドを実行するだけで拠点間・端末間で同一の設定を再現できます。基本用語や仕組みの詳細は [Ansibleとは（詳細）](docs/about-ansible.md) を参照してください。

このリポジトリでは、複数拠点・複数台の Raspberry Pi（NASサーバー・クライアント）に対して、OSセットアップ後の各種インストール・設定作業を Ansible で自動化しています。

## サービス概要

このリポジトリでは、Raspberry Pi を用いて拠点ごとに以下2種類の端末を構築します。

### NASサーバー

Samba によるファイル共有サーバーです。

- USB ドライブ（exFAT）をマウントし、ネットワーク経由でファイル共有（`\\<NASのIPアドレス>\nas` など）を提供
- 共有フォルダへの読み書きは Samba ユーザー（`sambauser`）で認証

### クライアント（授業用PC）

プログラミング学習用にセットアップされた端末です。

- **ブラウザ（Chromium）**: ホームページ・新しいタブページを教材ページに固定するポリシーを配信
- **VS Code**: 学習用拡張機能をインストール済み
- **Minecraft Pi**: Python（`mcpi` ライブラリ）でMinecraftを操作する学習環境
- **Python環境**: `pygame` / `flask` / `keyboard` 等の学習用ライブラリを導入
- **NASマウント**: NASサーバーの共有フォルダを起動時に自動マウント（`/mnt/nas`）

### 共通機能（NAS・クライアント共通）

- 日本語ロケール（`ja_JP.UTF-8`）・日本語キーボード（`fcitx5-mozc`）設定
- mDNS（avahi-daemon）による `<hostname>.local` 名前解決
- sudo権限付与、apt パッケージの自動アップデート、不要な印刷サービス（cups-browsed）の無効化
- Git のインストール
- スクリーンショットアプリ（gnome-screenshot）のインストール

## 前提条件

### 実行元PC

- Ansible インストール済み（未インストールの場合は [Ansible インストール手順](docs/setup-controller.md) を参照）

### クライアント用(授業用)

- Raspberry Pi OS インストール済み
- SSH 接続可能な状態

### NAS用

- Raspberry Pi OS インストール済み
- SSH 接続可能な状態
- USB ドライブが exFAT フォーマット済み

`Raspberry Pi OS` のインストール・初期設定手順は [Raspberry Pi OS セットアップ手順](docs/setup-raspberrypi.md) を参照してください。

## ファイル構成

```
ansible/
├── ansible.cfg
├── docs/                   # セットアップ手順・トラブルシューティング・変数リファレンス
├── inventory/
│   ├── shinagawa.ini       # 品川拠点（NAS・クライアント）
│   ├── mitaka.ini          # 三鷹拠点（NAS・クライアント）
│   ├── kashiwa.ini         # 柏拠点（NAS・クライアント）
│   ├── takadanobaba.ini    # 高田馬場拠点（NAS・クライアント）
│   └── group_vars/
│       ├── all.yml         # 全ホスト共通変数
│       ├── client.yml      # クライアント専用変数
│       └── nas.yml         # NASサーバー専用変数
├── playbooks/
│   ├── common.yml          # 共通ロール（全ホスト）
│   ├── nas.yml             # NASサーバーセットアップ
│   └── client.yml          # クライアントセットアップ
└── roles/
    ├── base/               # 共通: sudo・apt アップデート
    ├── locale/             # 共通: 日本語ロケール設定
    ├── keyboard/           # 共通: 日本語キーボード・入力設定
    ├── browser/            # 共通: Chromium 新しいタブページ・ポリシー設定
    ├── python_env/         # 共通: Python 環境
    ├── git/                # 共通: Git インストール
    ├── screenshot/         # 共通: スクリーンショットアプリ（gnome-screenshot）インストール
    ├── vscode/             # 共通: VS Code インストール
    │   ├── tasks/
    │   │   ├── main.yml        # include_tasks エントリーポイント
    │   │   ├── install.yml     # VS Code 本体インストール
    │   │   ├── extensions.yml  # 拡張機能インストール
    │   │   └── settings.yml    # ユーザー設定（エクスプローラー・キーボードショートカット等）
    │   └── files/extensions.txt # 拡張機能リスト
    ├── minecraft/          # 共通: Minecraft Pi
    ├── mdns/               # 共通: avahi-daemon による mDNS (.local) 名前解決
    ├── nas_server/         # NASサーバー: Samba・USB マウント設定
    │   ├── handlers/main.yml   # ハンドラ（smbd 再起動）
    │   ├── tasks/
    │   │   ├── main.yml        # include_tasks エントリーポイント
    │   │   ├── ansible.yml
    │   │   ├── usb_mount.yml
    │   │   ├── samba_install.yml
    │   │   ├── samba_user.yml
    │   │   └── samba_config.yml
    │   └── templates/smb.conf  # Samba 設定テンプレート
    └── nas_mount/          # クライアント: NAS マウント設定
```

## Ansible セットアップ手順

### 1. 接続先を編集する

拠点ごとのインベントリファイルを実際の環境に合わせて変更します。`[nas]` はIPアドレスで記載します。`[client]` は最終的にホスト名（`<hostname>.local`）で記載しますが、`mdns` ロール適用前（初回セットアップ時）は `.local` の名前解決ができないため、**初回のみIPアドレスを記載**し、初回セットアップ完了後にホスト名へ書き換えます（詳細は [Playbook の実行](#playbook-の実行) 参照）。

インベントリファイルは拠点ごとに `inventory/` 配下に分かれており、いずれも同じ `[client]` / `[nas]` の構成です。

| 拠点 | ファイル |
|---|---|
| 品川 | `inventory/shinagawa.ini` |
| 三鷹 | `inventory/mitaka.ini` |
| 柏 | `inventory/kashiwa.ini` |
| 高田馬場 | `inventory/takadanobaba.ini` |

以下は三鷹拠点を例にした編集内容です。他拠点も同様の形式で該当ファイルを編集してください。

**三鷹拠点** `inventory/mitaka.ini`:

```ini
[client]
# 初回セットアップ時（mDNS未設定）: IPアドレスを記載
raspi01 ansible_host=192.168.50.11
# 初回セットアップ完了後: ホスト名に書き換える
# raspi01 ansible_host=raspi01.local

[nas]
192.168.50.10
```

### 2. 疎通確認

`ansible` コマンドで対象ホストにSSH接続できるか確認します。`-i` で使用するインベントリファイルを指定し、`all` でそのファイル内の全ホスト（`[client]`・`[nas]`）に対して `-m ping` モジュール（Pythonが実行できるか確認するAnsibleの疎通確認モジュール。ネットワークの `ping` コマンドとは異なります）を実行します。

```bash
# 品川拠点
ansible all -i inventory/shinagawa.ini -m ping

# 三鷹拠点
ansible all -i inventory/mitaka.ini -m ping

# 柏拠点
ansible all -i inventory/kashiwa.ini -m ping

# 高田馬場拠点
ansible all -i inventory/takadanobaba.ini -m ping
```

特定のホストのみ確認したい場合は、`all` の代わりにホスト名を指定します。

```bash
ansible raspi01 -i inventory/mitaka.ini -m ping
```

接続できない場合は [トラブルシューティング](docs/troubleshooting.md) を参照してください。

## Playbook の実行

### 初回セットアップ時の注意（mDNS未設定の場合）

手順1の通り、`mdns` ロール適用前の初回セットアップ時はインベントリファイルの `[client]` にIPアドレスを記載します。

ラズパイを作り直した場合はSSHホスト鍵が変わり`known_hosts`との不一致警告で接続できなくなります。その場合は [トラブルシューティング](docs/troubleshooting.md) を参照して対処してから実行してください。

```bash
# 初回のみ: --limit で対象ホストを1台に絞って実行
# ※ 他拠点の場合は -i を該当のインベントリファイルに置き換える
ansible-playbook -i inventory/shinagawa.ini playbooks/client.yml --limit <ホスト名>
```

`--limit <ホスト名>` の `<ホスト名>` は、手順1でインベントリファイルの `[client]` に追加したホスト名（`raspi01` など。IPアドレスではなくインベントリ上のホスト名を指定します）です。複数台を同時に初回セットアップする場合は、ホストごとに `--limit` を変えてコマンドを1回ずつ実行します。

初回セットアップ完了後（`mdns` ロール適用後）は、インベントリファイルの `[client]` を手順1の通りホスト名（`<hostname>.local`）に書き換えます。以降は以下の通常コマンドで実行できます。

`ansible-playbook` は `-i` で指定したインベントリファイルに対して、`playbooks/` 配下のPlaybookを実行します。`nas.yml` は `hosts: nas`（NASサーバー）、`client.yml` は `hosts: client`（クライアント端末）を対象に、それぞれ `common.yml`（全ホスト共通ロール）を取り込んだ上で拠点固有のロールを適用し、最後に自動再起動します。

```bash
# NASサーバーセットアップ（共通ロール + Samba・USB マウント → 自動再起動）
ansible-playbook -i inventory/shinagawa.ini playbooks/nas.yml
ansible-playbook -i inventory/mitaka.ini playbooks/nas.yml
ansible-playbook -i inventory/kashiwa.ini playbooks/nas.yml
ansible-playbook -i inventory/takadanobaba.ini playbooks/nas.yml
```

```bash
# クライアントセットアップ（自動再起動）
# ※ --limit client を付けないと NAS にも common ロールが実行されるため必須
ansible-playbook -i inventory/shinagawa.ini playbooks/client.yml --limit client
ansible-playbook -i inventory/mitaka.ini playbooks/client.yml --limit client
ansible-playbook -i inventory/kashiwa.ini playbooks/client.yml --limit client
ansible-playbook -i inventory/takadanobaba.ini playbooks/client.yml --limit client
```

`--check` を付けると実際には変更を適用せず、変更予定の内容だけを確認できます（ドライラン）。

```bash
ansible-playbook -i inventory/shinagawa.ini playbooks/nas.yml --check
ansible-playbook -i inventory/mitaka.ini playbooks/nas.yml --check
ansible-playbook -i inventory/kashiwa.ini playbooks/nas.yml --check
ansible-playbook -i inventory/takadanobaba.ini playbooks/nas.yml --check
```

common ロール（`mdns`）適用後は、各ラズパイに `<hostname>.local`（例: `raspi-nas.local`）でIPアドレスの代わりにアクセスできます。

## VS Code の日本語化（クライアントのみ・手動）

`vscode` ロールで日本語言語パック（`ms-ceintl.vscode-language-pack-ja`）はインストール済みですが、表示言語の切り替えは自動化していないため、クライアント端末ごとに初回起動時に手動で切り替えます。

1. VS Code を起動する
2. `Ctrl+Shift+P` でコマンドパレットを開く
3. `Configure Display Language` と入力して実行
4. 一覧から `日本語` を選択する
5. 表示される通知から VS Code を再起動する

## NAS への接続方法

接続時のユーザー名・パスワードは `inventory/group_vars/nas.yml` の値を使用します。

| 項目 | デフォルト値 |
|---|---|
| ユーザー名 | `sambauser` |
| パスワード | `swimmy` |

| OS | 方法 |
|---|---|
| Windows | エクスプローラーに `\\<NASのIPアドレス>\nas` を入力し、上記で認証 |
| Linux（CLI） | `smbclient //<NASのIPアドレス>/nas -U sambauser` を実行しパスワードを入力 |
| Linux（GUI） | ファイルマネージャーのアドレス欄に `smb://<NASのIPアドレス>/nas` を入力し、上記で認証 |

NASのIPアドレスは拠点ごとのインベントリファイルの `[nas]` セクションを参照してください。

---

## 関連ドキュメント

- [Ansibleとは（詳細）](docs/about-ansible.md)
- [Ansible インストール（実行元PC）](docs/setup-controller.md)
- [Raspberry Pi OS セットアップ（手動）](docs/setup-raspberrypi.md)
- [トラブルシューティング](docs/troubleshooting.md)
- [変数リファレンス](docs/variables.md)
