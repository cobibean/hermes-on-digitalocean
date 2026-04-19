# hermes-on-digitalocean

A Claude Code / agent-skills compatible skill for deploying a [Hermes Agent](https://github.com/NousResearch/hermes-agent) to a DigitalOcean droplet so it runs 24/7 with messaging-platform access.

Covers:
- Provisioning via `doctl`
- Python 3.11 bootstrap on Ubuntu 24.04 (without deadsnakes)
- Non-interactive Hermes install
- Profile creation + workspace-repo symlink pattern
- Credential migration from a laptop install
- OAuth callback caveats (Codex)
- Gateway as a systemd service with linger

The skill is grounded in a real deploy — every gotcha listed actually happened.

## Use

Drop `SKILL.md` into your skills directory (`~/.hermes/skills/` for Hermes, or wherever your agent harness loads skills from).

## License

MIT
