# NAS 手動設定手順

Ansible を使わずに、NASサーバー用のラズパイをコマンド操作だけで手動セットアップする手順です。Raspberry Pi OS 自体のセットアップ（書き込み・IP確認・SSH確認）は [Raspberry Pi OS セットアップ手順](raspberrypi.md) を参照してください。

以下の値は例です。環境に合わせて読み替えてください。

| 項目 | 例 |
|---|---|
| ログインユーザー / Sambaユーザー | `swimmy` |
| USBデバイスパス | `/dev/sda1` |
| マウントポイント | `/media/swimmy/nas` |
| 共有フォルダ名 | `nas` |

## 1. USBドライブをexFATでフォーマットする

### Windows / Mac でフォーマットする場合

ラズパイに接続する前に、PC側でフォーマットしておく方法です。

| OS | 手順 |
|---|---|
| Windows | エクスプローラーでドライブを右クリック →「フォーマット」→ ファイルシステムに `exFAT` を選択 → 「開始」 |
| Mac | 「ディスクユーティリティ」でドライブを選択 →「消去」→ フォーマットに `ExFAT` を選択 → 「消去」 |

### ラズパイ上でフォーマットする場合

```bash
# exFATフォーマット用パッケージをインストール
sudo apt install exfatprogs

# デバイスパスを確認する（NAS用USBドライブのみを接続した状態で実行）
lsblk
```

出力例（`/dev/sda` にパーティション `/dev/sda1` がある場合）:

```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    1  119G  0 disk
└─sda1        8:1    1  119G  0 part
```

```bash
# /dev/sda1 を exFAT でフォーマットする（デバイスパスは環境に合わせて変更）
sudo mkfs.exfat -n NAS /dev/sda1
```

> ドライブにパーティションが存在しない場合は、`sudo fdisk /dev/sda` 等で事前にパーティションを作成してから `mkfs.exfat` を実行してください。
> `mkfs.exfat` はドライブ内のデータを消去します。既存データがある場合は事前にバックアップしてください。

## 2. USBドライブをマウントする

デバイスパスは `lsblk` で確認したものに読み替えてください（手順は [Raspberry Pi OS セットアップ手順の「USBデバイスパスを確認する」](raspberrypi.md#5-nasサーバーのみusb-デバイスパスを確認する) と同様）。

```bash
# マウントポイントを作成する
sudo mkdir -p /media/swimmy/nas

# 一時的にマウントして動作確認する
sudo mount -t exfat -o defaults,nofail,umask=000 /dev/sda1 /media/swimmy/nas
```

再起動後も自動マウントされるように `/etc/fstab` に登録します。

```bash
# UUIDを確認する（fstabにはデバイスパスではなくUUIDを使うと安全）
sudo blkid /dev/sda1
```

`/etc/fstab` に以下の1行を追記します（`UUID=...` は上記コマンドの出力に置き換えてください）。

```
UUID=<確認したUUID> /media/swimmy/nas exfat defaults,nofail,umask=000 0 0
```

```bash
# fstabの記述に問題がないか確認する（エラーが出なければOK）
sudo mount -a
```

## 3. Samba をインストールする

```bash
sudo apt install samba exfatprogs
```

## 4. Samba ユーザーを作成する

ログインユーザー（`swimmy`）をそのままSambaユーザーとして登録します。

```bash
# Sambaユーザーとして登録し、パスワードを設定する（対話式でパスワード入力を求められます）
sudo smbpasswd -a swimmy

# ユーザーを有効化する
sudo smbpasswd -e swimmy
```

## 5. smb.conf を設定する

`/etc/samba/smb.conf` を編集します（設定内容は [roles/nas_server/templates/smb.conf](../../roles/nas_server/templates/smb.conf) と同じです）。

```bash
sudo nano /etc/samba/smb.conf
```

以下の内容に書き換えます（`{{ }}` 部分は上記の例の値に置き換え済みです）。

```ini
[global]
   workgroup = WORKGROUP
   server string = Samba Server
   log file = /var/log/samba/log.%m
   max log size = 1000
   logging = file
   server role = standalone server
   map to guest = bad user
   server min protocol = SMB2
   server max protocol = SMB3

[nas]
   path = /media/swimmy/nas
   browseable = yes
   writable = yes

   valid users = swimmy
   force user = swimmy
   force group = swimmy

   create mask = 0664
   directory mask = 0775
```

## 6. smbd を起動する

```bash
# 設定ファイルの文法チェック
sudo testparm

# smbdを再起動して設定を反映し、自動起動を有効にする
sudo systemctl restart smbd
sudo systemctl enable smbd
```

## 7. 動作確認

```bash
# smbdの起動状態を確認する
sudo systemctl status smbd

# 共有フォルダの一覧・接続確認（ローカルから）
smbclient -L localhost -U swimmy
```

クライアントPCからの接続方法（Windows/Linux）は [README の「NAS への接続方法」](../../README.md#nas-への接続方法) を参照してください。
