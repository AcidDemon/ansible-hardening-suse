# openSUSE Hardening (Ansible)

Security-hardening layer for openSUSE Leap hosts. Sibling of
`acidnetworks.hardening_debian` — same role set, same tiered var contract, same
systemd-timer rollback dead-man's-switch — ported to zypper / firewalld / SUSE PAM.

Packaged as the `acidnetworks.hardening_suse` collection. OS identity, chrony
enablement and zypper auto-updates live in `acidnetworks.base_suse`; this layer
owns the security posture.

## Roles

| Role | What it does |
|------|--------------|
| `ssh_hardening` | `/etc/ssh/sshd_config.d/10-hardening.conf` drop-in, `sshd -t`-validated; ensures the `Include` line and the `ssh_allow_groups` group |
| `ssh_2fa` | TOTP/FIDO2 via `pam_google_authenticator.so` + a `20-2fa.conf` sshd drop-in (SUSE `pam-config` stack) |
| `accounts` | pwquality, faillock, login.defs, `%wheel` sudoers hardening drop-in |
| `sysctl_hardening` | `/etc/sysctl.d/99-hardening.conf` (layers over base_suse's `90-baseline.conf`) |
| `hardening_misc` | coredump/journald/limits/tmp.mount/hidepid/cron.allow + optional msmtp relay |
| `auditd` | `audit` package, `rules.d/hardening.rules`, `augenrules --load` |
| `aide` | `aide --init` on a single `/etc/aide.conf` + a nightly `aidecheck` timer |
| `lynis` | installs Lynis and runs `lynis audit system` |
| `rootkit_scanners` | rkhunter + chkrootkit with SUSE (`/etc/sysconfig`) scheduling |
| `fail2ban` | `fail2ban` with a firewalld banaction (`firewallcmd-rich-rules`), `backend=systemd` |
| `crowdsec` | CrowdSec agent + firewall-bouncer from the packagecloud RPM repo |
| `security_reporting` | logwatch + a nightly security-report mail (optional) |
| `time_sync` | owns `/etc/chrony.conf` (base_suse enables `chronyd`) |

## Usage

Consume via `requirements.yml`:

```yaml
collections:
  - name: git+ssh://git@github.com/AcidDemon/ansible-hardening-suse.git
    type: git
    version: v0.1.0
```

Put hosts in the `hardened_hosts` group and, as the final play of your `site.yml`:

```yaml
- import_playbook: acidnetworks.hardening_suse.harden
```

Run order is **base_suse → hardening_suse → app**. The `harden` play arms a
`systemd-run` rollback timer, applies SSH changes, then probes connectivity
before disarming — a lockout auto-reverts after `harden_rollback_minutes`.
