# keepalived & VRRP: what it is, what was already there, and how to install it

This is the companion explainer to [README.md](README.md) (which covers the
overall HA architecture and failover test results). This file answers three
things specifically: what `keepalived` and VRRP actually are, whether they
were already on these machines or got installed fresh, and how to reproduce
the install from scratch.

## Was any of this already installed?

Checked directly on both hosts via `rpm -qi <pkg>` (which prints an
`Install Date`):

| Package | 172.21.129.16 | 172.21.129.138 |
|---|---|---|
| `nginx` / `nginx-mod-stream` | **Already installed** — Fri 07 Aug 2026 (from earlier work in this lab, before the HA setup) | **Not installed** — installed fresh Tue 11 Aug 2026 as part of this task |
| `keepalived` | **Not installed** — installed fresh Tue 11 Aug 2026 | **Not installed** — installed fresh Tue 11 Aug 2026 |

So: nginx pre-existed only on 172.21.129.16 from earlier load-balancer work.
**`keepalived` was not on either machine before this** — it's a separate
package from nginx, with nothing to do with web serving; it had to be
installed on both hosts specifically to add VRRP/failover.

One thing worth knowing if you inspect the box yourself: RHEL's `keepalived`
RPM ships a *default example* `/etc/keepalived/keepalived.conf` on install
(referencing a placeholder `eth0` and example email addresses) — that's
standard package content, not anything specific to this setup. It gets
completely overwritten by the real config in this repo.

## What is keepalived?

`keepalived` is a Linux userspace daemon that does two independent things
(we only use the first):

1. **VRRP implementation** — lets a group of Linux boxes share one floating
   IP address, with automatic failover if the current holder becomes
   unhealthy. This is what we use it for.
2. **LVS/IPVS health-checking** — a Layer-4 load balancer/health-checker for
   `ipvsadm`-managed virtual services. Not used here; nginx is doing the
   actual SSH load balancing, keepalived is purely doing IP failover.

## What is VRRP?

VRRP = **Virtual Router Redundancy Protocol** (RFC 3768 for VRRPv2, which is
what's used here since `auth_type PASS` is a VRRPv2-only feature; RFC 5798
for VRRPv3, which dropped authentication).

The core idea: a group of routers/hosts agree to jointly own one **virtual
IP address**, identified by a shared `virtual_router_id` (0–255, must be
unique per subnet — we use `51`, chosen after confirming no other VRRP
traffic was already using it on this network). At any moment **exactly one**
member is **MASTER** and actually holds the IP; the rest are **BACKUP** and
stay silent/idle.

How the election and failover actually work, mechanically:

- VRRP runs directly over IP (protocol number **112** — not a TCP or UDP
  port), sent to multicast address **224.0.0.18**. This is why
  `firewall-cmd --add-protocol=vrrp` was needed on both hosts — it's not a
  normal port-based firewall rule.
- The MASTER sends a VRRP **advertisement** to that multicast group at a
  fixed interval (`advert_int 1`, i.e. once a second here). It's basically
  a heartbeat: "I'm still alive, my priority is X."
- Every BACKUP listens for those adverts. If it doesn't hear one for about
  **3× `advert_int`** (~3 seconds here), it assumes the MASTER is gone and
  promotes itself.
- **`priority`** (0–255) breaks ties / decides who should be MASTER when
  more than one candidate is eligible. We set `172.21.129.16` to `150` and
  `172.21.129.138` to `100`, so `.16` is preferred whenever it's healthy.
- **`preempt`** (the default, and what we use) means: if a higher-priority
  node comes back online while a lower-priority node currently holds the
  VIP, the higher-priority node takes it back. (The alternative,
  `nopreempt`, would leave `.138` holding the VIP even after `.16`
  recovers — we don't want that, since `.16` is meant to be "the main
  server.")
- The moment a node becomes MASTER, it assigns the virtual IP to its own
  network interface and sends a **Gratuitous ARP** — an unsolicited
  "this IP is now at my MAC address" broadcast — so every switch and host
  on the LAN updates its ARP cache immediately, instead of waiting for
  stale entries to time out. This is what makes failover fast (a few
  seconds) instead of slow.
- **`vrrp_script` / `track_script`** is a keepalived extension (not part of
  the VRRP protocol itself): it runs an arbitrary health-check script on an
  interval and adjusts this node's effective priority based on the result.
  We use it to check `pgrep -x nginx` — if nginx itself crashes (even though
  the box and network are fine), this node's priority drops below its
  peer's and the VIP moves. Without this, VRRP alone would only detect
  "the whole box/network died," not "nginx died but the box is still up."
- **`authentication` / `auth_pass`** is VRRPv2 simple-text authentication.
  It is *not* real security (unencrypted, and VRRPv3 dropped it entirely)
  — its only real purpose is to stop two unrelated, misconfigured VRRP
  groups from accidentally interfering with each other on the same subnet.

## Installing this from scratch (RHEL 8.8)

On **both** hosts:

```bash
# nginx with the stream (TCP/L4 proxy) module, + keepalived for VRRP
dnf install -y nginx nginx-mod-stream keepalived

# Allow nginx to bind an address before VRRP has actually brought it up
# locally (needed on whichever node is currently BACKUP)
cat > /etc/sysctl.d/99-nonlocal-bind.conf <<'EOF'
net.ipv4.ip_nonlocal_bind = 1
EOF
sysctl -p /etc/sysctl.d/99-nonlocal-bind.conf

# Open the firewall for VRRP (protocol 112, not a port)
firewall-cmd --permanent --add-protocol=vrrp
firewall-cmd --reload
```

Then deploy the per-host files from this repo to their real paths (see the
table in [README.md](README.md)):

- `nginx.conf`, `ssh-lb.conf` → nginx's config (identical LB policy on both
  hosts)
- `sshd_config` → note the explicit `ListenAddress <this host's real IP>`;
  a wildcard bind here would conflict with nginx's bind on the VIP
- `keepalived.conf`, `check_nginx.sh` → the VRRP config and its health check
- `keepalived-override.conf` → systemd drop-in so `keepalived` itself
  auto-restarts on crash (`Restart=on-failure`) — the packaged systemd unit
  does **not** do this by default, which we only discovered by deliberately
  crash-testing it (see README's failover tests)

SELinux (kept **Enforcing** throughout, not disabled):

```bash
# Let nginx (httpd_t) bind the SSH port — generated via audit2allow from a
# real AVC denial, see selinux/ssh_lb_nginx.te for the exact one-line rule
semodule -i selinux/ssh_lb_nginx.pp
setsebool -P httpd_can_network_connect on
```

Finally, bring keepalived up on both hosts:

```bash
systemctl daemon-reload   # picks up the Restart=on-failure override
systemctl enable --now keepalived
```

## Verifying it's working

```bash
# Which host currently holds the VIP?
ip -4 addr show eno1 | grep 172.21.129.200

# Watch VRRP state transitions live
journalctl -u keepalived -f

# Confirm the VIP actually answers SSH
ssh root@172.21.129.200
```

To actually test failover (what the README's test results are from):

```bash
# Graceful:
systemctl stop keepalived      # on whichever host currently holds the VIP

# Ungraceful (closer to a real crash):
kill -9 $(pgrep -f 'keepalived -D' | head -1)
```

Then watch the VIP reappear on the other host within a few seconds, and
confirm `ssh root@172.21.129.200` still works throughout.
