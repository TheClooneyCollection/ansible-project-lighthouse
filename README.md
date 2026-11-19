# Project Lighthouse

Ansible automation for rebuilding the Project Lighthouse droplet on Debian 13. The playbook bootstraps baseline tooling (shell preferences, deploy user) and now adds an `initialize` role that upgrades the OS, installs the remaining base packages, and ensures Docker is enabled for downstream services.

## Repository Layout
- `inventory/hosts.yml.template` – Template inventory for the `debian_lighthouse` group; copy to `inventory/hosts.yml` (ignored) for local customization.
- `group_vars/debian_lighthouse/main.yml.template` – Template for non-secret variables (domains, Certbot config, Umami defaults, swap sizing); copy to `main.yml` locally.
- `group_vars/debian_lighthouse/vault.yml.template` – Template for sensitive values; copy to `vault.yml`, fill in secrets, then encrypt with Ansible Vault.
- `roles/initialize/` – Packages/dist-upgrade/Docker setup for the new node.
- `roles/core/`, `roles/deploy_user/` – Existing roles that manage CLI tooling and the deploy account.
- `roles/firewall/`, `roles/swapfile/`, `roles/locales/` – Network hardening, optional swap management, and locale activation.
- `roles/nginx/`, `roles/v2ray/`, `roles/certbot/`, `roles/umami_nginx/` – Web/proxy foundation to be extended with site-specific configs.
- `ansible-node-plan.md` – Work plan & checklist as the remaining roles are implemented.
- `main.yml` – Entry-point play that will be extended with the new roles as they come online.
- `collections/requirements.yml` – Galaxy collections required for modules used in the roles.
- `scripts/install-ansible.sh` – Convenience installer for Ansible and required collections on the controller.

> Note: `.gitignore` excludes the live copies (`inventory/hosts.yml`, `group_vars/debian_lighthouse/main.yml`, `group_vars/debian_lighthouse/vault.yml`) so you can tweak local configuration without committing secrets.

## Requirements
- Ansible 2.15+ (controller machine).
- SSH access to the target host (root or a sudo-capable user). Remote-access tooling such as Tailscale must be installed and authenticated manually before running the playbook:
  ```bash
  curl -fsSL https://tailscale.com/install.sh | sh
  sudo tailscale up
  ```
- Python available on the target (default on Debian 13).
- `ansible-lint` for static checks (install via `pip install -r requirements-dev.txt` or `pipx install ansible-lint`).
- Required collections (auto-installed by `scripts/install-ansible.sh`, or run `ansible-galaxy collection install -r collections/requirements.yml`).

## Before You Run the Playbook
1. **Create local config from templates** (if not already present):
   ```bash
   cp inventory/hosts.yml.template inventory/hosts.yml
   cp group_vars/debian_lighthouse/main.yml.template group_vars/debian_lighthouse/main.yml
   cp group_vars/debian_lighthouse/vault.yml.template group_vars/debian_lighthouse/vault.yml
   ```
2. **Inventory** – Edit `inventory/hosts.yml` with the real droplet IP/hostname and SSH user.
3. **Group vars** – Adjust values in `group_vars/debian_lighthouse/main.yml` to match your domains, email, access allowlist, etc.
4. **Secrets** – Fill out `group_vars/debian_lighthouse/vault.yml` with real passwords/salts, then encrypt it:
   ```bash
   ansible-vault encrypt group_vars/debian_lighthouse/vault.yml
   ```
5. **SSH access** – Confirm you can log in non-interactively (SSH key or appropriate auth) because the playbook connects over SSH.
6. **Collections** – Install required collections if you haven’t already:
   ```bash
   ansible-galaxy collection install -r collections/requirements.yml
   ```

## Running the Playbook
Run the full playbook against the `debian_lighthouse` group:
```bash
ansible-playbook -i inventory/hosts.yml main.yml --limit debian_lighthouse
```

If you encrypted `vault.yml`, supply the password at runtime:
```bash
ansible-playbook -i inventory/hosts.yml main.yml --limit debian_lighthouse --ask-vault-pass
```

By default the play gathers facts and runs the `core`, `deploy_user`, `initialize`, `locales`, `firewall`, `swapfile`, `nginx`, `v2ray`, `certbot`, and `umami_nginx` roles. Use `--tags` or `--skip-tags` as needed during development.
To run only the `initialize` role during development:
```bash
ansible-playbook -i inventory/hosts.yml main.yml --limit debian_lighthouse --tags initialize
```

## Private EDControlRoom market snapshots

The `edcr_eddn_listener` role installs an hourly snapshot timer alongside the listener.
It writes `/var/lib/edcontrolroom/snapshots` as `edcontrolroom` (directory 0755,
files 0644 with `UMask=0022`) and keeps the newest three gzip snapshots. Override
`edcr_eddn_listener_snapshot_dir`, `edcr_eddn_listener_snapshot_keep`, or
`edcr_eddn_listener_snapshot_schedule` in inventory variables as needed.
Deploy only after EDControlRoom main includes `market_snapshot.py write --keep`.
The listener now uses shipped defaults without requiring a comment-only `config.toml`;
any existing config file is left in place.

Apply just the listener/snapshot units and the separate directory-serving instance:

```bash
ansible-playbook -i inventory/hosts.yml main.yml --limit debian_lighthouse --tags edcr_eddn_listener,tailscale_serve_edcr_markets --check --diff
ansible-playbook -i inventory/hosts.yml main.yml --limit debian_lighthouse --tags edcr_eddn_listener,tailscale_serve_edcr_markets
```

This also updates the EDControlRoom checkout/dependencies and may restart its listener.
Only the snapshot timer is enabled/started; the snapshot service is not explicitly
started during deployment. `Persistent=true` may catch up a missed scheduled run.
For the first snapshot, run `sudo systemctl start edcr-market-snapshot.service` on
the host after deploying. No reboot is required.

The `tailscale_serve_edcr_markets` tag can subsequently apply just directory serving.
Its port is `edcr_eddn_listener_snapshot_https_port` (default 8443); choose an unused
port distinct from Umami's 443. The new `tailscale-serve-edcr-markets` unit uses
`tailscale serve --bg --https=<port> <snapshot_dir>` and stops only its own port.
The existing Umami unit stays in foreground mode with its existing settings.
Tailscale must already be authenticated with HTTPS enabled. Directory targets are
supported by [Tailscale Serve](https://tailscale.com/docs/reference/tailscale-cli/serve).
No public listener, Funnel, nginx site, or firewall change is added.

Configure the hub privately with `[eddn] snapshot_url` pointing to
`https://<vps-tailnet-name>:<port>/manifest.json`; keep the actual tailnet name out of
this repository. Check the URL from a tailnet device and inspect `tailscale serve
status`, the snapshot timer, file permissions and public listeners on the host.
These live checks are separate from local syntax/lint validation.

## Linting & Validation
Install tooling:
```bash
pip install -r requirements-dev.txt
```

Run the Ansible linter from the repo root:
```bash
./scripts/lint.sh
```

Pass extra paths or flags as needed, e.g. `./scripts/lint.sh roles/initialize`.

## Next Steps
Follow `ansible-node-plan.md` for the outstanding work items (firewall, swapfile, web stack, Umami services, backups, and documentation updates). Each completed milestone should be checked off in that file and reflected in this README.
