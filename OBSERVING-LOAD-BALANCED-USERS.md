# Observing and Verifying SSH Load-Balanced Sessions (this repo's architecture)

This repo's architecture has the floating VIP `172.21.129.200` and
keepalived/VRRP HA — nginx runs on **both** `172.21.129.16` and
`172.21.129.138`, but only one is actually reachable at a time. Users
connect with `ssh user@172.21.129.200`. The LB policy here is
**max_conns=5 + backup** (fill primary to 5, then overflow) — not
round-robin.

The general methodology is identical to the fuller writeup in
[nginx_round-robin's OBSERVING-LOAD-BALANCED-USERS.md](https://github.com/dev1237/nginx_round-robin/blob/main/OBSERVING-LOAD-BALANCED-USERS.md)
(a fully worked 10-user example with real captured output for every method,
against the *same* HA architecture — only the LB algorithm differs between
that repo and this one, which doesn't change any of the observation
commands below). Read that one for the complete walkthrough and the two
caveats that apply here unchanged (PTY needed for `who`/`w`; backend never
sees the true client IP).

## Extra step specific to HA repos: find out which node is actually active first

Since either physical box could be the one actually serving traffic at any
given moment, check this before running anything else — it tells you which
node's `stream_access.log` is the live one, and which node is currently
"primary-active" vs. standing by:

```bash
# run on both nodes; whichever prints the VIP is the active one
ip -4 addr show eno1 | grep 172.21.129.200

# or, more directly:
systemctl status keepalived | grep -i state
journalctl -u keepalived --no-pager -n 5
```

## Commands specific to this repo's policy

```bash
# Live connection count on the primary -- always 172.21.129.16:22 here,
# regardless of which node is currently running the active nginx
ssh root@172.21.129.16 "ss -tn state established '( sport = :22 )' | tail -n +2 | wc -l"

# Live connection count on the overflow backend
ssh root@172.21.129.138 "ss -tn state established '( sport = :22 )' | tail -n +2 | wc -l"

# Per-user live view on whichever host you're checking (needs -t -t client-side)
who
w
```

## Real captured result (from this repo's own re-verification test)

8 concurrent SSH connections through the VIP `172.21.129.200`, read from the
active node's `stream_access.log`:

```
upstream_addr=172.21.129.138:22   (overflow)
upstream_addr=172.21.129.138:22   (overflow)
upstream_addr=172.21.129.16:22    (primary)
upstream_addr=172.21.129.16:22    (primary)
upstream_addr=172.21.129.16:22    (primary)
upstream_addr=172.21.129.16:22    (primary)
upstream_addr=172.21.129.16:22    (primary)
upstream_addr=172.21.129.138:22   (overflow)
```

5 on the primary, 3 overflowed — same policy as `nginx_updated`, confirmed
working identically even with HA added on top (adding failover to the LB
tier doesn't change the connection-limiting behavior at all, since the two
concerns are independent — see this repo's main README).

**Note on scope:** as with `nginx_updated`, this specific test verified
counts via the nginx log, not a per-username `who`/`w` snapshot or auth-log
excerpt for *this* max_conns+backup policy specifically. Those commands are
identical regardless of LB algorithm (they only care about which backend a
connection landed on, not how it was chosen), so for real captured
per-username output, see `nginx_round-robin`'s guide — the only difference
you'd see running it here is the aggregate split trending 5:3 instead of
roughly even.
