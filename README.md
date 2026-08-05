# Ansible セットアップ

Raspberry Pi をNASサーバー・クライアントとして構成する Ansible Playbook です。

## Ansibleとは

Ansible は、サーバーやPCなどの構成管理・自動化を行うオープンソースのツールです。

- **エージェントレス**: 対象端末に専用ソフトウェアを事前インストールする必要がなく、SSHで接続できればセットアップ可能
- **宣言的な設定（Playbook）**: 「どうやるか」ではなく「どうあるべきか」をYAML形式の Playbook に記述する
- **冪等性**: 同じ Playbook を何度実行しても、既に設定済みの項目はスキップされ、常に同じ状態に収束する

このリポジトリでは、複数拠点・複数台の Raspberry Pi（NASサーバー・クライアント）に対して、OSセットアップ後の各種インストール・設定作業を Ansible で自動化しています。手作業でのセットアップと異なり、`ansible-playbook` コマンドを実行するだけで、拠点間・端末間で同一の設定を再現できます。

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
- sudo権限付与、apt パッケージの自動アップデート

## 前提条件

### 実行元PC

- Ansible インストール済み（詳細は下記「Ansible インストール（実行元PC）」参照）

### クライアント用(授業用)

- Raspberry Pi OS インストール済み
- SSH 接続可能な状態

### NAS用

- Raspberry Pi OS インストール済み
- SSH 接続可能な状態
- USB ドライブが exFAT フォーマット済み

## ファイル構成

```
ansible/
├── ansible.cfg
├── inventory/
│   ├── shinagawa.ini       # 品川拠点（NAS・クライアント）
│   ├── mitaka.ini          # 三鷹拠点（NAS・クライアント）
│   ├── kashiwa.ini         # 柏拠点（NAS・クライアント）
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
    ├── vscode/             # 共通: VS Code インストール
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

## Ansible インストール（実行元PC）

Ansible を実行するPCにAnsibleをインストールします。OSごとの手順は以下の通りです。

> 仮想環境（venv）の利用は任意です。他のPythonプロジェクトとパッケージを分離したい場合は、`pip install` の前に以下を実行してください（以降、`pip` / `ansible` コマンドを使うたびに `source` の実行が必要です）。
>
> ```bash
> python3 -m venv ~/.venv/ansible
> source ~/.venv/ansible/bin/activate    # Windows(WSL)は同じ、Windows(PowerShell等)は非対応のためWSL必須
> ```

### macOS

1. Python のバージョンを確認します（3.9 以上が必要）。未インストールの場合は [Homebrew](https://brew.sh/) 等でインストールします。

   ```bash
   python3 --version
   ```

2. Ansible をインストールします。

   ```bash
   pip3 install --upgrade pip
   pip3 install ansible
   ```

3. インストールを確認します。

   ```bash
   ansible --version
   ```

### Windows

Ansible の実行環境は Linux/macOS 前提のため、Windows では **WSL（Windows Subsystem for Linux）** を利用します。

1. PowerShell を管理者権限で開き、WSL と Ubuntu をインストールします。

   ```powershell
   wsl --install
   ```

   インストール後、PCを再起動しUbuntuの初回セットアップ（ユーザー名・パスワード設定）を行います。

2. WSL（Ubuntu）のターミナルを開き、パッケージを更新して Python / pip をインストールします。

   ```bash
   sudo apt update
   sudo apt install -y python3 python3-pip
   ```

3. Ansible をインストールします。

   ```bash
   pip3 install --upgrade pip
   pip3 install ansible
   ```

4. インストールを確認します。

   ```bash
   ansible --version
   ```

> 以降の `ansible` / `ansible-playbook` コマンドはすべて WSL（Ubuntu）のターミナル上で実行してください。

### Linux（Ubuntu/Debian系）

1. パッケージを更新して Python / pip をインストールします。

   ```bash
   sudo apt update
   sudo apt install -y python3 python3-pip
   ```

2. Ansible をインストールします。

   ```bash
   pip3 install --upgrade pip
   pip3 install ansible
   ```

3. インストールを確認します。

   ```bash
   ansible --version
   ```

バージョン情報が表示されればインストール完了です。

## Raspberry Pi OS セットアップ（手動）

Ansible 実行前に各ラズパイで以下を手動で行います。

### 1. OS を書き込む

[Raspberry Pi Imager](https://www.raspberrypi.com/software/) でSDカードに書き込みます。

- **NASサーバー・クライアント共通**: Raspberry Pi OS（64-bit）推奨

Imager の Setup steps に沿って以下を設定します。

**Device**

| 項目 | 設定値 |
|---|---|
| デバイス | `Raspberry Pi 4` |

**OS**

| 項目 | 設定値 |
|---|---|
| OS | `Raspberry Pi OS (64-bit)`（Recommended） |

**ストレージ**

| 項目 | 設定値 |
|---|---|
| ストレージ | 書き込み先の SD カードを選択 |

**Customisation → Hostname**

| 項目 | 設定値 |
|---|---|
| ホスト名 | NASサーバー: `raspi-nas` / クライアント: `raspi0x`（x = IPアドレス第4オクテットの一の位、例: IP末尾が13なら `raspi03`） |

**Customisation → Localisation**

| 項目 | 設定値 |
|---|---|
| Capital city | `Tokyo (Japan)` |
| Time zone | `Asia/Tokyo` |
| キーボードレイアウト | `jp` |

**Customisation → User**

| 項目 | 設定値 |
|---|---|
| ユーザー名 | `swimmy` |
| パスワード | `swimmy` |

**Customisation → Wi-Fi**

| 項目 | 設定値 |
|---|---|
| Wi-Fi | 必要に応じて設定 |

**Customisation → Remote access**

| 項目 | 設定値 |
|---|---|
| SSH | 有効化（パスワード認証） |

**Customisation → Raspberry Pi Connect**

| 項目 | 設定値 |
|---|---|
| Enable Raspberry Pi Connect | 無効（OFF） |

**Writing**

Summary に以下が表示されていることを確認して「WRITE」をクリックします。

| 項目 | 値 |
|---|---|
| Device | Raspberry Pi 4 |
| Operating system | Raspberry Pi OS (64-bit) |
| Storage | 書き込み先の SD カード |

Customisations to apply に以下が表示されていることを確認します。

- Hostname configured
- Localisation configured
- User account configured
- SSH enabled

### 2. IPアドレスを確認する

ラズパイ起動後、SSH でログインしてIPアドレスを確認します。

```bash
# ラズパイ上で実行
ip addr show
```

または、ルーターの管理画面から確認します。

### 3. IPアドレスを固定化する(GUI/CUIのどちらかを実施)
#### GUI

1. タスクバー右上のネットワークアイコンを右クリック →「Edit Connections...」を選択
2. 使用中の接続（有線: `Wired connection 1` など）を選択 → 鉛筆アイコン（編集）をクリック
3. 「IPv4 Settings」タブを開く
4. 「Method」を `Automatic (DHCP)` → `Manual` に変更
5. 「Add」をクリックし、以下を入力する

   | 項目 | 値 |
   |---|---|
   | Address | `192.168.0.13` |
   | Netmask | `24` |
   | Gateway | `192.168.0.1` |

6. 「DNS servers」に `8.8.8.8` を入力
7. 「Save」をクリック
8. ネットワークアイコンから一度切断し、再接続する

#### CLI
ラズパイ上で以下のコマンドを実行します。接続名（`有線接続 1` など）は環境によって異なるため、事前に確認します。

```bash
# 接続名を確認する
nmcli con show
```

確認した接続名を使って固定IPを設定します。

```bash
# NASサーバーの場合（例: 品川 192.168.0.10 / 三鷹 192.168.50.10）
sudo nmcli con mod "有線接続 1" \
  ipv4.method manual \
  ipv4.addresses <NASのIPアドレス>/24 \
  ipv4.gateway <ゲートウェイ> \
  ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con up "有線接続 1"

# クライアントの場合（192.168.0.13）
sudo nmcli con mod "有線接続 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.0.13/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con up "有線接続 1"
```

設定後、固定IPで再接続できることを確認します。

```bash
ip addr show
```


### 4. SSH 接続を確認する

実行元PCから接続できることを確認します。

```bash
ssh swimmy@<IPアドレス>
```

> **ラズパイを作り直した（SDカードを焼き直した）場合**
> 同じIPアドレスでもSSHホストキーが変わるため、`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` という警告が出て接続できません。
> 以下のコマンドで実行元PCの `known_hosts` から古い鍵を削除してから、再度SSH接続してください。
>
> ```bash
> ssh-keygen -R <IPアドレス>
> ```
>
> 削除後、再度 `ssh swimmy@<IPアドレス>` を実行すると鍵の確認を求められるので `yes` と入力して承認します。
>
> ```bash
> ssh swimmy@<IPアドレス>
> # The authenticity of host '<IPアドレス>' can't be established.
> # ...
> # Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
> ```
>
> このリポジトリの `ansible.cfg` では `host_key_checking = False` によりAnsible自体はホストキー不一致でも実行できますが、疎通確認・パスワード認証の動作確認を兼ねて、`ansible-playbook` の前に一度手動SSHで接続できることを確認してください。

### 5. （NASサーバーのみ）USB デバイスパスを確認する

USB ドライブを接続して、デバイスパスを確認します。

```bash
lsblk
```

出力例:
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    1  119G  0 disk
└─sda1        8:1    1  119G  0 part
mmcblk0     179:0    0   32G  0 disk
├─mmcblk0p1 179:1    0  256M  0 part /boot
└─mmcblk0p2 179:2    0 31.8G  0 part /
```

`/dev/sda1` 以外のパスが表示された場合は、他のUSBストレージデバイスが接続されていないか確認します。
NAS用USBドライブのみを接続すると、常に `/dev/sda1` として認識されます。

---

## Ansible セットアップ手順

### 1. 接続先を編集する

拠点ごとのインベントリファイルのIPアドレスを実際の環境に合わせて変更します。

**品川拠点** `inventory/shinagawa.ini`:

```ini
[client]
shinagawa-client ansible_host=192.168.0.13

[shinagawa]
shinagawa-client

[nas]
192.168.0.10
```

**三鷹拠点** `inventory/mitaka.ini`:

```ini
[client]
mitaka-client ansible_host=192.168.0.13

[mitaka]
mitaka-client

[nas]
192.168.50.10
```

### 2. 疎通確認

```bash
# 品川拠点
ansible all -i inventory/shinagawa.ini -m ping

# 三鷹拠点
ansible all -i inventory/mitaka.ini -m ping

# 柏拠点
ansible all -i inventory/kashiwa.ini -m ping
```

## Playbook の実行

```bash
# NASサーバーセットアップ（共通ロール + Samba・USB マウント → 自動再起動）
ansible-playbook -i inventory/shinagawa.ini playbooks/nas.yml
ansible-playbook -i inventory/mitaka.ini playbooks/nas.yml
ansible-playbook -i inventory/kashiwa.ini playbooks/nas.yml

# クライアントセットアップ（拠点ごとに -i で指定 → 自動再起動）
# ※ --limit client を付けないと NAS にも common ロールが実行されるため必須
ansible-playbook -i inventory/shinagawa.ini playbooks/client.yml --limit client
ansible-playbook -i inventory/mitaka.ini playbooks/client.yml --limit client
ansible-playbook -i inventory/kashiwa.ini playbooks/client.yml --limit client

# ドライラン（実際には変更しない）
ansible-playbook -i inventory/shinagawa.ini playbooks/nas.yml --check
ansible-playbook -i inventory/kashiwa.ini playbooks/nas.yml --check
```

common ロール（`mdns`）適用後は、各ラズパイに `<hostname>.local`（例: `raspi-nas.local`）でIPアドレスの代わりにアクセスできます。

## NAS への接続方法

接続時のユーザー名・パスワードは `inventory/group_vars/nas.yml` の値を使用します。

| 項目 | デフォルト値 |
|---|---|
| ユーザー名 | `sambauser` |
| パスワード | `swimmy` |

| OS | 方法 |
|---|---|
| Windows | エクスプローラーに `\\<NASのIPアドレス>\nas` を入力し、上記で認証 |
| Linux | `smbclient //<NASのIPアドレス>/nas -U sambauser` を実行しパスワードを入力 |
smb://192.168.x.10

NASのIPアドレスは拠点ごとのインベントリファイルの `[nas]` セクションを参照してください。

---

## 変数リファレンス

### 全ホスト共通 (`inventory/group_vars/all.yml`)

```yaml
local_user: swimmy                        # ログインユーザー
vscode_user: swimmy                       # VS Code 拡張機能インストール対象ユーザー
ansible_user: swimmy                      # SSH接続ユーザー
ansible_password: swimmy                  # SSH接続パスワード
ansible_become_password: swimmy           # sudo パスワード
ansible_python_interpreter: /usr/bin/python3
```

### クライアント専用 (`inventory/group_vars/client.yml`)

```yaml
nas_share: nas            # 共有フォルダ名
nas_user: sambauser       # Samba ユーザー名
nas_pass: swimmy          # Samba パスワード
mount_point: /mnt/nas     # クライアント側マウントポイント
```

> NASのIPアドレスは拠点ごとのインベントリファイル（`[nas]` セクション）から自動的に参照されます。

### NASサーバー専用 (`inventory/group_vars/nas.yml`)

```yaml
samba_user: sambauser        # Samba アクセス用ユーザー
samba_password: swimmy       # Samba パスワード
share_name: nas              # 共有フォルダ名
usb_device: /dev/sda1        # USB デバイスパス
nas_mount: /media/swimmy/nas # USB マウントポイント
```
