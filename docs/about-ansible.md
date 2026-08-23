# Ansibleとは

Ansible は、サーバーやPCなどの構成管理・自動化を行うオープンソースのツールです。Red Hat（IBM）が開発・メンテナンスしており、クラウドサーバーからオンプレミス機器、今回のようなRaspberry Piまで幅広い対象を管理できます。

## 特徴

- **エージェントレス**: 対象端末に専用ソフトウェアを事前インストールする必要がなく、SSHで接続できればセットアップ可能。Python がインストールされていれば追加の常駐プロセスは不要です。
- **宣言的な設定（Playbook）**: 「どうやるか（手順）」ではなく「どうあるべきか（状態）」をYAML形式の Playbook に記述します。同じサーバーを手順書通りに手作業で構築するより、設定内容そのものをコードとして管理できます（Infrastructure as Code）。
- **冪等性（べきとうせい）**: 同じ Playbook を何度実行しても、既に設定済みの項目はスキップされ、常に同じ状態に収束します。「既にインストール済みか」を毎回自分で確認する必要がありません。

## 基本用語

| 用語 | 説明 |
|---|---|
| インベントリ（Inventory） | 管理対象ホストの一覧。このリポジトリでは `inventory/*.ini` が該当し、`[client]`・`[nas]` のようなグループでホストを分類します。 |
| Playbook | 「どのホストに」「どのロールを」適用するかを記述するYAMLファイル。このリポジトリの `playbooks/*.yml` が該当します。 |
| ロール（Role） | 特定の設定作業（例: VS Codeのインストール、日本語ロケール設定）をひとまとめにした再利用可能な単位。`roles/` 配下の各ディレクトリが1つのロールです。 |
| タスク（Task） | ロール内の個々の作業単位。「パッケージをインストールする」「ファイルを配置する」など、モジュールを1回呼び出す処理に対応します。 |
| モジュール（Module） | タスクが実際に呼び出す処理の実体。`ansible.builtin.apt`（パッケージ管理）、`ansible.builtin.copy`（ファイル配置）、`ansible.builtin.systemd`（サービス管理）など、目的別に多数用意されています。 |
| 変数（Variables） | ホストやグループごとに異なる値を注入する仕組み。このリポジトリでは `inventory/group_vars/*.yml`（全ホスト・グループ共通）、`inventory/site_vars/<拠点>.yml`（拠点別）、`.ini` ファイル内のホスト変数（`static_ip` 等）で定義しています（詳細は [変数リファレンス](variables.md)）。 |
| become | 対象ホスト上で管理者権限（sudo）に昇格して実行するオプション。Playbook内で `become: true` と指定します。 |
| ハンドラー（Handler） | タスクの変更をトリガーに実行される処理（例: 設定ファイル変更後の `smbd` 再起動）。`nas_server` ロールで使用しています。 |
| Vault | パスワードなどの秘密情報をファイルに暗号化したまま保存する仕組み。このリポジトリでは `ansible_password` 等を `ansible-vault encrypt_string` で暗号化しており、実行時に `--ask-vault-pass` でVaultパスワードを入力します。 |

## フォルダ構成の基本形

上記の用語が実際にどうディレクトリへ対応するかの例です（Ansibleプロジェクトの一般的な構成）。

```
project/
├── ansible.cfg              # Ansible の動作設定（このリポジトリでは host_key_checking 等を設定）
├── inventory/                # インベントリ
│   ├── <拠点>.ini             # ホスト一覧・グループ定義（[client]・[nas] など）
│   └── group_vars/
│       ├── all.yml            # 全ホスト共通変数
│       └── <グループ名>.yml    # グループ別変数
├── playbooks/                 # Playbook（どのホストにどのロールを適用するか）
│   └── <用途>.yml
└── roles/
    └── <ロール名>/
        ├── tasks/main.yml      # タスク定義（エントリーポイント）
        ├── handlers/main.yml   # ハンドラー定義
        ├── templates/          # Jinja2テンプレート（変数展開が必要なファイル）
        ├── files/               # そのまま配置する静的ファイル
        └── defaults/main.yml    # ロールのデフォルト変数
```

- `templates/` と `files/` はどちらも「配布するファイル」を置く場所ですが、`templates/` はJinja2記法（`{{ 変数 }}` など）で内容を動的に変えられるのに対し、`files/` は変数展開せずそのままコピーする点が異なります。
- `defaults/main.yml` はロール単位のデフォルト値で、`inventory/group_vars/` の変数より優先度が低く設定されています（`group_vars` で上書きされます）。

このリポジトリの実際のディレクトリ構成は [README のファイル構成](../README.md#ファイル構成) を参照してください。

## Playbook の構造（このリポジトリの例）

```yaml
- name: 共通セットアップ
  hosts: all
  become: true

  roles:
    - base
    - locale
    - keyboard
    # ...
```

- `hosts:` — インベントリ内のどのグループ（またはホスト）に適用するかを指定します。`all` は全ホスト、`client` / `nas` はグループ名です。
- `become:` — `true` の場合、タスクを sudo 権限で実行します。パッケージインストールやシステム設定変更にはほぼ必須です。
- `roles:` — 適用するロールを順番に列挙します。上から順に実行されます。

`playbooks/client.yml` や `playbooks/nas.yml` は `import_playbook: common.yml` を使って `common.yml` の内容を先頭で取り込み、その後にクライアント/NAS固有のロールを追加する構成になっています。

## モジュール（Module）とは

タスクが実際に呼び出す処理の実体です。「パッケージをインストールする」「ファイルを配置する」といった1つの操作をコマンドラインの代わりに担当し、多くのモジュールは**べき等**です。つまり、対象の状態を先に確認し、既に目的の状態であれば何もしない（`changed: false`）、まだ違えば変更する（`changed: true`）という動きをします。シェルスクリプトのように毎回同じコマンドを実行するのではなく、「目的の状態」を宣言するだけで済むのはこの仕組みのおかげです。

タスク内では `<コレクション名>.<モジュール名>:` の形式で指定し、その下にモジュールごとのパラメータを書きます。

```yaml
- name: git をインストールする
  ansible.builtin.apt:
    name: git
    state: present
```

このリポジトリで実際によく使われているモジュールです。

| モジュール | 用途 |
|---|---|
| `ansible.builtin.apt` | apt パッケージのインストール・削除・更新 |
| `ansible.builtin.copy` / `ansible.builtin.template` | ファイルの配置（`template` はJinja2で変数展開しながら配置） |
| `ansible.builtin.file` | ディレクトリ作成・パーミッション変更などファイル状態の管理 |
| `ansible.builtin.user` | システムユーザーの作成・グループ所属の変更 |
| `ansible.builtin.service` / `ansible.builtin.systemd` | サービスの起動・停止・自動起動設定 |
| `ansible.builtin.lineinfile` / `ansible.builtin.replace` | 既存ファイルの一部（1行、または正規表現に一致する箇所）だけを書き換え |
| `ansible.builtin.mount` | マウント・`/etc/fstab` への登録 |
| `ansible.builtin.reboot` | 対象ホストの再起動（再起動完了まで待機） |
| `community.general.nmcli` | NetworkManagerの接続プロファイル管理（`static_ip` ロールでの固定IP設定） |
| `ansible.builtin.command` / `ansible.builtin.shell` | 任意のコマンドを直接実行する「何でも屋」。対応する専用モジュールがない場合の最終手段で、べき等性は保証されないため `changed_when` / `failed_when` で挙動を自分で制御する必要があります |

対応するモジュールが存在しない操作（今回だと `nmcli`・`arping`・`ip addr` の呼び出しなど）は `command`/`shell` モジュールで代替しています。その場合、実行結果を `register` で受け取り、`changed_when: false`（状態を変えない確認コマンド）や `when` 条件（重複実行の回避）と組み合わせて、擬似的にべき等な振る舞いにするのがこのリポジトリの基本パターンです（例: [roles/static_ip/tasks/set_ip.yml](../roles/static_ip/tasks/set_ip.yml)）。

各モジュールの詳細なパラメータは `ansible-doc <モジュール名>`（例: `ansible-doc ansible.builtin.apt`）のほか、以下の公式リファレンスでも確認できます。

- [Ansible builtin モジュール一覧](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html)（`ansible.builtin.*`。今回使用している `apt` / `copy` / `template` / `file` / `user` / `service` / `lineinfile` / `mount` / `reboot` / `command` 等はすべてここに含まれます）
- [community.general モジュール一覧](https://docs.ansible.com/ansible/latest/collections/community/general/index.html)（`community.general.nmcli` はこちら）

## その他のAnsible機能（このリポジトリでは未採用）

以下はAnsibleの一般的な機能で、知っておくと便利ですが、このリポジトリでは現時点で使用していません（理由も記載）。

### block / rescue / always

`tasks:` の代わりに `block:` でタスクをまとめ、`rescue:`（失敗時のみ実行）・`always:`（成否に関わらず実行）を組み合わせられます。プログラミング言語のtry/catch/finallyに近い書き方です。

```yaml
- block:
    - name: 何らかの処理
      ansible.builtin.command: some_command
  rescue:
    - name: 失敗時のリカバリ処理
      ansible.builtin.debug:
        msg: "失敗したのでリカバリします"
  always:
    - name: 成否に関わらず必ず実行する処理
      ansible.builtin.debug:
        msg: "後片付け"
```

> このリポジトリの [playbooks/static_ip.yml](../playbooks/static_ip.yml) は `when` 条件や `register` の組み合わせでエラー処理をしており、`block`/`rescue`を使えばもう少し整理できる可能性がありますが、現状の実装で動作確認済みのため書き換えは行っていません。

### ansible-pull

通常のAnsibleは制御用PCから対象ホストへ「push」する方式ですが、`ansible-pull` は対象ホスト自身がgitリポジトリを取得してPlaybookを自分に適用する「pull」方式です。対象ホスト側で`cron`等に登録すれば、制御用PCなしで定期的に設定を最新化できます。

```bash
# ラズパイ側で実行するイメージ
ansible-pull -U <gitリポジトリURL> playbooks/client.yml
```

> このリポジトリでは、`playbooks/static_ip.yml`（DHCPで割り振られたIPを人が目視確認して`ansible_host`に記載する前提）や、Vaultパスワードの対話入力が必須な構成になっているため、`cron`のような無人実行が前提の`ansible-pull`とは相性が良くありません。採用していません。

### カスタムファクト（`/etc/ansible/facts.d/`）

対象ホスト上に `.fact` ファイル（JSON/INI/実行可能スクリプト）を置いておくと、`ansible_facts.ansible_local` としてPlaybookから参照できる、ホスト側に永続的な独自情報を持たせる仕組みです。

```ini
; /etc/ansible/facts.d/example.fact
[general]
provisioned_at = 2026-01-01
```

```yaml
- name: カスタムファクトを表示する
  ansible.builtin.debug:
    var: ansible_local.example.general.provisioned_at
```

> このリポジトリでは、ホスト固有の情報は `inventory/*.ini`・`inventory/site_vars/` 側（コントローラー側）で管理しており、ラズパイ側に情報を持たせる具体的な必要性がまだないため未採用です。

## よく使うコマンド

| コマンド | 用途 |
|---|---|
| `ansible <host> -i <inventory> -m ping --ask-vault-pass` | 対象ホストへのSSH接続・Python実行可否を確認する疎通確認（`-m` で指定したモジュールを1回実行するアドホックコマンド） |
| `ansible-playbook -i <inventory> <playbook> --ask-vault-pass` | Playbookを実行する |
| `ansible-playbook ... --limit <host>` | 対象ホストを絞り込む |
| `ansible-playbook ... --tags <タグ名>` | 指定したロールだけ実行する |
| `ansible-playbook ... -e "key=value"` | 変数を実行時に上書きする（インベントリファイルの値より優先されます） |
| `ansible-playbook ... --check` | 実際には変更を適用せず、変更予定の内容だけを確認する（ドライラン） |
| `ansible-vault encrypt_string '値' --name '変数名' --ask-vault-pass` | 値をVault化する（詳細は [README](../README.md#playbook-の実行)） |

このリポジトリではパスワード類がVault化されているため、上記の `ansible`・`ansible-playbook` コマンドには基本的に `--ask-vault-pass` が必要です。

具体的な実行例は [README](../README.md#playbook-の実行) を参照してください。
