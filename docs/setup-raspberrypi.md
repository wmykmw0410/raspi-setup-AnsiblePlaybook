# Raspberry Pi OS セットアップ（手動）

Ansible 実行前に各ラズパイで以下を手動で行います。

## 1. OS を書き込む

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

## 2. IPアドレスを確認する

ラズパイ起動後、SSH でログインしてIPアドレスを確認します。

```bash
# ラズパイ上で実行
ip addr show
```

または、ルーターの管理画面から確認します。

確認したIPアドレスを `inventory/<拠点>.ini` の該当ホストの `ansible_host` に記載します（`static_ip` は固定化したい最終的なIPアドレスなので、この時点のDHCPのIPアドレスと一致するとは限りません。詳細は [README](../README.md#1-接続先を編集する) 参照）。

```ini
raspi01 ansible_host=192.168.50.23 static_ip=192.168.50.11
```

## 3. IPアドレスを固定化する

`inventory/<拠点>.ini` の `[client]`・`[nas]` セクションに各ホストの `ansible_host`（DHCPのIPアドレス）・`static_ip`（固定化したいIPアドレス）が設定済みであれば、以下を実行して固定化できます。

```bash
ansible-playbook -i inventory/<拠点>.ini playbooks/static_ip.yml
```

実行が成功すると、`inventory/<拠点>.ini` の該当ホストの `ansible_host` はDHCPのIPアドレスから `<hostname>.local` へ自動的に書き換わります（手動での編集は不要です）。

```ini
; 変更前
raspi01 ansible_host=192.168.50.23 static_ip=192.168.50.11
; 変更後（Playbookが自動で書き換える）
raspi01 ansible_host=raspi01.local static_ip=192.168.50.11
```

このコマンドが使えない場合は、GUI/CLIのいずれかで手動設定してください。

### GUI

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

### CLI
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
  ipv4.dns "8.8.8.8"
sudo nmcli con up "有線接続 1"
```

設定後、固定IPで再接続できることを確認します。

```bash
ip addr show
```

## 4. SSH 接続を確認する

実行元PCから接続できることを確認します。

```bash
ssh swimmy@<IPアドレス>
```

> ラズパイを作り直した（SDカードを焼き直した）場合にSSH接続できない場合は、[トラブルシューティング](./troubleshooting.md#ssh接続時にremote-host-identification-has-changedと出る) を参照してください。

`ansible.cfg` の `host_key_checking = False` は「未知のホスト」を自動承認する設定であり、ホストキーが変化したホストへの接続はブロックされたままです（詳細は[トラブルシューティング](./troubleshooting.md#ssh接続時にremote-host-identification-has-changedと出る)参照）。疎通確認・パスワード認証の動作確認を兼ねて、`ansible-playbook` の前に必ず一度手動SSHで接続できることを確認してください。

## 5. （NASサーバーのみ）USB デバイスパスを確認する

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
