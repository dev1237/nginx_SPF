# Highly-Available SSH Load Balancing (no single point of failure)

Follow-up to [nginx_updated](https://github.com/dev1237/nginx_updated), which
put a single nginx instance on 172.21.129.16 in front of SSH for both hosts.
That worked, but nginx (and the whole box it ran on) was a single point of
failure: if 172.21.129.16 went down, the entire SSH entry point went down
with it, even though 172.21.129.138 was perfectly healthy.

This repo removes that SPOF: **nginx now runs on both hosts**, and a
**floating virtual IP (VIP)**, moved between them by `keepalived` (VRRP),
is what users actually connect to. Users still type exactly one address —
they just don't know, or need to know, which physical box currently answers
it.

```
                      ssh root@172.21.129.200   <-- the ONE address users use
                                |
                    (VRRP elects exactly one owner)
                                |
              +-----------------------------------+
              |                                    |
   +--------------------+              +--------------------+
   | 172.21.129.16       |              | 172.21.129.138      |
   | nginx (listens only |   VRRP heart |  nginx (listens only|
   | on VIP:22)          |<--beat/prio->|  on VIP:22)          |
   | keepalived: MASTER  |   (proto 112,|  keepalived: BACKUP  |
   | priority 150        |   multicast) |  priority 100        |
   +--------------------+              +--------------------+
              |                                    |
              v  (proxy_pass, SAME policy on both) v
   upstream ssh_pool { primary=172.21.129.16:22 max_conns=5,
                        backup=172.21.129.138:22 }
```

Only ONE of the two nginx instances is actually reachable at any moment —
whichever node currently holds `172.21.129.200`. The other sits idle,
ready to take over. Which node owns the VIP is a completely separate
question from which backend gets priority for connections (still always
172.21.129.16, max 5, per the original requirement) — that load-balancing
policy is identical in both nginx configs and doesn't change based on
where nginx happens to be running.

## Why this design

- **`keepalived` / VRRP** is the standard Linux mechanism for a floating IP.
  Exactly one node is MASTER at a time and owns the VIP; the other is
  BACKUP. They exchange VRRP advertisements (IP protocol 112, multicast
  `224.0.0.18`) roughly once a second; if the MASTER stops advertising,
  BACKUP promotes itself after a short timeout.
- **`vrrp_script` health check** (`check_nginx.sh`, just `pgrep -x nginx`)
  is tied into the VRRP instance via `track_script`. If nginx itself dies on
  the MASTER (even if the box is otherwise fine), its priority drops below
  the BACKUP's and the VIP moves — this is what actually fixes "nginx might
  run off due to some error", not just "the box is unreachable".
  - We first tried checking `systemctl is-active nginx`, but SELinux's
    `keepalived_t` domain is deliberately not allowed to even `getattr`
    `systemctl` (a keepalived script controlling arbitrary systemd units is
    a meaningfully bigger attack surface than we needed). `pgrep -x nginx`
    needs no such access and checks the thing we actually care about.
- **172.21.129.16 stays preferred primary** (`priority 150` vs `138`'s
  `100`, with `preempt` — the default). This matches the original
  requirement that .16 is "the main server". `.138` only holds the VIP
  while `.16` (or its nginx) is unavailable, and hands it back the moment
  `.16` recovers.
- **nginx binds only the VIP, not `0.0.0.0`.** This is what makes running
  nginx on both nodes possible at all without port conflicts: each node's
  real sshd keeps using its own real IP on port 22 (`172.21.129.16:22`,
  `172.21.129.138:22`), and nginx separately owns `172.21.129.200:22`. This
  actually *simplifies* the original design, which had moved 172.21.129.16's
  sshd to a loopback-only `127.0.0.1:2222` to avoid conflicting with nginx's
  old wildcard `listen 22`. That workaround is no longer needed or present
  here.
  - Consequence: sshd must bind an **explicit** `ListenAddress` (its own
    real IP), not the default wildcard — a wildcard `0.0.0.0:22` bind
    cannot coexist with nginx's specific `172.21.129.200:22` bind on the
    same port; the kernel treats a wildcard bind as overlapping any
    specific-address bind on that port.
  - Consequence: both nodes need `net.ipv4.ip_nonlocal_bind = 1`
    (`99-nonlocal-bind.conf`), so nginx can bind the VIP even while it's
    BACKUP and the address isn't actually assigned to that node's interface
    yet.
- **`enable_script_security` + `script_user root`** in keepalived's
  `global_defs` — required by keepalived 2.x to run `vrrp_script` checks at
  all; without it keepalived refuses to execute them.
- **`systemd` `Restart=on-failure`** override for `keepalived.service`
  itself (`keepalived-override.conf`) on both nodes. By default the
  packaged unit does **not** auto-restart keepalived after a crash — found
  this the hard way during testing (see below): after a `kill -9`, the
  service stayed in `failed` state until manually restarted. For a
  component whose entire job is "notice failure and react", that's a gap
  worth closing.
- **VIP and VRRP router ID were chosen carefully, not arbitrarily**, since
  this runs on a shared lab subnet (172.21.128.0/20) with many other
  devices: `172.21.129.200` was verified free (no ping/ARP response) before
  use, and `virtual_router_id 51` was chosen after passively listening for
  existing VRRP traffic (protocol 112) on the segment and seeing none.
  **Caveat:** this only guarantees the address was free *at setup time* and
  stays defended by ARP as long as keepalived keeps running — it does not
  reserve the address in this network's DHCP server, which is outside our
  control. Anyone making this permanent should get `172.21.129.200`
  formally excluded from the lab's DHCP pool.
- **VRRP `auth_pass`** (`sshlb26`) uses VRRPv2 simple-text authentication,
  which is not real security (it's not encrypted or resistant to a
  determined attacker on the same L2 segment) — it exists only to stop
  *accidental* collisions between unrelated VRRP instances using the same
  router ID on the same subnet, so it's fine to have it visible here.

## File locations on the actual RHEL hosts

| File in this repo | Path on the real host | Host |
|---|---|---|
| `server1-172.21.129.16/nginx.conf` | `/etc/nginx/nginx.conf` | 172.21.129.16 |
| `server1-172.21.129.16/ssh-lb.conf` | `/etc/nginx/stream.conf.d/ssh-lb.conf` | 172.21.129.16 |
| `server1-172.21.129.16/sshd_config` | `/etc/ssh/sshd_config` | 172.21.129.16 |
| `server1-172.21.129.16/keepalived.conf` | `/etc/keepalived/keepalived.conf` | 172.21.129.16 |
| `server1-172.21.129.16/check_nginx.sh` | `/etc/keepalived/check_nginx.sh` | 172.21.129.16 |
| `server1-172.21.129.16/keepalived-override.conf` | `/etc/systemd/system/keepalived.service.d/override.conf` | 172.21.129.16 |
| `server1-172.21.129.16/99-nonlocal-bind.conf` | `/etc/sysctl.d/99-nonlocal-bind.conf` | 172.21.129.16 |
| `server1-172.21.129.16/firewalld-public-zone.txt` | output of `firewall-cmd --list-all` | 172.21.129.16 |
| `server2-172.21.129.138/*` | same paths as above | 172.21.129.138 |
| `selinux/ssh_lb_nginx.te` / `.pp` | installed via `semodule -i` (same module, both hosts) | both |

No private SSH host keys, `/etc/shadow`, or account passwords are included.

## Failover tests performed

Test users `lbtest1`..`lbtest5` (from the previous repo) already exist on
both hosts. All tests below used SSH client connections to the single VIP
`172.21.129.200`, from a genuinely separate machine (not loopback).

### 1. Graceful stop (`systemctl stop keepalived` on .16)

| | |
|---|---|
| VIP before | `172.21.129.16` |
| Action | `systemctl stop keepalived` on .16 |
| VIP after ~2s | `172.21.129.138` |
| SSH via VIP during/after | kept working throughout |

### 2. Ungraceful crash (`kill -9` on keepalived's process on .16)

This is the harder test — no graceful VRRP "priority 0" shutdown notice,
just the process vanishing, closer to what an actual crash or lost network
looks like. Failover here depends purely on the peer noticing missed
advertisements (`advert_int 1`, so within a few seconds).

| | |
|---|---|
| VIP before | `172.21.129.16` |
| Action | `kill -9` on keepalived's main process on .16 |
| VIP after ~4s | `172.21.129.138` |
| SSH via VIP during/after | kept working throughout |
| Extra finding | keepalived itself did **not** auto-restart after the kill (stayed `failed`) until `Restart=on-failure` was added — see above |

### 3. Recovery / preemption

After restarting keepalived on .16 in both scenarios above, the VIP moved
back to `172.21.129.16` within ~3s each time (`priority 150` > `.138`'s
`100`, `preempt` is on), and `.138` correctly released it.

### 4. Load-balancing policy re-verified post-HA

8 concurrent SSH connections through the VIP (mixing `root` and the 5
`lbtest*` users), read from nginx's `stream_access.log`:

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

**5 to the primary, 3 overflowed** — confirms adding HA to the load-balancer
tier didn't change the original connection-limiting behavior at all; the two
concerns (which node is reachable, which backend gets priority) are fully
independent, as intended.

## Known limitations (lab setup, not production-hardened)

- Root login over SSH with a shared password is enabled for this exercise;
  use key-based auth and disable `PermitRootLogin`/`PasswordAuthentication`
  for anything beyond a lab.
- `172.21.129.200` is only actually reserved by continuously running
  keepalived defending it via ARP — get it excluded from DHCP for a
  permanent setup (see above).
- Both nodes' configs must be kept in sync manually (nginx policy,
  keepalived priorities/router-id, host users). A real deployment would
  automate this (config management, or a shared config source).
- VRRP's simple-text auth is not real security against a malicious actor on
  the same L2 segment — anyone on this subnet could in principle inject
  competing VRRP advertisements. Out of scope for a lab exercise, but worth
  knowing.
