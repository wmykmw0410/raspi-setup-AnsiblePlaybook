# 変数リファレンス

## 全ホスト共通 (`inventory/group_vars/all.yml`)

```yaml
local_user: swimmy                        # ログインユーザー
vscode_user: swimmy                       # VS Code 拡張機能インストール対象ユーザー
ansible_user: swimmy                      # SSH接続ユーザー
ansible_password: swimmy                  # SSH接続パスワード
ansible_become_password: swimmy           # sudo パスワード
ansible_python_interpreter: /usr/bin/python3
```

## クライアント専用 (`inventory/group_vars/client.yml`)

```yaml
nas_share: nas            # 共有フォルダ名
nas_user: sambauser       # Samba ユーザー名
nas_pass: swimmy          # Samba パスワード
mount_point: /mnt/nas     # クライアント側マウントポイント
```

> NASのIPアドレスは拠点ごとのインベントリファイル（`[nas]` セクション）から自動的に参照されます。

## NASサーバー専用 (`inventory/group_vars/nas.yml`)

```yaml
samba_user: sambauser        # Samba アクセス用ユーザー
samba_password: swimmy       # Samba パスワード
share_name: nas              # 共有フォルダ名
usb_device: /dev/sda1        # USB デバイスパス
nas_mount: /media/swimmy/nas # USB マウントポイント
```
