# portainer-stack

[日本語版はこちら](README.ja.md)

An Ansible-based setup for a VPS container hosting environment using Docker, Docker Compose, and Portainer. Deploy and manage hobby projects without relying on expensive cloud platforms.

By combining this with [portainer-stack-deploy](https://github.com/carlrygart/portainer-stack-deploy), you can trigger deployments from GitHub Actions — enabling Git-based releases while avoiding cloud cost overruns.

> **Dev Container ready**  
> This repository uses [Dev Containers](https://containers.dev/).  
> Open it in VS Code with the Dev Containers extension and run all local commands from inside the container.

## What this sets up

| Component | Description |
|-----------|-------------|
| Docker CE | Container runtime + Compose plugin |
| Portainer CE | Web UI for managing containers and stacks |
| cloudflared | (Optional) Cloudflare Tunnel for secure access without exposing ports |

## Setup

### 1. Configure connection settings

Generate `.env` interactively.

```shell
make setup
```

During setup, you will be asked whether to apply security settings:

```
⚠️  This project stores sensitive credentials in .env.
    To prevent AI tools (GitHub Copilot, etc.) from reading them,
    it is strongly recommended to disable AI features in VS Code settings
    and remove AI skill/agent definition files from this workspace.

Apply these security settings? [y/N]:
```

If you answer `y`, the following changes are applied automatically:

- Adds to `.vscode/settings.json`:
  ```json
  "github.copilot.enable": { "*": false },
  "chat.disableAIFeatures": true
  ```
- Deletes `.github/agents/` and `.github/skills/portainer-stack/SKILL.md`

> **Why?** The `.env` file contains sensitive credentials such as SSH private key paths, sudo passwords, and Cloudflare tokens. Disabling AI assistance in this workspace prevents these values from being inadvertently read or suggested by AI tools.

The generated `.env` looks like:

```shell
# .env
HOST=xxx.xxx.xxx.xxx
USER=ubuntu
PORT=22
KEY=/root/.ssh/your-key.pem

ENABLE_TUNNEL=false

SUDO_PASS=your-sudo-password
```

> If you get a permission error for the private key, run:
> ```shell
> chmod 600 /root/.ssh/your-key.pem
> ```

### 2. Run Ansible provisioning

```shell
make setup-ansible
```

This installs Docker CE and deploys Portainer CE on the remote VPS.

Once complete, access Portainer at:

```
https://<YOUR_VPS_IP>:9443
```

Set the admin password on first login.

> **Note: Accessing deployed applications**  
> Portainer itself is reachable directly on port 9443. However, for applications deployed as stacks, traffic routing from the VPS to each container depends on how you expose them:
>
> | Approach | Description |
> |---|---|
> | **Cloudflare Tunnel (Public Hostname)** | Register each app's hostname in Cloudflare Zero Trust. Cloudflare routes requests directly to the container — no reverse proxy needed. |
> | **Without Cloudflare Public Hostname** | You must run a reverse proxy (e.g. Nginx, Traefik, Caddy) on the VPS to forward incoming requests to the appropriate container by hostname or path. |

## File transfer

### rsync

```shell
# Upload
make rsync ARGS="-av ./local/path/ {user}@{host}:/remote/path/"

# Download
make rsync ARGS="-av {user}@{host}:/remote/path/ ./local/path/"
```

### SFTP

```shell
make sftp ARGS="{user}@{host}"
```

## Optional: Cloudflare Tunnel

Using Cloudflare Tunnel, you can connect to the VPS without exposing ports directly to the internet.

### Prerequisites

- A Cloudflare Zero Trust tunnel has been created
- The SSH hostname is registered as a Public Hostname on the tunnel
- `cloudflared` is installed inside the Dev Container

### Configuration

When running `make setup`, select yes to enable Cloudflare Tunnel. The following variables will be added to `.env`.

```shell
ENABLE_TUNNEL=true
SUBDOMAIN=subdomain
DOMAIN=example.com
CF_TUNNEL_TOKEN=your-cloudflare-tunnel-token
```

With `ENABLE_TUNNEL=true`, all commands (`make ssh`, `make rsync`, `make sftp`) automatically route through Cloudflare Tunnel.

### Ansible provisioning with Cloudflare Tunnel

To also install `cloudflared` on the VPS via Ansible, set `ENABLE_TUNNEL=true` in `.env` before running:

```shell
make setup-ansible
```

## GitHub Actions deployment

Use [portainer-stack-deploy](https://github.com/carlrygart/portainer-stack-deploy) to deploy stacks from GitHub Actions:

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

## Ansible Provisioning

### Dry-run

```shell
make setup-ansible ARGS="--check --diff"
```

### Limit to a specific host

```shell
make setup-ansible ARGS="--limit target"
```

### Ansible Vault (optional)

Instead of storing secrets in `.env`, you can manage them via Ansible Vault.

```shell
# Create an encrypted vault file from the example
ansible-vault create ansible/vault/secrets.yml
```

Then run with vault authentication:

```shell
make setup-ansible ARGS="--ask-vault-pass"
# or
make setup-ansible ARGS="--vault-password-file /path/to/.vault_pass"
```
