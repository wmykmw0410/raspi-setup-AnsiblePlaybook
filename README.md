# Ansible セットアップ

Raspberry Pi をNASサーバー・クライアントとして構成する Ansible Playbook です。

## 前提条件

- Raspberry Pi OS インストール済み
- SSH 接続可能な状態
- 実行元PCに Ansible インストール済み（`pip install ansible`）
- NASサーバー用: USB ドライブが exFAT フォーマット済み

## ファイル構成

```
ansible/
├── ansible.cfg
├── inventory/
│   ├── shinagawa.ini       # 品川拠点（NAS・クライアント）
│   ├── mitaka.ini          # 三鷹拠点（NAS・クライアント）
│   └── group_vars/
│       ├── all.yml         # 全ホスト共通変数
│       └── client.yml      # クライアント専用変数
├── playbooks/
│   ├── common.yml          # 共通ロール（全ホスト）
│   ├── nas.yml             # NASサーバーセットアップ
│   └── client.yml          # クライアントセットアップ
└── roles/
    ├── base/               # 共通: sudo・apt アップデート
    ├── locale/             # 共通: 日本語ロケール設定
    ├── keyboard/           # 共通: 日本語キーボード・入力設定
    ├── browser/            # 共通: Chromium ホームページ・ブックマーク設定
    ├── python_env/         # 共通: Python 環境
    ├── vscode/             # 共通: VS Code インストール
    ├── minecraft/          # 共通: Minecraft Pi
    ├── nas_server/         # NASサーバー: Samba・USB マウント設定
    │   ├── defaults/main.yml   # NASサーバー変数
    │   ├── tasks/main.yml
    │   └── templates/smb.conf  # Samba 設定テンプレート
    └── nas_mount/          # クライアント: NAS マウント設定
```

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

## NAS への接続方法

接続時のユーザー名・パスワードは `roles/nas_server/defaults/main.yml` の値を使用します。

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

### NASサーバー専用 (`roles/nas_server/defaults/main.yml`)

```yaml
samba_user: sambauser        # Samba アクセス用ユーザー
samba_password: swimmy       # Samba パスワード
share_name: nas              # 共有フォルダ名
usb_device: /dev/sda1        # USB デバイスパス
nas_mount: /media/swimmy/nas # USB マウントポイント
```
