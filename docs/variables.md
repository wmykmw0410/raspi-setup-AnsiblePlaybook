# 変数リファレンス

パスワード類は `ansible-vault` で暗号化されています。実際の値を確認する方法は [README](../README.md#nas-への接続方法) を参照してください。

## 全ホスト共通 (`inventory/group_vars/all.yml`)

```yaml
local_user: swimmy                        # ログインユーザー
vscode_user: swimmy                       # VS Code 拡張機能インストール対象ユーザー
ansible_user: swimmy                      # SSH接続ユーザー
ansible_password: <Vault化>               # SSH接続パスワード
ansible_become_password: <Vault化>        # sudo パスワード
ansible_python_interpreter: /usr/bin/python3
dns_servers:                              # 固定IP化時に設定するDNSサーバー
  - 8.8.8.8
```

## クライアント専用 (`inventory/group_vars/client.yml`)

```yaml
nas_share: nas             # 共有フォルダ名
nas_user: sambauser        # Samba ユーザー名
nas_pass: <Vault化>        # Samba パスワード
mount_point: /mnt/nas      # クライアント側マウントポイント
root_password: <Vault化>   # credential.txt に記載する root ユーザパスワード（全拠点共通）
```

> NASのマウント先IPアドレスは、拠点ごとのインベントリファイル（`[nas]` セクションの `static_ip`）から自動的に参照されます。

## NASサーバー専用 (`inventory/group_vars/nas.yml`)

```yaml
samba_user: sambauser        # Samba アクセス用ユーザー
samba_password: <Vault化>    # Samba パスワード
share_name: nas              # 共有フォルダ名
usb_device: /dev/sda1        # USB デバイスパス
nas_mount: /media/swimmy/nas # USB マウントポイント
```

## 拠点別 (`inventory/site_vars/<拠点>.yml`)

拠点ごとに異なる値を持つ変数です。`playbooks/common.yml` の `pre_tasks` が、実行時のインベントリファイル名（`mitaka.ini` → `mitaka` など）から対応するファイルを自動的に読み込みます。

```yaml
wifi_ssid: "Swimmy_Shinagawa-Wi-Fi_5G"  # credential.txt に記載するWi-Fi SSID
wifi_password: <Vault化 または 平文>     # credential.txt に記載するWi-Fiパスワード
googledrive_password: <Vault化 または 平文>  # credential.txt に記載するGoogleドライブパスワード

extra_shortcuts:                        # ブラウザ新しいタブページに表示する拠点固有ショートカット
  - label: 座席表
    url: "https://docs.google.com/..."
  - label: GoogleDrive
    url: "https://drive.google.com/..."
```

## インベントリのホスト変数 (`inventory/<拠点>.ini`)

`[client]`・`[nas]` の各ホスト行に直接記載する変数です。

```ini
raspi01 ansible_host=raspi01.local static_ip=192.168.50.11
```

- `ansible_host`: 実際にAnsibleが接続する宛先。初回はDHCPのIPアドレス、`playbooks/static_ip.yml` 実行後は自動的に `<hostname>.local` に書き換わります
- `static_ip`: 固定化したいIPアドレス。`static_ip` ロールでの固定化と、`nas_mount` ロールのマウント先の両方に使われます

詳細は [README の「1. 接続先を編集する」](../README.md#1-接続先を編集する) を参照してください。
