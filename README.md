# Red Hat IdM (FreeIPA) Two-Node Lab — AWS EC2

## Overview
A two-node Red Hat IdM/FreeIPA lab built on AWS EC2 to practice centralized
identity management ahead of a Cloud Infrastructure Support Engineer role.
Covers server install, client enrollment, user/group management, Host-Based
Access Control (HBAC), and centralized sudo policy.

## Architecture
- **idm-server** — AlmaLinux OS 9.8, `t3.medium`, runs Directory Server (LDAP),
  Kerberos KDC, Dogtag CA, and Apache (no integrated DNS — relies on `/etc/hosts`).
- **idm-client** — AlmaLinux OS 9.8, `t3.micro`, enrolled against idm-server.
- Same VPC, custom subnet, shared security group scoping internal IdM ports
  (53, 88, 389, 464, 636, 443) to the security group itself, and admin ports
  (22, 80, 443) to a single home IP.

## Setup Steps
1. Launch two EC2 instances (AlmaLinux 9, official AlmaLinux OS Foundation AMI)
   in the same VPC/subnet.
2. Set real hostnames (`hostnamectl set-hostname`) and populate `/etc/hosts`
   on both boxes with each other's private IPs.
3. Open required security group ports (see Architecture above).
4. `dnf install -y ipa-server` → `ipa-server-install` on idm-server (no
   integrated DNS).
5. `dnf install -y ipa-client` → `ipa-client-install --domain=... --server=...`
   on idm-client.
6. Create users/groups: `ipa user-add`, `ipa group-add`, `ipa group-add-member`.
7. HBAC: `ipa hbacrule-add`, scope by user group/host/service, `ipa hbactest`
   to verify.
8. Sudo: `ipa sudorule-add`, attach commands and RunAs user, verify with
   `sudo -l` / a live sudo attempt.

## Troubleshooting Log

| Issue | Root Cause | Fix |
|---|---|---|
| `OptInRequired` on `run-instances` | AMI is an AWS Marketplace listing requiring one-time subscription | Subscribe via the Marketplace console link (free for official AlmaLinux OS Foundation AMI) |
| `UnsupportedOperation` on `run-instances` | `t2.micro` not supported by this Marketplace AMI | Use `t3.micro`/`t3.small`/`t3.medium` instead |
| `Unsupported` — instance type not available in AZ | Subnet was in `us-east-1e`, which doesn't support the requested instance type | Create a new subnet in a supported AZ (`us-east-1a`), associate it with the route table that has the internet gateway route, enable auto-assign public IP |
| SSH `Permission denied (publickey...)` | Wrong or missing `.pem` file path in the `ssh -i` command | Locate the correct `.pem` file; always pass the full path |
| `ipa-server-install`: "Less than the minimum 1.2GB RAM" | `t3.micro`/`t3.small` insufficient for Directory Server + KDC + CA | Resize to `t3.medium` before installing |
| SSH connection drops mid-install (`client_loop: send disconnect`) | Sustained CPU load during cert/CA generation on a CPU-credit-limited instance can cause network hiccups; foreground install has no resilience to a dropped session | Run `ipa-server-install` inside `tmux` so the process survives disconnects; consider a burstable-credit check (`CPUCreditBalance` in CloudWatch) before heavy installs |
| `AssertionError: Another instance named 'LAB-LOCAL' may already exist` | A crashed install left a partial Directory Server instance behind; `ipactl status` alone doesn't catch this | `dsctl -l` to find it, `dsctl <instance> remove --do-it` to clean it up before retrying |
| Repeated partial-install debris across multiple failed attempts | Manual cleanup after a crash doesn't always fully reset state | When in doubt after 2+ failed installs, terminate and relaunch a fresh instance — faster and more reliable than chasing leftover state |
| `ipa-client-install`: `Joining realm failed: JSON-RPC call failed: Timeout` | Port 443 was scoped to home IP only, not to the security group — client couldn't reach the server's API over HTTPS | Add a security group rule opening 443 `--source-group` (in addition to the home-IP rule for browser access) |
| `su - jdoe` → `Permission denied` on a freshly reset/expired password | `su`'s PAM path doesn't always handle "password expired" prompts the way `sshd`'s does | Use `ssh user@host` instead of `su - user` when a password change is pending |
| `sudo`: `PAM account management error: Permission denied` | HBAC only allowed the `sshd` service, not `sudo` — sudo itself is gated by HBAC | Add the `sudo` service to the relevant HBAC rule: `ipa hbacrule-add-service <rule> --hbacsvcs=sudo` |
| `sudo`: "jdoe is not allowed to run sudo on \<host\>" (after HBAC fix) | Sudo rule existed but had no allowed commands or RunAs user attached | `ipa sudorule-add-allow-command` and `ipa sudorule-add-runasuser` |
| Sudo rule updated on server but client still denies | SSSD caches sudo rules locally and doesn't always pick up changes immediately | `sudo sss_cache -E && sudo systemctl restart sssd` on the client |
| `su - jdoe` → "Could not chdir to home directory" | `oddjob-mkhomedir` not installed/enabled, so IdM never created a local home directory on the client | Deferred — install `oddjob-mkhomedir`, enable `oddjobd`, run `authselect enable-feature with-mkhomedir` |

## Verification Commands
```bash
# Server health
sudo ipactl status
kinit admin && klist

# Client identity resolution
id admin

# HBAC test (no live login needed)
ipa hbactest --user=jdoe --host=idm-client.lab.local --service=sshd
ipa hbactest --user=jdoe --host=idm-client.lab.local --service=sudo

# Sudo scoping test (from idm-client, logged in as jdoe)
sudo systemctl status sshd   # should succeed
sudo whoami                  # should be denied
```

## Known Gaps / Next Time
- `oddjob-mkhomedir` never installed — home directories don't auto-create on
  the client. Fix and document properly on the next repeat of this lab.
- No integrated DNS was configured (deliberate, to save time) — worth trying
  a DNS-integrated variant in a future lab for comparison.
