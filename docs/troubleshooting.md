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

なお、このリポジトリの `ansible.cfg` は `host_key_checking = False` を設定しているため、Ansible自体はホストキー不一致でも実行できます。ただし疎通確認・パスワード認証の動作確認を兼ねて、`ansible-playbook` の前に一度手動SSHで接続できることを確認することを推奨します。
