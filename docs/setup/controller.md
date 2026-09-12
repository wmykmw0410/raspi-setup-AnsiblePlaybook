# Ansible インストール（実行元PC）

Ansible を実行するPCにAnsibleをインストールします。OSごとの手順は以下の通りです。

> 仮想環境（venv）の利用は任意です。他のPythonプロジェクトとパッケージを分離したい場合は、`pip install` の前に以下を実行してください（以降、`pip` / `ansible` コマンドを使うたびに `source` の実行が必要です）。
>
> ```bash
> python3 -m venv ~/.venv/ansible
> source ~/.venv/ansible/bin/activate    # Windows(WSL)は同じ、Windows(PowerShell等)は非対応のためWSL必須
> ```

## macOS

Homebrew 経由でのインストール（推奨）、または pip 経由でのインストールのいずれかを選択してください。

### Homebrew を使う場合

1. [Homebrew](https://brew.sh/) が未インストールの場合はインストールします。

   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

2. Ansible をインストールします。

   ```bash
   brew install ansible
   ```

3. インストールを確認します。

   ```bash
   ansible --version
   ```

### pip を使う場合

1. Python のバージョンを確認します（3.9 以上が必要）。未インストールの場合は [Homebrew](https://brew.sh/) 等でインストールします。

   ```bash
   python3 --version
   ```

2. Ansible をインストールします。

   ```bash
   pip3 install --upgrade pip
   pip3 install ansible
   ```

3. インストールを確認します。

   ```bash
   ansible --version
   ```

## Windows

Ansible の実行環境は Linux/macOS 前提のため、Windows では **WSL（Windows Subsystem for Linux）** を利用します。

1. PowerShell を管理者権限で開き、WSL と Ubuntu をインストールします。

   ```powershell
   wsl --install
   ```

   インストール後、PCを再起動しUbuntuの初回セットアップ（ユーザー名・パスワード設定）を行います。

2. WSL（Ubuntu）のターミナルを開き、パッケージを更新して Python / pip をインストールします。

   ```bash
   sudo apt update
   sudo apt install -y python3 python3-pip
   ```

3. Ansible をインストールします。

   ```bash
   pip3 install --upgrade pip
   pip3 install ansible
   ```

4. インストールを確認します。

   ```bash
   ansible --version
   ```

> 以降の `ansible` / `ansible-playbook` コマンドはすべて WSL（Ubuntu）のターミナル上で実行してください。

## Linux（Ubuntu/Debian系）

1. パッケージを更新して Python / pip をインストールします。

   ```bash
   sudo apt update
   sudo apt install -y python3 python3-pip
   ```

2. Ansible をインストールします。

   ```bash
   pip3 install --upgrade pip
   pip3 install ansible
   ```

3. インストールを確認します。

   ```bash
   ansible --version
   ```

バージョン情報が表示されればインストール完了です。
