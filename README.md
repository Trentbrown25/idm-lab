# Red Hat IdM (FreeIPA) Multi-Node Lab — AWS EC2

## Overview
A multi-node Red Hat IdM/FreeIPA lab built on AWS EC2 to practice centralized
identity management ahead of a Cloud Infrastructure Support Engineer role.
Covers server install, client enrollment (including a second client host),
user/group management, tiered role-based access via HBAC and sudo, password
policies, nested group inheritance, per-host access control, and multi-master
replication.

## Architecture
- **idm-server** (`t3.medium`, 10.0.2.87) — Directory Server (LDAP), Kerberos
  KDC, Dogtag CA, Apache. No integrated DNS; relies on `/etc/hosts`.
- **idm-replica** (`t3.medium`, 10.0.2.22) — full IdM server peer, promoted
  from a client via `ipa-replica-install`. Syncs the entire directory with
  idm-server (multi-master replication).
- **idm-client** (`t3.micro`, 10.0.2.159) — enrolled client, general access.
- **idm-client2** (`t3.micro`, 10.0.2.125) — enrolled client, restricted to
  sysadmins only (per-host HBAC scoping).
- Shared security group: internal IdM ports (53, 80, 88, 389, 464, 636, 443,
  7389) scoped to the security group itself; admin ports (22, 80, 443) also
  open to a single home IP for direct access.

## Identity & Access Model
| Group | Users | SSH access | Sudo access | Password policy |
|---|---|---|---|---|
| sysadmins | asmith | idm-client, idm-client2 | Full (`ALL` as root) | Stricter: 30-day max life, 12-char min, lockout after 3 failed attempts/60s |
| helpdesk | bwhite | idm-client only | None (denied at HBAC layer) | Global default |
| developers | jdoe, cgreen | idm-client only | Scoped (`systemctl`/`who` as root) | Global default (90-day, 8-char min) |
| all-staff (nested: sysadmins + developers) | — | — | Shared baseline (`who` as root) via inheritance | — |

## Setup Steps (Server)
1. Launch EC2 instance (AlmaLinux OS Foundation AMI — requires a one-time
   free Marketplace subscription), `t3.medium` minimum (see Troubleshooting).
2. Set hostname, populate `/etc/hosts`, open required security group ports.
3. `dnf install -y ipa-server` → `ipa-server-install` (no integrated DNS;
   run inside `tmux` to survive SSH disconnects during the CA/KDC setup).

## Setup Steps (Client)
1. `dnf install -y ipa-client` → `ipa-client-install --domain=... --server=...`
2. Verify with `id admin` from the client.

## Setup Steps (Replica)
1. Enroll the new host as a regular client first (steps above).
2. `dnf install -y ipa-server` (same package as the original server).
3. `ipa-replica-install` inside `tmux` — this promotes the client to a full
   server peer and begins directory replication.
4. Verify replication by creating a user on one server and confirming it
   appears on the other (`ipa user-show <user>` after `kinit admin` on the
   second server).

## Access Control Patterns Practiced
- **Per-group HBAC/sudo** — different groups, different SSH/sudo privileges.
- **Password policies** — group-specific `ipa pwpolicy-add` overriding the
  global default, with priority ordering.
- **Account lockout** — triggered via repeated failed logins, diagnosed with
  `ipa user-status` (not the vague SSH error), resolved with `ipa user-unlock`.
- **Nested/indirect group membership** — a parent group (`all-staff`)
  containing other groups; members inherit its rules without being added
  directly. Verified with `ipa user-show <user> --all | grep -i member`.
- **Per-host access control** — the same user pool, different rules per
  target host, so one client can be more restricted than another.
- **Multi-master replication** — two independent, synchronized directory
  servers.

## Troubleshooting Log

| Issue | Root Cause | Fix |
|---|---|---|
| `OptInRequired` on `run-instances` | AMI is an AWS Marketplace listing requiring one-time subscription | Subscribe via the Marketplace console link (free for official AlmaLinux OS Foundation AMI) |
| `UnsupportedOperation` on `run-instances` | `t2.micro` not supported by this Marketplace AMI | Use `t3.micro`/`t3.small`/`t3.medium` instead |
| `Unsupported` — instance type not available in AZ | Subnet was in an AZ that doesn't support the requested instance type | Create a subnet in a supported AZ, associate the route table with the internet gateway, enable auto-assign public IP |
| `ipa-server-install`: "Less than the minimum 1.2GB RAM" | `t3.micro`/`t3.small` insufficient for Directory Server + KDC + CA | Resize to `t3.medium` before installing |
| SSH connection drops mid-install | Sustained CPU load during cert/CA generation on a CPU-credit-limited instance | Run heavy installs (`ipa-server-install`, `ipa-replica-install`) inside `tmux` |
| `AssertionError: Another instance named 'X-Y' may already exist` | A crashed install left a partial Directory Server instance behind | `dsctl -l` to find it, `dsctl <instance> remove --do-it` to clean up before retrying; after 2+ failed attempts, prefer terminating and relaunching fresh |
| `ipa-client-install`: `Joining realm failed: JSON-RPC call failed: Timeout` | Port 443 was scoped to home IP only, not the security group | Add a rule opening 443 `--source-group` in addition to the home-IP rule |
| `ipa-replica-install` connection check fails on port 80 | Port 80 had the same home-IP-only scoping issue as 443 originally did | Open port 80 `--source-group` as well |
| `--hbacsvcs=a,b` or `--groups=a,b` silently does nothing (0 members added) | Comma-separated values in one flag are read as one literal string, not split | Repeat the flag once per value: `--hbacsvcs=a --hbacsvcs=b` |
| `su - user` → `Permission denied` on an expired/reset password | `su`'s PAM path doesn't handle "password expired" prompts the way `sshd`'s does | Use `ssh user@host` instead of `su - user` when a password change is pending |
| `sudo`: `PAM account management error: Permission denied` | HBAC allows `sshd` but not the `sudo` service on that rule | Add the service: `ipa hbacrule-add-service <rule> --hbacsvcs=sudo` |
| `sudo`: "user is not allowed to run sudo" (after HBAC fix) | Sudo rule has no allowed commands or RunAs user attached | `ipa sudorule-add-allow-command` and `ipa sudorule-add-runasuser` |
| Sudo/HBAC rule updated on server but client still denies | SSSD caches rules locally | `sudo sss_cache -E && sudo systemctl restart sssd` on the client |
| `su - user` → "Could not chdir to home directory" | `oddjob-mkhomedir` not installed/enabled | Deferred — install `oddjob-mkhomedir`, enable `oddjobd`, `authselect enable-feature with-mkhomedir` |
| `ipa-replica-manage list` fails with a DNS error even using `/etc/hosts` | This specific tool insists on real DNS resolution, unlike most `ipa`/Kerberos tooling | Skip the tool; check `/var/log/dirsrv/slapd-*/errors` and `ldapsearch` against `cn=mapping tree,cn=config` directly |
| Replication: initial sync succeeds, incremental sync fails with a GSSAPI decoding error ("Unable to parse the response to the startReplication extended operation") | Stale GSSAPI/Kerberos session state left over from the promotion process (not clock skew — verified via `chronyc tracking` on both sides first) | Restart Directory Server on both peers: `sudo systemctl restart dirsrv@<REALM>` |

## Verification Commands
```bash
# Server/replica health
sudo ipactl status
kinit admin && klist

# Client identity resolution
id admin

# HBAC test (no live login needed)
ipa hbactest --user=<user> --host=<host> --service=<sshd|sudo>

# Nested group inheritance
ipa user-show <user> --all | grep -i member

# Account lockout status and recovery
ipa user-status <user>
ipa user-unlock <user>

# Replication proof
# On server A:
ipa user-add repltest --first=Repl --last=Test --password
# On server B:
kinit admin
ipa user-show repltest
```

## Known Gaps / Next Time
- `oddjob-mkhomedir` never installed — home directories don't auto-create on
  clients.
- No integrated DNS configured — worth trying a DNS-integrated variant for
  comparison in a future lab.
- CA service only runs on idm-server, not idm-replica (`ipa-ca-install` would
  add CA redundancy — beyond scope of what was asked for this round).
