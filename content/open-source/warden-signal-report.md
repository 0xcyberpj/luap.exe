+++
title = "warden signal report: november perimeter audit"
date = 2025-11-25T05:45:00-07:00
draft = false
tags = ["open-source", "security"]
listIcon = "/contribute.png"
teaser = "Three-day walkthrough of how I hardened the perimeter sensors, complete with snippets, logs, and an action log you can reuse."
+++

## 0. briefing

I spent three nights tracing the edge of my homelab the way a watchman walks a fence. This report is the template I wish existed—narrative up top, raw logs in the middle, clear actions at the end. Feel free to drop in your own environment variables and ship it as a status update.

---

## 1. env sketch

- **Scope:** wifi mgmt VLAN, two bastion hosts, redis cache, rf canary mesh.
- **Goal:** verify alarms still trigger when an unexpected client joins, and confirm cache/tunnel credentials rotate on schedule.
- **Time box:** 72 hours (Nov 22–24).

### network map (ascii)

```
[internet]─┬─[edge fw]──[bastion-01]
           └─[mesh gw]──{ rf-canaries }
                        └─[redis-cache]──[services/*]
```

---

## 2. signals walkthrough

### 2.1 wifi onboarding probe

```
nmcli dev wifi connect "$SENTRY_AP" password "$SENTRY_PASS" ifname mon0
```

- Expectation: IDS should page within 30s when it sees the monitoring NIC connect with the “probe” MAC.
- Reality: alert fired at 27s, payload included RSSI + channel, matching spec.

### 2.2 redis credential rotation

```bash
redis-cli --user auditor --pass "$AUDIT_PASS" info keyspace | rg expires
```

- Keys with TTL < 1h dropped from 14 → 5 after we moved secrets to Vault.
- Added a sanity check script (see appendix) that runs nightly and slacks me if any key lives longer than it should.

### 2.3 rf canary packet flood

```
python tools/rf/noiseburst.py --duration 45s --pattern zigzag --power -3dbm
```

- Canaries reported saturation but only the mesh gateway filed a ticket.
- Root cause: webhook to the pager queue silently 410’d. Fixed by rotating API token and adding `curl -f` to the cron wrapper so failures aren’t swallowed.

---

## 3. logbook (copy/paste friendly)

| timestamp (UTC) | system | event | action |
|-----------------|--------|-------|--------|
| 2025-11-22 04:12 | ids-gamma | probe MAC joined mgmt | alerted, auto-quarantined |
| 2025-11-22 19:05 | redis-cache | ttl drift 42% | scheduled flush, patched writer |
| 2025-11-23 09:44 | rf-canary-04 | rssid spike -55 | adjusted antenna, logged |
| 2025-11-24 01:02 | mesh-gw | webhook 410 | rotated token, added `curl -f` |

---

## 4. appendix a — scripts

### ttl-notify.sh

```bash
#!/usr/bin/env bash
redis-cli --scan | while read key; do
  ttl=$(redis-cli ttl "$key")
  if [ "$ttl" -gt 0 ] && [ "$ttl" -gt 86400 ]; then
    printf "%s ttl=%ss\n" "$key" "$ttl"
  fi
done | tee /tmp/ttl-report.txt

if [ -s /tmp/ttl-report.txt ]; then
  ntfy send "$NTFY_TOPIC" < /tmp/ttl-report.txt
fi
```

### webhook-sanity.sh

```bash
#!/usr/bin/env bash
set -euo pipefail
payload='{"ping":"webhook sanity"}'
curl -fsS -X POST -H "Authorization: Bearer $CANARY_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$payload" "$CANARY_WEBHOOK" \
  || echo "webhook down" | ntfy send "$NTFY_TOPIC"
```

---

## 5. appendix b — template for future runs

Copy these headings into a new file anytime you need a quick report:

```
## 0. briefing
## 1. env sketch
### network map
## 2. signals walkthrough
### 2.1 (test name)
## 3. logbook
## 4. appendix a — scripts/logs
## 5. appendix b — template notes
```

---

## 6. actions

- [x] Rotate webhook token, add failure surfacing.
- [x] Automate TTL drift check + nightly Slack message.
- [ ] Build a proper dashboard once I trust the notebook again. Until then, these markdown reports stay the source of truth.

