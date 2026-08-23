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
- **credential.txt**: `~/Documents` に、rootパスワード・Wi-Fi情報・Googleドライブパスワードを記載したファイルを配置（拠点固有の値は `inventory/site_vars/<拠点>.yml` で設定。詳細は[変数リファレンス](docs/variables.md)）

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
├── ansible.cfg             # Ansible設定（roles_path・host_key_checking等）
├── docs/                   # セットアップ手順・トラブルシューティング・変数リファレンス
├── inventory/
│   ├── shinagawa.ini       # 品川拠点（NAS・クライアント）
│   ├── mitaka.ini          # 三鷹拠点（NAS・クライアント）
│   ├── kashiwa.ini         # 柏拠点（NAS・クライアント）
│   ├── takadanobaba.ini    # 高田馬場拠点（NAS・クライアント）
│   ├── group_vars/
│   │   ├── all.yml         # 全ホスト共通変数
│   │   ├── client.yml      # クライアント専用変数
│   │   └── nas.yml         # NASサーバー専用変数
│   └── site_vars/
│       ├── shinagawa.yml   # 品川拠点固有の変数（Wi-Fi・GoogleDriveパスワード等）
│       ├── mitaka.yml      # 三鷹拠点固有の変数
│       ├── kashiwa.yml     # 柏拠点固有の変数
│       └── takadanobaba.yml # 高田馬場拠点固有の変数
├── playbooks/
│   ├── common.yml          # 共通ロール（全ホスト）
│   ├── nas.yml             # NASサーバーセットアップ
│   ├── client.yml          # クライアントセットアップ
│   └── static_ip.yml       # 固定IPアドレスの設定（初回のみ）
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
    │   │   ├── ansible.yml     # Ansible本体のインストール（NAS上でも実行できるように）
    │   │   ├── usb_mount.yml   # USBドライブのマウント・fstab登録
    │   │   ├── samba_install.yml # Samba・exFATサポートのインストール
    │   │   ├── samba_user.yml  # Sambaユーザーの作成・パスワード設定
    │   │   └── samba_config.yml # smb.conf 配布・smbd 起動
    │   └── templates/smb.conf  # Samba 設定テンプレート
    ├── nas_mount/          # クライアント: NAS マウント設定
    ├── document/           # クライアント: Documentsフォルダに credential.txt を配置
    └── static_ip/          # 共通: 固定IPアドレスの設定・検証
        └── tasks/
            ├── set_ip.yml      # nmcli による固定IP設定（重複IP検知・新IP宛て接続先の登録含む）
            ├── wait_ssh.yml    # 新IPへのSSHポート到達待ち
            ├── check.yml       # 再起動後のIP設定確認
            └── ssh_auth.yml    # ID・パスワードでのSSH認証確認
```

## Ansible セットアップ手順

### 1. 接続先を編集する

拠点ごとのインベントリファイルを実際の環境に合わせて変更します。`[client]`・`[nas]` とも、各ホストに `ansible_host`（接続先）と `static_ip`（固定化したいIPアドレス）を指定します。

- `static_ip`: 各ホストに割り当てたい固定IPアドレス。[固定IPアドレスの設定](#固定ipアドレスの設定初回のみ)がこの値を使って実際にIPを固定化するほか、`nas_mount` ロールのマウント先にも使われるため、固定化がまだでも常に設定しておきます。
- `ansible_host`: 実際にAnsibleが接続する宛先。**初回はDHCPで割り振られたIPアドレスを記載**します。ラズパイ起動後、[Raspberry Pi OS セットアップ手順](docs/setup-raspberrypi.md)の「IPアドレスを確認する」の通り `ip addr show` またはルーターの管理画面で確認してください。DHCP予約をしていない環境では、このIPアドレスは `static_ip` と一致するとは限りません。[固定IPアドレスの設定](#固定ipアドレスの設定初回のみ)が完了すると、Playbookが自動的に `<hostname>.local`（mDNS）へ書き換えます。

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
# ansible_host には起動直後にDHCPで割り振られたIPアドレスを記載する（例）
raspi01 ansible_host=192.168.50.23 static_ip=192.168.50.11

[nas]
raspi-nas ansible_host=192.168.50.30 static_ip=192.168.50.10
```

[固定IPアドレスの設定](#固定ipアドレスの設定初回のみ)完了後は、`ansible_host` が自動的に `<hostname>.local` に書き換わります（手動での編集は不要です）。

```ini
[client]
raspi01 ansible_host=raspi01.local static_ip=192.168.50.11
```

### 2. 疎通確認

`ansible` コマンドで対象ホストにSSH接続できるか確認します。`-i` で使用するインベントリファイルを指定し、`all` でそのファイル内の全ホスト（`[client]`・`[nas]`）に対して `-m ping` モジュール（Pythonが実行できるか確認するAnsibleの疎通確認モジュール。ネットワークの `ping` コマンドとは異なります）を実行します。

```bash
# 品川拠点
ansible all -i inventory/shinagawa.ini -m ping --ask-vault-pass

# 三鷹拠点
ansible all -i inventory/mitaka.ini -m ping --ask-vault-pass

# 柏拠点
ansible all -i inventory/kashiwa.ini -m ping --ask-vault-pass

# 高田馬場拠点
ansible all -i inventory/takadanobaba.ini -m ping --ask-vault-pass
```

特定のホストのみ確認したい場合は、`all` の代わりにホスト名を指定します。

```bash
ansible raspi01 -i inventory/mitaka.ini -m ping --ask-vault-pass
```

接続できない場合は [トラブルシューティング](docs/troubleshooting.md) を参照してください。

## Playbook の実行

パスワード類（`ansible_password` 等）は `ansible-vault` で暗号化されているため、以降のすべての `ansible-playbook` 実行に `--ask-vault-pass` が必要です（Vaultパスワードを知っている人のみ実行できます）。

### 固定IPアドレスの設定（初回のみ）

手順1で `ansible_host` に記載したDHCPのIPアドレスへ接続し、`playbooks/static_ip.yml` を実行すると `static_ip` に指定したIPアドレスへの固定化・再起動・接続確認までを自動で行います。IP変更後は `ansible_host` の値が古くなりますが、Playbook内部で `static_ip` 宛ての接続先を動的に登録し直すため、この1回の実行の中で再起動・接続確認まで完結します。

```bash
ansible-playbook -i inventory/shinagawa.ini playbooks/static_ip.yml --ask-vault-pass
ansible-playbook -i inventory/mitaka.ini playbooks/static_ip.yml --ask-vault-pass
ansible-playbook -i inventory/kashiwa.ini playbooks/static_ip.yml --ask-vault-pass
ansible-playbook -i inventory/takadanobaba.ini playbooks/static_ip.yml --ask-vault-pass
```

実行が成功すると、インベントリファイルの `ansible_host` はDHCPのIPアドレスから `<hostname>.local` へ自動的に書き換わります（手動での編集は不要です）。以降はこのホスト名で接続します。

ラズパイを作り直した場合はSSHホスト鍵が変わり`known_hosts`との不一致警告で接続できなくなることがあります。その場合は [トラブルシューティング](docs/troubleshooting.md) を参照して対処してから実行してください。

`ansible-playbook` は `-i` で指定したインベントリファイルに対して、`playbooks/` 配下のPlaybookを実行します。`nas.yml` は `hosts: nas`（NASサーバー）、`client.yml` は `hosts: client`（クライアント端末）を対象に、それぞれ `common.yml`（全ホスト共通ロール）を取り込んだ上で拠点固有のロールを適用し、最後に自動再起動します。

```bash
# NASサーバーセットアップ（共通ロール + Samba・USB マウント → 自動再起動）
ansible-playbook -i inventory/shinagawa.ini playbooks/nas.yml --ask-vault-pass
ansible-playbook -i inventory/mitaka.ini playbooks/nas.yml --ask-vault-pass
ansible-playbook -i inventory/kashiwa.ini playbooks/nas.yml --ask-vault-pass
ansible-playbook -i inventory/takadanobaba.ini playbooks/nas.yml --ask-vault-pass
```

```bash
# クライアントセットアップ（全台・自動再起動）
# ※ --limit client を付けないと NAS にも common ロールが実行されるため必須
ansible-playbook -i inventory/shinagawa.ini playbooks/client.yml --limit client --ask-vault-pass
ansible-playbook -i inventory/mitaka.ini playbooks/client.yml --limit client --ask-vault-pass
ansible-playbook -i inventory/kashiwa.ini playbooks/client.yml --limit client --ask-vault-pass
ansible-playbook -i inventory/takadanobaba.ini playbooks/client.yml --limit client --ask-vault-pass
```

`--limit` にはグループ名の代わりに具体的なホスト名も指定できます。特定の1台（または一部）だけに絞って実行したい場合に使います。

```bash
# クライアントセットアップ（raspi01 だけに絞って実行。複数台の場合はカンマ区切り: raspi01,raspi02）
ansible-playbook -i inventory/mitaka.ini playbooks/client.yml --limit raspi01 --ask-vault-pass
```

`--check` を付けると実際には変更を適用せず、変更予定の内容だけを確認できます（ドライラン）。

```bash
ansible-playbook -i inventory/shinagawa.ini playbooks/nas.yml --check --ask-vault-pass
ansible-playbook -i inventory/mitaka.ini playbooks/nas.yml --check --ask-vault-pass
ansible-playbook -i inventory/kashiwa.ini playbooks/nas.yml --check --ask-vault-pass
ansible-playbook -i inventory/takadanobaba.ini playbooks/nas.yml --check --ask-vault-pass
```

`--diff` を付けると、`smb.conf` などテンプレート系ファイルの変更差分を確認できます（`--check`と併用すると実際には適用せず差分だけ確認できます）。

```bash
ansible-playbook -i inventory/mitaka.ini playbooks/nas.yml --check --diff --ask-vault-pass
```

### タグを指定した実行

`common.yml`（`base` / `locale` / `keyboard` / `browser` / `python_env` / `git` / `screenshot` / `vscode` / `minecraft` / `mdns`）と `static_ip.yml`（`mdns` / `static_ip` / `static_ip_check`）の各ロールにはタグが付いています。`--tags`・`--skip-tags` で一部のロールだけ実行・除外できます。

```bash
# browser ロールだけ実行
ansible-playbook -i inventory/mitaka.ini playbooks/client.yml --limit client --tags browser --ask-vault-pass

# vscode ロールだけ除外して実行
ansible-playbook -i inventory/mitaka.ini playbooks/client.yml --limit client --skip-tags vscode --ask-vault-pass
```

指定できるタグの一覧は `--list-tags` で確認できます。

```bash
ansible-playbook -i inventory/mitaka.ini playbooks/common.yml --list-tags
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

接続時のユーザー名・パスワードは `inventory/group_vars/nas.yml` の値を使用します。パスワードはVaultで暗号化されているため、確認するには以下を実行してください（Vaultパスワードが必要です）。

```bash
ansible localhost -m debug -a "var=samba_password" -e "@inventory/group_vars/nas.yml" --ask-vault-pass
```

| 項目 | デフォルト値 |
|---|---|
| ユーザー名 | `sambauser` |
| パスワード | ******（ヒント: スクール名） |

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
