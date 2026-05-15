# ansible-echoshield

Ansible role for bulk-deploying [Echoshield](https://github.com/ec-pur/echoshield) FPV-drone detector firmware onto a fleet of Raspberry Pi 5 nodes from a BYO Deployment Kit (`.tar.gz` shipped by the operator).

| Status                             | What this role does                                                                            |
| ---------------------------------- | ---------------------------------------------------------------------------------------------- |
| Per-Pi setup time                  | ~30 s after the kit is staged on the control host (vs ~5 min interactive `install-pi.sh`)      |
| Idempotency                        | A Pi with `[license].token` in `/etc/echoshield/echoshield.toml` is detected and skipped       |
| Hardware binding                   | Default `--with-fingerprint` ON — claim is bound to Pi serial + LibreSDR serial per Phase H7   |
| Internet on each Pi                | **Only at install** — afterwards the Pi runs airgapped                                         |

## Quick start

1. Receive the Deployment Kit from the operator (`echoshield-kit.tar.gz`).
2. Verify and extract:

   ```bash
   sha256sum echoshield-kit.tar.gz   # must match the operator's published hash
   mkdir -p ~/echoshield-kit && tar xzf echoshield-kit.tar.gz -C ~/echoshield-kit
   ```

3. Write an inventory mapping each Pi hostname/IP to one **unused** row of `claim-codes.csv`:

   ```ini
   [echoshield_fleet]
   pi-01.local claim_code=9PEP36NFATGE85CUGJVLNK
   pi-02.local claim_code=A1B2C3D4E5F6G7H8J9K0LM
   pi-03.local claim_code=ZXC9V8B7N6M5L4K3J2H1G0
   ```

4. Run the playbook:

   ```bash
   ansible-galaxy install -r requirements.yml   # installs this role + any deps
   ansible-playbook -i inventory.ini deploy.yml
   ```

5. Watch the per-host summary box stream in — each Pi prints its device-ID, local UI URL, and "internet no longer required" footer when enrolment completes.

Failures? See the [airgap troubleshooting Q&A](https://github.com/ec-pur/echoshield/blob/master/docs/runbooks/customer/airgap-troubleshooting.md#ansible-failures) in the main repo.

## Example playbook

```yaml
# deploy.yml
- hosts: echoshield_fleet
  become: true
  gather_facts: true
  vars:
    echoshield_kit_local_path: "{{ playbook_dir }}/echoshield-kit"
    echoshield_api_url: "https://164-92-233-43.nip.io"
    echoshield_magic_token: "{{ lookup('env', 'ECHOSHIELD_MAGIC_TOKEN') }}"
  roles:
    - role: ansible-echoshield
```

## Role variables

| Variable                          | Default                                | Notes                                                                       |
| --------------------------------- | -------------------------------------- | --------------------------------------------------------------------------- |
| `echoshield_kit_local_path`       | _(required)_                           | Path on the control host to the unpacked kit dir                            |
| `echoshield_kit_remote_path`      | `/var/cache/echoshield/kit`            | Where the role uploads the kit on each Pi                                   |
| `echoshield_api_url`              | _(required)_                           | Fleet backend base URL                                                      |
| `echoshield_magic_token`          | `""`                                   | Magic token from the install URL; empty allowed for direct-issue flows      |
| `echoshield_with_fingerprint`     | `true`                                 | Bind claim to SHA-256(Pi serial) + SHA-256(LibreSDR serial)                  |
| `echoshield_with_web`             | `true`                                 | Install `echoshield-web` (local UI on `:8443`)                              |
| `echoshield_with_bastion`         | `false`                                | Install `echoshield-bastion` (reverse-SSH tunnel for operator support)      |
| `echoshield_kit_already_present`  | `false`                                | Skip kit upload when the Pi already has it (golden-image pre-stage)          |
| `echoshield_force_os_check`       | `false`                                | Pass `ECHOSHIELD_FORCE=1` — allows non-Bookworm/non-arm64 hosts (debug)     |
| `claim_code`                      | _(per-host inventory var)_             | One **unused** row from `claim-codes.csv`                                   |

## Requirements on the Pi

- RaspiOS Bookworm or Trixie, 64-bit (arm64)
- ≥ 4 GB free on `/`
- Outbound HTTPS to `echoshield_api_url` (one-shot, during the claim task only)
- NTP synced (the role's pre-task waits up to 60 s for sync)
- SSH access for the user named in your inventory's `ansible_user`

## Requirements on the control host

- Ansible Core ≥ 2.16
- Python 3.11+ on the control host (for templating)
- The unpacked kit at `echoshield_kit_local_path`

## Idempotency contract

The role re-runs cleanly on the same fleet:

- A Pi that already has `[license].token` in `/etc/echoshield/echoshield.toml` is **skipped** (no apt invocation, no claim attempt).
- The same `claim_code` is rejected on a second Pi with HTTP 409 — pick the next unused row.
- Repeated runs against the same Pi with the same claim code produce no apt churn.

## Tags

| Tag           | Effect                                                  |
| ------------- | ------------------------------------------------------- |
| `prereq`      | Install apt prerequisites + ensure NTP sync             |
| `kit`         | Upload kit tarball + extract on the Pi                  |
| `install`     | Run `install-pi.sh` Phase 1-4                           |
| `verify`      | Run Phase 5 (poll for JWT, post enrollment-complete)    |
| `inspection`  | Read-only — collect device-ID + journalctl for support  |

Example: re-run only the verify task on the whole fleet after a manual fix:

```bash
ansible-playbook -i inventory.ini deploy.yml --tags verify
```

## License

Apache-2.0 — see [`LICENSE`](LICENSE).

## Support

- Operational issues: [Echoshield wiki](https://github.com/ec-pur/echoshield/wiki) (separate `echoshield.wiki.git` repo)
- Bugs in this role: [open an issue](https://github.com/ec-pur/ansible-echoshield/issues) on this repo
- Contact: `echoshield-ops@hlybchenko.dev`
