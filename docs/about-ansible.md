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
| 変数（Variables） | ホストやグループごとに異なる値を注入する仕組み。このリポジトリでは `inventory/group_vars/*.yml` で定義しています（詳細は [変数リファレンス](variables.md)）。 |
| become | 対象ホスト上で管理者権限（sudo）に昇格して実行するオプション。Playbook内で `become: true` と指定します。 |
| ハンドラー（Handler） | タスクの変更をトリガーに実行される処理（例: 設定ファイル変更後の `smbd` 再起動）。`nas_server` ロールで使用しています。 |

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

## よく使うコマンド

| コマンド | 用途 |
|---|---|
| `ansible <host> -i <inventory> -m ping` | 対象ホストへのSSH接続・Python実行可否を確認する疎通確認（`-m` で指定したモジュールを1回実行するアドホックコマンド） |
| `ansible-playbook -i <inventory> <playbook>` | Playbookを実行する |
| `ansible-playbook ... --limit <host>` | 対象ホストを絞り込む |
| `ansible-playbook ... -e "key=value"` | 変数を実行時に上書きする（インベントリファイルの値より優先されます） |
| `ansible-playbook ... --check` | 実際には変更を適用せず、変更予定の内容だけを確認する（ドライラン） |

具体的な実行例は [README](../README.md#playbook-の実行) を参照してください。
