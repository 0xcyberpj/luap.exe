+++
title = "packet aegis: phishing-resistant playbook"
date = 2025-11-26T02:15:00-07:00
draft = false
tags = ["open-source", "security"]
listIcon = "/contribute.png"
teaser = "Step-by-step template for rolling out hardware-backed auth with tidy logs and repeatable commands."
+++

## 0. why this exists

I needed a single document that blends narrative, commands, and checklists so I can repeat the rollout without thinking. Use this as a ready-to-publish blog post or strip the sections you don’t need.

---

## 1. baseline

- **fleet:** 42 engineers on macOS, 8 on Linux.
- **idp:** self-hosted Authentik with WebAuthn + TOTP.
- **goal:** hardware key + SSH CA with automatic revocation.

```
┌─IDP──┬──────────┐
│authn │ yubikey  │
└──────┴──────────┘
     │
┌────▼────┐    ┌──────────┐
│ssh ca   │───▶│ bastions │
└─────────┘    └──────────┘
```

---

## 2. rollout diary

### day 1 — inventory + dry run

```bash
webauthn credential list --json | jq '.[] | {user, created}'
```

- exported existing credentials into `/var/aegis/ledger.json`.
- scheduled office hours with hardware tokens + printed recovery cards.

### day 2 — ssh ca bootstrap

```bash
ssh-keygen -f /var/aegis/ssh_ca -a 256 -t ed25519 -C "aegis-ca"
cat /var/aegis/ssh_ca.pub | tee /etc/ssh/ca.pub
```

- appended `TrustedUserCAKeys /etc/ssh/ca.pub` to every bastion.
- built a tiny signer script that tags certificates with `purpose=production` and a 12-hour lifetime.

### day 3 — enforcement + comms

- flipped the IdP policy: WebAuthn OR hardware OTP, passwords disabled.
- rotated all bastion user entries to `AuthenticationMethods publickey`.
- sent the template below to every team so future updates look professional.

---

## 3. status email template

```
subject: [aegis] hardware auth rollout status D3

hi team,

- 34/50 keys enrolled, remaining slots scheduled tomorrow.
- ssh ca certs minted: 112, oldest expires in 11h.
- incident tracker remains clear.

todo:
- wipe unused keys after 7 days.
- finish Linux agent install (eta tomorrow 10:00).

— luap.exe
```

---

## 4. verification checklist

- [ ] Run `ssh -oIdentitiesOnly=yes -i signed-cert user@bastion` from a new laptop.
- [ ] Revoke a key via `webauthn credential revoke --user <id>`.
- [ ] Confirm the audit trail posts to `#aegis-feed` Slack channel.

---

## 5. appendix — reusable signer

```bash
#!/usr/bin/env bash
set -euo pipefail
user="$1"
ssh-keygen -s /var/aegis/ssh_ca -I "$user-$(date +%s)" \
  -n "$user" -V +12h \
  -O extension:purpose=production \
  "$HOME/.ssh/id_ed25519.pub"
```

Paste this snippet whenever you need to reissue certs. Replace `/var/aegis` with your own path and commit the script to `infra/security`.

