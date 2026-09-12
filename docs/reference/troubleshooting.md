# トラブルシューティング

## SSH接続時に「REMOTE HOST IDENTIFICATION HAS CHANGED」と出る

ラズパイを作り直した（SDカードを焼き直した）場合、同じIPアドレスでもSSHホストキーが変わるため、`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` という警告が出て接続できません。

以下のコマンドで実行元PCの `known_hosts` から古い鍵を削除してから、再度SSH接続してください。

```bash
ssh-keygen -R <IPアドレスまたはホスト名>
```

削除後、再度 `ssh swimmy@<IPアドレス>` を実行すると鍵の確認を求められるので `yes` と入力して承認します。

```bash
ssh swimmy@<IPアドレス>
# The authenticity of host '<IPアドレス>' can't be established.
# ...
# Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

開発・検証などでラズパイを何度も作り直す場合は、`ansible-playbook` の前に上記の `ssh-keygen -R` と手動SSH接続確認を都度行ってください。

### ホスト名（`.local`）でSSHする場合の注意

`known_hosts` は接続に使った文字列（IPアドレスかホスト名か）ごとに**別々のエントリ**として鍵を記録します。そのため、IPアドレスで `ssh-keygen -R` しても `raspi01.local` のようなホスト名側のエントリは削除されず、逆も同様です。`ansible_host` がIPからホスト名（またはその逆）に変わるタイミング（[固定IPアドレスの設定](./setup-raspberrypi.md#3-ipアドレスを固定化する)前後など）で接続方法が切り替わる場合は、両方削除しておくと安全です。

```bash
ssh-keygen -R <IPアドレス>
ssh-keygen -R <hostname>.local
```

さらにこのリポジトリでは、`raspi01`〜`raspi10`・`raspi-nas` というホスト名を**全拠点で使い回して**います。`.local`（mDNS）名はネットワークセグメント内でしか一意性が保証されないため、「三鷹の`raspi06`」と「品川の`raspi06`」は物理的に別のPiでも同じ`raspi06.local`を名乗ります。1台のPCから複数拠点のPiに順に接続する場合、SDカードを焼き直していなくても、**別拠点の同名ホストの鍵が`known_hosts`に残っていることが原因で**この警告が出ることがあります。拠点を切り替える前に、使い回しているホスト名をまとめて削除しておくと事故を防げます。

```bash
for h in raspi01 raspi02 raspi03 raspi04 raspi05 raspi06 raspi07 raspi10 raspi-nas; do
  ssh-keygen -R "${h}.local" 2>/dev/null
done
```

なお、このリポジトリの `ansible.cfg` は `host_key_checking = False` を設定していますが、これは「未知のホスト」を自動承認する設定であり、**今回のように鍵が変化したホストへの接続はブロックされたままです**（OpenSSHがman-in-the-middle対策としてパスワード認証自体を無効化するため）。`ansible-playbook`・`ansible`コマンドの実行前に、上記の `ssh-keygen -R` と手動SSH接続確認を必ず行ってください。
