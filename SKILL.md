---
name: hermes-on-digitalocean
description: Step-by-step guide for deploying a Hermes Agent (by Nous Research) to a DigitalOcean droplet so it runs 24/7 with messaging-platform access (Telegram, Discord, Slack, etc.). Covers provisioning via doctl, Python 3.11 bootstrap on Ubuntu 24.04, non-interactive Hermes install, profile setup, credential migration from a laptop, OAuth caveats, and gateway-as-systemd-service. Load this skill when a user wants to move a Hermes agent off their laptop onto a cheap always-on cloud host.
version: 1.0.0
author: cobibean
license: MIT
---

# Deploy a Hermes Agent to a DigitalOcean droplet

[Hermes Agent](https://github.com/NousResearch/hermes-agent) is a self-improving CLI/messaging AI agent by Nous Research. It belongs on a small always-on box — not your laptop. This skill walks through provisioning a $12/mo droplet, installing Hermes non-interactively, migrating credentials from a laptop install, and running the messaging gateway as a systemd service that survives logout.

**Prereqs on your laptop:**
- A working local Hermes install (`hermes --version`)
- A configured profile with the messaging platform you want (`hermes -p <name> gateway setup`)
- `doctl` installed and authenticated to a DigitalOcean account
- `gh` (only if your workspace repo is private and the droplet needs to clone it)

## Pick specs

- **`s-1vcpu-2gb` ($12/mo):** the right floor. 1GB works only if you disable the `browser` toolset and never run anything heavy.
- **Ubuntu 24.04 LTS:** stable, but ships Python 3.12. Hermes needs 3.11 — `uv` bootstraps it cleanly (see below).
- **Region:** pick the one nearest your messaging users (Telegram users hit DO's Frankfurt or NYC POPs fine).

## 1. Provision the droplet

```bash
# Generate a project-scoped SSH key
ssh-keygen -t ed25519 -f ~/.ssh/hermes_droplet_ed25519 -N "" -C "hermes-droplet"

# Upload to DO
KEY_ID=$(doctl compute ssh-key import hermes-droplet-key \
  --public-key-file ~/.ssh/hermes_droplet_ed25519.pub \
  --format ID --no-header)

# Create droplet
doctl compute droplet create hermes \
  --image ubuntu-24-04-x64 \
  --size s-1vcpu-2gb \
  --region nyc3 \
  --ssh-keys $KEY_ID \
  --tag-names hermes-agent \
  --enable-monitoring \
  --wait \
  --format ID,Name,PublicIPv4,Status --no-header
```

Add an SSH alias to `~/.ssh/config`:
```
Host hermes
  HostName <droplet-ip>
  User root
  IdentityFile ~/.ssh/hermes_droplet_ed25519
  IdentitiesOnly yes
  StrictHostKeyChecking accept-new
```

## 2. Wait for cloud-init, install prereqs, install Hermes

Ubuntu 24.04 holds the apt lock for ~60s on first boot (cloud-init). Wait it out:

```bash
ssh hermes '
  cloud-init status --wait
  export DEBIAN_FRONTEND=noninteractive
  apt-get update -qq
  apt-get install -y -qq curl git build-essential ripgrep ffmpeg ca-certificates

  # uv bootstraps Python 3.11 without needing the deadsnakes PPA
  curl -LsSf https://astral.sh/uv/install.sh | sh
  export PATH="/root/.local/bin:$PATH"
  uv python install 3.11

  # Hermes installer (--skip-setup because there is no TTY)
  curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh \
    | bash -s -- --skip-setup
'
```

The installer pulls Playwright Chromium (~165MB) regardless. On 2GB you can keep it; on 1GB disable the `browser` toolset after install:

```bash
ssh hermes 'hermes tools disable browser'
```

## 3. Create the profile + (optionally) wire a workspace repo

If you're running multiple agents or want the agent's SOUL/AGENTS files version-controlled, use a profile + symlinks pattern:

```bash
ssh hermes '
  hermes profile create my-agent
  # If you have a workspace repo for the agent (recommended):
  mkdir -p ~/DEV && cd ~/DEV
  git clone git@github.com:<you>/<repo>.git workspace
  rm -f ~/.hermes/profiles/my-agent/SOUL.md
  ln -s ~/DEV/workspace/SOUL.md ~/.hermes/profiles/my-agent/SOUL.md
  ln -s ~/DEV/workspace/AGENTS.md ~/.hermes/profiles/my-agent/AGENTS.md
'
```

For a private repo, generate a deploy key on the droplet and add it to the repo:

```bash
ssh hermes '
  ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -N "" -C "hermes-deploy"
  cat ~/.ssh/github_deploy.pub
  printf "Host github.com\n  IdentityFile ~/.ssh/github_deploy\n  IdentitiesOnly yes\n  StrictHostKeyChecking accept-new\n" > ~/.ssh/config
  chmod 600 ~/.ssh/config
'
# Then add the printed pubkey as a deploy key (write access if the agent should commit back):
gh repo deploy-key add /dev/stdin --repo <you>/<repo> --title "hermes-droplet" --allow-write <<< "<pubkey>"
```

## 4. Migrate credentials from your laptop

Hermes splits credentials across **two** files. Both must move:

```bash
# Per-profile config + secrets (Telegram bot token, etc.)
scp ~/.hermes/profiles/<profile>/config.yaml hermes:/root/.hermes/profiles/<profile>/config.yaml
scp ~/.hermes/profiles/<profile>/.env        hermes:/root/.hermes/profiles/<profile>/.env

# Account-wide OAuth credentials (Nous Portal, Codex, Qwen) — outside the profile
scp -p ~/.hermes/auth.json                    hermes:/root/.hermes/auth.json

ssh hermes 'chmod 600 ~/.hermes/profiles/<profile>/.env ~/.hermes/auth.json'
```

Then fix any laptop-specific paths in the profile's config:

```bash
ssh hermes 'hermes -p <profile> config set messaging.working_directory /root/DEV/workspace'
```

### OAuth provider caveat

Some providers (notably **OpenAI Codex**) use PKCE OAuth with a localhost callback, not device code. On a headless droplet, a fresh `hermes auth add openai-codex --type oauth` will hang waiting for a callback that can never reach you. Two workarounds:

- **Copy the laptop's `auth.json`** (above) — refresh tokens are usually portable.
- **SSH port-forward** the callback port (Codex defaults to ~1455):
  ```bash
  ssh -L 1455:localhost:1455 hermes 'hermes -p <profile> auth add openai-codex --type oauth --timeout 600'
  ```
  Open the printed URL on your laptop; the OAuth redirect to `localhost:1455` tunnels back to the droplet.

API-key providers (Anthropic, OpenRouter, Gemini, etc.) just need their key in `.env` — no OAuth dance.

## 5. Install gateway as a systemd service

```bash
ssh hermes 'hermes -p <profile> gateway install && hermes -p <profile> gateway start'
```

The service is **profile-namespaced** — name is `hermes-gateway-<profile>.service`, not `hermes-gateway.service`. `gateway install` also enables systemd linger automatically (`loginctl enable-linger root`), so the service survives SSH logout.

Verify:
```bash
ssh hermes 'systemctl --user is-active hermes-gateway-<profile>'
ssh hermes 'journalctl --user -u hermes-gateway-<profile> --no-pager -n 30'
```

Then message your bot. Round-trip = success.

## 6. Scrub the laptop (if applicable)

If you don't want the laptop ever competing with the droplet for messages — only one process can hold a given Telegram bot token at a time — strip the platform credentials from the laptop's profile:

```bash
sed -i.bak '/^TELEGRAM_BOT_TOKEN=/d; /^TELEGRAM_ALLOWED_USERS=/d; /^TELEGRAM_HOME_CHANNEL=/d' \
  ~/.hermes/profiles/<profile>/.env
```

Keep the `.bak` (gitignored) for rollback.

## Day-2 ops cheatsheet

| Task | Command |
|---|---|
| Tail gateway logs | `ssh hermes 'journalctl --user -u hermes-gateway-<profile> -f'` |
| Restart after config change | `ssh hermes 'systemctl --user restart hermes-gateway-<profile>'` |
| Update Hermes | `ssh hermes 'hermes update && systemctl --user restart hermes-gateway-<profile>'` |
| Push workspace changes | `git push` (laptop) → `ssh hermes 'cd ~/DEV/workspace && git pull'` |
| Cost check | `doctl compute droplet list --format Name,Size,PriceMonthly --no-header` |

## Gotchas (real ones, hit during a real deploy)

- **`doctl auth init` requires a TTY** — fails in non-interactive shells with `unknown terminal`. Write `~/Library/Application Support/doctl/config.yaml` directly to bypass.
- **Ubuntu 24.04 has no `python3.11` package.** Use `uv python install 3.11` — don't waste time on the deadsnakes PPA.
- **Cloud-init holds apt for ~60s** post-boot. Always `cloud-init status --wait` before `apt-get`.
- **The setup wizard always needs a TTY** — pass `--skip-setup` to the installer for unattended installs.
- **`hermes login` is removed** — replaced by `hermes auth add <provider>`.
- **OAuth callback hang** — see the Codex note above. Symptom: `hermes auth add ... --no-browser` prints nothing and hangs.
- **`pkill` over SSH can drop your connection** (exit 255). The kill usually succeeds anyway; reconnect and verify with `ps`.
- **Service name is profile-namespaced.** `systemctl --user status hermes-gateway` returns "Unit not found" — use `hermes-gateway-<profile>.service`.

## Why this is worth doing

A Hermes agent on a $12/mo droplet costs ~40¢/day for hosting. It runs 24/7, accepts messages from Telegram/Discord/Slack while your laptop is closed, runs cron jobs unattended, and accumulates persistent memory across sessions. The same setup on serverless would idle near zero but adds cold-start latency to every message. For an always-responsive assistant, a small always-on droplet is the better default.

## References

- Hermes docs: https://hermes-agent.nousresearch.com/docs/
- Hermes repo: https://github.com/NousResearch/hermes-agent
- DO doctl docs: https://docs.digitalocean.com/reference/doctl/
- Skills standard: https://agentskills.io
