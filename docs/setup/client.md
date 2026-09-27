# クライアント（授業用PC）手動設定手順

`common.yml`（日本語ロケール・キーボード・ブラウザ・VS Code・Python環境・Minecraft Pi等）のセットアップは完了済みであることを前提に、クライアント固有の設定である「NAS共有フォルダへの接続」のみをAnsibleを使わず手動で行う手順です。

基本セットアップがまだの場合は、[README](../../README.md) の手順で `playbooks/common.yml`（または `playbooks/client.yml`）を実行してください。

以下の値は例です。環境に合わせて読み替えてください。

| 項目 | 例 |
|---|---|
| ログインユーザー | `swimmy` |
| NASサーバーのホスト名 | `raspi-nas.local` |
| NAS共有フォルダ名 | `nas` |
| NAS接続用ユーザー | `sambauser`（[NAS 手動設定手順](nas.md)参照） |
| クライアント側マウントポイント | `/mnt/nas` |

## 1. NAS共有フォルダをマウントする

NASサーバー側の設定（未実施の場合）は [NAS 手動設定手順](nas.md) を参照してください。

```bash
sudo mkdir -p /mnt/nas
sudo apt install -y cifs-utils
```

`/etc/fstab` に以下の1行を追記します（ホスト名・共有名・ユーザー名・パスワードは環境に合わせて変更）。

```
//raspi-nas.local/nas   /mnt/nas   cifs   username=sambauser,password=<NASのパスワード>,iocharset=utf8,uid=1000,gid=1000,nofail,_netdev   0   0
```

```bash
# fstabの記述に問題がないか確認する（エラーが出なければOK）
sudo mount -a

# デスクトップにNASフォルダのショートカットを作成する
ln -s /mnt/nas ~/Desktop/NAS
```

## 2. 動作確認

1. デスクトップの `NAS` アイコンをダブルクリックする（またはデスクトップ左上のファイルマネージャーを開き、手順2へ）
2. アドレスバーに `/mnt/nas` と入力して移動する（アドレスバーが表示されていない場合は `Ctrl+L`）
3. NASサーバー上の共有フォルダ内のファイル・フォルダが表示されることを確認する
