# portainer-stack

[English version here](README.md)

VPS 上に Docker / Docker Compose / Portainer を使ったコンテナホスティング環境を構築する Ansible 定義集です。高価なクラウドサービスに依存せず、趣味プロジェクトをリリースできます。

[portainer-stack-deploy](https://github.com/carlrygart/portainer-stack-deploy) と組み合わせることで、GitHub Actions からのデプロイや Git ベースのリリースが可能になります。

> **Dev Container 対応**  
> このリポジトリは [Dev Containers](https://containers.dev/) を使用しています。  
> VS Code + Dev Containers 拡張機能でコンテナを起動し、その中から操作してください。

## セットアップ内容

| コンポーネント | 説明 |
|---|---|
| Docker CE | コンテナランタイム + Compose プラグイン |
| Portainer CE | コンテナ・スタックを管理する Web UI |
| cloudflared | (オプション) ポートを公開せずに安全にアクセスするための Cloudflare Tunnel |

## セットアップ

### 1. 接続情報の設定

対話形式で `.env` を生成します。

```shell
make setup
```

Setup 実行中、セキュリティ設定を適用するか設問されます:

```
⚠️  This project stores sensitive credentials in .env.
    To prevent AI tools (GitHub Copilot, etc.) from reading them,
    it is strongly recommended to disable AI features in VS Code settings
    and remove AI skill/agent definition files from this workspace.

Apply these security settings? [y/N]:
```

`y` を選択すると、以下が自動で実行されます:

- `.vscode/settings.json` に以下を追加:
  ```json
  "github.copilot.enable": { "*": false },
  "chat.disableAIFeatures": true
  ```
- `.github/agents/` と `.github/skills/portainer-stack/SKILL.md` を削除

> **なぜ必要か?** `.env` には SSH 秘密鍵パス、sudo パスワード、Cloudflare トークンなどの機密情報が含まれます。このワークスペースでの AI 機能を無効化することで、これらの値が AI ツールに読み取られるリスクを予防します。

生成される `.env` の内容:

```shell
# .env
HOST=xxx.xxx.xxx.xxx
USER=ubuntu
PORT=22
KEY=/root/.ssh/your-key.pem

ENABLE_TUNNEL=false

SUDO_PASS=your-sudo-password
```

> 秘密鍵のパーミッションエラーが出る場合は以下を実行してください:
> ```shell
> chmod 600 /root/.ssh/your-key.pem
> ```

### 2. Ansible プロビジョニングの実行

```shell
make setup-ansible
```

リモート VPS に Docker CE をインストールし、Portainer CE をデプロイします。

完了後、以下の URL から Portainer にアクセスできます:

```
https://<VPS の IP アドレス>:9443
```

初回ログイン時に管理者パスワードを設定してください。

> **デプロイしたアプリへのアクセスについて**  
> Portainer 自体はポート 9443 で直接アクセスできます。ただし、スタックとしてデプロイしたアプリケーションへのトラフィックルーティングは、公開方法によって異なります:
>
> | 方式 | 説明 |
> |---|---|
> | **Cloudflare Tunnel (Public Hostname)** | Cloudflare Zero Trust でアプリごとにホスト名を登録します。Cloudflare がリクエストをコンテナに直接転送するため、リバースプロキシは不要です。|
> | **Cloudflare Public Hostname を使わない場合** | VPS 上にリバースプロキシ（Nginx / Traefik / Caddy など）を立て、ホスト名またはパスに応じて各コンテナへ転送する実装が必要です。|

## ファイル転送

### rsync

```shell
# アップロード
make rsync ARGS="-av ./local/path/ {user}@{host}:/remote/path/"

# ダウンロード
make rsync ARGS="-av {user}@{host}:/remote/path/ ./local/path/"
```

### SFTP

```shell
make sftp ARGS="{user}@{host}"
```

## オプション: Cloudflare Tunnel 経由での接続

Cloudflare Tunnel を使うことで、VPS のポートをインターネットに直接公開せずに接続できます。

### 前提条件

- Cloudflare Zero Trust でトンネルが作成済みであること
- トンネルの Public Hostname に SSH 用のホスト名が登録済みであること
- Dev Container に `cloudflared` がインストール済みであること

### 設定

`make setup` 実行時に Cloudflare Tunnel の有効化を選択すると、以下の変数が `.env` に追加されます。

```shell
ENABLE_TUNNEL=true
SUBDOMAIN=subdomain
DOMAIN=example.com
CF_TUNNEL_TOKEN=your-cloudflare-tunnel-token
```

`ENABLE_TUNNEL=true` に設定すると、`make ssh` / `make rsync` / `make sftp` はすべて自動的に Cloudflare Tunnel 経由で動作します。

### Cloudflare Tunnel も Ansible でプロビジョニングする

VPS 上に `cloudflared` もインストールする場合は、`.env` に `ENABLE_TUNNEL=true` を設定してから実行:

```shell
make setup-ansible
```

## GitHub Actions によるデプロイ

[portainer-stack-deploy](https://github.com/carlrygart/portainer-stack-deploy) を使って GitHub Actions からスタックをデプロイできます:

```yaml
- name: Deploy stack to Portainer
  uses: carlrygart/portainer-stack-deploy@v1
  with:
    portainer-host: ${{ secrets.PORTAINER_HOST }}
    username: ${{ secrets.PORTAINER_USERNAME }}
    password: ${{ secrets.PORTAINER_PASSWORD }}
    stack-name: 'my-app'
    stack-definition: 'docker-compose.yml'
    image: ${{ env.DOCKER_IMAGE_URI }}:${{ env.IMAGE_TAG }}
```

## Ansible プロビジョニング詳細

### ドライラン

```shell
make setup-ansible ARGS="--check --diff"
```

### 特定ホストのみを対象にする

```shell
make setup-ansible ARGS="--limit target"
```

### Ansible Vault（オプション）

シークレットを `.env` に書く代わりに、Ansible Vault で管理することもできます。

```shell
ansible-vault create ansible/vault/secrets.yml
```

Vault 認証付きで実行:

```shell
make setup-ansible ARGS="--ask-vault-pass"
# または
make setup-ansible ARGS="--vault-password-file /path/to/.vault_pass"
```
