# CrowdSec for the Homelab

A hands-on lab that shows how [CrowdSec](https://www.crowdsec.net/) detects attacks in your logs and blocks the attacker, step by step, on a real self-hosted web service.

You'll put a small web app on the internet, scan it with a real vulnerability scanner, and watch CrowdSec spot the scan and ban the scanner's IP within seconds. You won't write a single firewall rule.

```
          Internet: visitors, bots, scanners
                         │
                         ▼   whoami.example.com  →  A record  →  your public IP
                ┌──────────────────┐
                │      Router      │   forwards TCP 443 → homelab (blocks nothing, for now)
                └────────┬─────────┘
                         ▼
┌──────────────────────── Homelab host ──────────────────────────┐
│                                                                │
│   nftables  ◄── firewall bouncer ◄── CrowdSec Local API        │
│      │ drops banned IPs                    ▲                   │
│      ▼                                     │ decisions         │
│   Caddy (TLS, access log) ──► logs/access.log ──► CrowdSec     │
│      │                                     Security Engine     │
│      ▼                                                         │
│   whoami (Docker)                                              │
└────────────────────────────────────────────────────────────────┘
```

> **The golden rule of this series: CrowdSec decides, bouncers enforce.**
> CrowdSec reads logs and decides who to ban. A *bouncer* turns those decisions into actual blocks. Without a bouncer, nothing gets blocked.

## The series

| Part | Topic | Status |
|------|-------|--------|
| **1** | **CrowdSec basics: detect on the host, block on the host (nftables)** | ✅ This guide |
| 2 | Push bans out to a MikroTik router so bad traffic never enters the LAN | Coming soon |
| 3 | Services behind Cloudflare, where your server no longer sees the real client IP | Coming soon |
| 4 | AppSec / WAF: block malicious requests before they reach the app | Coming soon |

---

## Contents

- [Before you start](#before-you-start)
- [What's in this repo](#whats-in-this-repo)
- [Quick start](#quick-start)
- [Step-by-step guide](#step-by-step-guide)
  - [1. Install CrowdSec](#1-install-crowdsec)
  - [2. (Optional) Enroll in the CrowdSec Console](#2-optional-enroll-in-the-crowdsec-console)
  - [3. How CrowdSec works](#3-how-crowdsec-works)
  - [4. Look inside: acquisition, parsers, scenarios](#4-look-inside-acquisition-parsers-scenarios)
  - [5. Test: SSH brute force](#5-test-ssh-brute-force)
  - [6. Decisions](#6-decisions)
  - [7. Install a bouncer](#7-install-a-bouncer)
  - [8. Expose a web service](#8-expose-a-web-service)
  - [9. Teach CrowdSec to read Caddy logs](#9-teach-crowdsec-to-read-caddy-logs)
  - [10. Attack: scan the site and watch the ban](#10-attack-scan-the-site-and-watch-the-ban)
  - [11. Why some requests still got through](#11-why-some-requests-still-got-through)
- [What to expose, and how](#what-to-expose-and-how)
- [Troubleshooting](#troubleshooting)
- [cscli cheat sheet](#cscli-cheat-sheet)
- [Clean up](#clean-up)

---

## Before you start

> ⚠️ **Only scan and attack servers you own or have explicit written permission to test.** Scanning other people's systems without permission is illegal.

### You'll need

| What | Why |
|------|-----|
| A Linux host (Ubuntu/Debian) with Docker and Docker Compose | Runs CrowdSec, Caddy and the demo app |
| A domain where you can add DNS records | Caddy needs a hostname to get a Let's Encrypt certificate |
| A router where you can forward TCP 443 to the host | So the internet can reach Caddy |
| **A machine outside your network** (a cheap cloud VM works, or a laptop on mobile data) | To act as the "attacker". Attacks from your home network are whitelisted and **will never be banned** |
| On the attacker machine: `curl`, [`ffuf`](https://github.com/ffuf/ffuf), `sshpass`, and a wordlist such as [SecLists `common.txt`](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt) | For the tests |

### Placeholders used in this guide

Replace these with your own values:

| Placeholder | Meaning |
|-------------|---------|
| `whoami.example.com` | Your demo hostname |
| `<PUBLIC_IP>` | Your home/server public IP |
| `192.168.1.10` | The homelab host's LAN IP |
| `<ATTACKER_IP>` | Public IP of your test/attacker machine |
| `~/crowdsec-demo` | Where you cloned this repo on the homelab host |

---

## What's in this repo

| File | Purpose |
|------|---------|
| [`docker-compose.yaml`](docker-compose.yaml) | Caddy (reverse proxy, port 443) + `traefik/whoami` (the demo app, not exposed directly) |
| [`Caddyfile`](Caddyfile) | JSON access log, proxy `/` to whoami, **404 for everything else**, Let's Encrypt TLS |
| `logs/` | Caddy writes `access.log` here; CrowdSec reads it |
| [`diagrams/`](diagrams/) | Architecture diagrams used in the video (open the `.html` files in a browser) |

---

## Quick start

Already know CrowdSec and just want the lab running? Here's the short version. Each step is explained below.

```bash
# On the homelab host
curl -s https://install.crowdsec.net -o install-crowdsec.sh   # read it first!
sudo sh install-crowdsec.sh
sudo apt install crowdsec crowdsec-firewall-bouncer-nftables
sudo cscli collections install crowdsecurity/caddy

git clone https://github.com/filip-lebiecki/crowdsec-homelab.git ~/crowdsec-demo && cd ~/crowdsec-demo
# edit Caddyfile: replace the hostname with yours
docker compose up -d

sudo tee /etc/crowdsec/acquis.d/caddy-demo.yaml >/dev/null <<EOF
source: file
filenames:
  - $HOME/crowdsec-demo/logs/*.log
labels:
  type: caddy
EOF
sudo systemctl restart crowdsec
```

Then add a DNS A record (no proxy) and forward TCP 443 on your router to the host, and run the [attack](#10-attack-scan-the-site-and-watch-the-ban) from an outside machine.

---

## Step-by-step guide

### 1. Install CrowdSec

**Don't use your distro's package.** Check what Ubuntu ships:

```bash
apt-cache policy crowdsec
```

It's usually an old release (e.g. 1.4.x). Use CrowdSec's official repository instead. Their docs pipe the install script straight into `sudo sh`. Don't do that with any script you haven't read. Download it, read it, then run it:

```bash
curl -s https://install.crowdsec.net -o install-crowdsec.sh
less install-crowdsec.sh        # it only adds the CrowdSec apt repo and signing key
sudo sh install-crowdsec.sh
sudo apt install crowdsec
```

The installer detects services on the host (syslog, SSH, and Nginx/Apache if present) and sets up log collection for them. Note the yellow line in the output:

> *CrowdSec alone will not block any IP address.*

That's the most important line on the screen. We'll come back to it.

Check that both APIs are healthy:

```bash
sudo cscli lapi status    # Local API: your local decision database
sudo cscli capi status    # Central API: the community ("the crowd")
```

### 2. (Optional) Enroll in the CrowdSec Console

The [web console](https://app.crowdsec.net) gives you a map of alerts, extra blocklists, and a view of what your bouncers block. It's optional.

```bash
sudo cscli console enroll --quick
```

Open the printed URL, click **Enroll engine**, then confirm:

```bash
sudo cscli console status
```

Good to know:
- **Central API ≠ Console.** The Central API shares threat intel and gives you the community blocklist. The Console is only a dashboard.
- Alerts about **private IPs are never sent** to the console, and the free plan keeps only the **last 500 alerts**. If your console looks empty while testing, that's probably why.

### 3. How CrowdSec works

Detection and blocking are **two separate jobs**.

```
 Logs ──► Security Engine ─────────────────────────────────────► Local API ──► Bouncers ──► block
          Acquisition → Parsers → Enrichment → Scenarios → Profiles    │
                                                                       ▼
                                                                  Central API
                                                          (share bans, get the community blocklist)
```

| Component | What it does |
|-----------|--------------|
| **Acquisition** | Reads log sources: files, Docker containers, journald and more |
| **Parsers** | Turn raw log lines into structured fields (IP, path, status code, user...) |
| **Enrichment** | Adds country / network owner (ASN), and **drops whitelisted IPs** (private ranges by default) |
| **Scenarios** | Look for *patterns* of bad behaviour using "leaky buckets". An overflow becomes an **alert** |
| **Profiles** | Turn an alert into a **decision**, by default "ban this IP for 4 hours" |
| **Local API (LAPI)** | Local database of decisions. `cscli` talks to it |
| **Bouncers** | Poll the LAPI (about every 10 s) and actually block traffic |
| **Central API (CAPI)** | Anonymously shares your bans, gives you the community blocklist (about 15,000 IPs) |

> Everything in Part 1 is **log-based**: CrowdSec reads a request *after* it hit your server, so a few malicious requests get through before the ban. To stop the very first bad request you need AppSec (CrowdSec's WAF), covered in Part 4.

### 4. Look inside: acquisition, parsers, scenarios

#### Acquisition: which logs, and what format

```bash
sudo cscli metrics show acquisition
ls /etc/crowdsec/acquis.d/
cat /etc/crowdsec/acquis.d/setup.sshd.yaml
```

Each file lists the **paths** to read and a **`type` label**. The label tells CrowdSec which parser to use (`syslog`, `nginx`, `caddy`...).

#### Parsers: raw text into structured events

```bash
sudo cscli parsers list
```

Parsing runs in stages:

| Stage | Example file | Job |
|-------|--------------|-----|
| `s00-raw` | `/etc/crowdsec/parsers/s00-raw/syslog-logs.yaml` | Clean-up: timestamp, program, message |
| `s01-parse` | `/etc/crowdsec/parsers/s01-parse/sshd-logs.yaml` | Real parsing: source IP, user, event type |
| `s02-enrich` | `/etc/crowdsec/parsers/s02-enrich/geoip-enrich.yaml` | Adds GeoIP / ASN data |
| `s02-enrich` | `/etc/crowdsec/parsers/s02-enrich/whitelists.yaml` | **Drops private ranges** (`10/8`, `172.16/12`, `192.168/16`, `127/8`) |
| `s02-enrich` | `/etc/crowdsec/parsers/s02-enrich/public-dns-allowlist.yaml` | Never bans big public DNS resolvers |

A parser **doesn't decide what's bad**. It only makes every log look the same. Scenarios do the deciding.

> 💡 **Testing tip:** attacks from your home network are whitelisted and never banned. Always test from outside: mobile data or a cloud VM.
>
> 💡 **Don't lock yourself out:** whitelist the public IPs you use (office, VPN exit, etc.). Create `/etc/crowdsec/parsers/s02-enrich/my-whitelist.yaml`:
>
> ```yaml
> name: my/whitelist
> description: "My own public IPs"
> whitelist:
>   reason: "my own IPs"
>   ip:
>     - "198.51.100.7"
>   cidr:
>     - "198.51.100.0/24"
> ```
>
> Then run `sudo systemctl reload crowdsec`.

Browse more parsers on the Hub with `sudo cscli parsers list -a`.

#### Scenarios: patterns of bad behaviour

```bash
sudo cscli scenarios list
sudo less /etc/crowdsec/scenarios/ssh-bf.yaml
```

Most scenarios are **leaky buckets**: every IP gets its own bucket and every matching event adds a drop.

| Field | Example (`ssh-bf`) | Meaning |
|-------|--------------------|---------|
| `leakspeed` | `10s` | One drop leaks out every 10 seconds |
| `capacity` | `5` | More than 5 drops in a burst means **overflow, which raises an alert** |
| `blackhole` | `1m` | After an alert, stay quiet for this IP for 1 minute (no alert spam) |
| `reprocess` | `true` | Feed the overflow back in so other scenarios can react to it |

Slow attackers (say, one try every 15 s) never fill this bucket. That's what `ssh-slow-bf` is for: a bigger bucket that leaks much more slowly.

```bash
sudo cscli metrics show scenarios                 # how often each scenario fired
sudo cscli scenarios inspect crowdsecurity/ssh-bf # full description
sudo cscli scenarios list -a | less               # everything on the Hub
```

### 5. Test: SSH brute force

Start clean:

```bash
sudo cscli alerts delete --all
sudo cscli alerts list
```

On the server, watch two terminals:

```bash
sudo journalctl -u ssh -f            # on non-Ubuntu distros the unit may be "sshd"
```

```bash
sudo tail -f /var/log/crowdsec.log
```

From the **outside** machine, fail 12 SSH logins on purpose. The target must be your server's SSH as seen from the internet. If SSH isn't exposed (it shouldn't be!), forward a port temporarily for this test and remove it afterwards, or skip to the [web test](#8-expose-a-web-service).

```bash
for i in $(seq 1 12); do
  echo "SSH attempt $i"
  sshpass -p wrongpass ssh \
    -o StrictHostKeyChecking=accept-new \
    -o PubkeyAuthentication=no \
    -o PreferredAuthentications=password \
    -o NumberOfPasswordPrompts=1 \
    -o ConnectTimeout=5 \
    demo-attacker@<PUBLIC_IP> true
done
```

Back on the server:

```bash
sudo cscli metrics show scenarios    # the overflow counter went up
sudo cscli alerts list               # usually two alerts: ssh-bf and ssh-slow-bf
sudo cscli alerts inspect -d <ALERT_ID>
```

`inspect` shows the full story: attacker IP, scenario, every event that filled the bucket, plus the country and network owner added by enrichment.

### 6. Decisions

An alert says something bad happened. A **decision** is what actually gets enforced. Profiles create decisions from alerts.

```bash
sudo cscli decisions list
```

Your attacker is banned for 4 hours, with source `crowdsec` (your own engine). You can also add decisions by hand:

```bash
sudo cscli decisions add --ip 203.0.113.10 --type ban --duration 5m --reason "Manual test"
sudo cscli decisions list            # source: cscli
sudo cscli decisions delete --ip 203.0.113.10
```

> `203.0.113.0/24` is a documentation range (RFC 5737). It belongs to nobody, so it's safe for tests.

Decision types:
- **`ban`**: block the IP completely.
- **`captcha`**: show a challenge page instead. Only useful for websites, and only some bouncers support it (covered in Part 3).

**The crowd.** `decisions list` shows only your own decisions by default. See what the community blocklist gives you:

```bash
sudo cscli metrics show decisions
```

That's roughly 15,000 known-bad IPs (scanners, crawlers, brute forcers), already loaded. It works both ways: your engine anonymously reports the IPs it bans. One report alone doesn't ban an IP for everyone. Many independent machines have to agree, so nobody can poison the list.

### 7. Install a bouncer

```bash
sudo cscli bouncers list
```

Empty. **Right now CrowdSec blocks nothing.** Add the simplest bouncer: nftables on the host itself.

```bash
sudo apt install crowdsec-firewall-bouncer-nftables
sudo cscli bouncers list              # registered automatically
sudo nft list table ip crowdsec       # already full of community-blocklist IPs
```

Within about 10 seconds the bouncer pulls all decisions, including your SSH attacker from step 5.

> 🐳 **Docker note:** this bouncer hooks both the **input** and **forward** chains. Traffic to Docker containers goes through **forward**, not input, so a bouncer that only hooks input does **not** protect your containers.

**Prove the golden rule.** Stop the bouncer:

```bash
sudo systemctl stop crowdsec-firewall-bouncer
sudo nft list tables                  # the crowdsec table is gone
sudo cscli decisions list             # ...but every decision is still there
sudo systemctl start crowdsec-firewall-bouncer
```

The decision list says what to block. The bouncer does the blocking.

### 8. Expose a web service

Nobody should expose SSH to the internet anyway. So let's protect something that really is public: a web app.

#### DNS: plain A record, no proxy

Create an **A record** `whoami.example.com → <PUBLIC_IP>`. If you use Cloudflare, make sure the **proxy is OFF (grey cloud)**. With the orange cloud, your server only sees Cloudflare's IPs, never the attacker's real IP, and CrowdSec can't work. (Part 3 covers that case.)

```bash
host whoami.example.com
```

#### Router: forward TCP 443 to the host

Any router works. Forward **TCP 443** to the homelab. Only 443 is needed: the Caddyfile disables the HTTP-01 challenge, so Let's Encrypt validates over 443 (TLS-ALPN).

MikroTik example (adjust the interface and IP):

```routeros
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp dst-port=443 action=dst-nat to-addresses=192.168.1.10 to-ports=443 comment="crowdsec-demo"
/ip firewall filter add chain=forward action=accept connection-nat-state=dstnat in-interface=WAN comment="crowdsec-demo"
```

Make sure the filter rule sits **above** your forward-chain drop rule.

#### The Docker stack

Edit the hostname in [`Caddyfile`](Caddyfile), then start the stack:

```bash
cd ~/crowdsec-demo
docker compose up -d
```

Why put Caddy in front of whoami? Three reasons:

1. **It writes a JSON access log**, which CrowdSec can parse.
2. **It returns 404 for every unknown path.** This matters: CrowdSec's web scenarios count 404s. `whoami` answers *every* path with 200 OK (as do many apps, or they redirect everything to a login page). Then a scanner looks exactly like a normal visitor. Returning 404 for junk paths gives CrowdSec a clear signal.
3. **Automatic HTTPS** from Let's Encrypt.

Test from the outside machine:

```bash
curl https://whoami.example.com
```

whoami prints the client IP it saw, which should be the attacker machine's real public IP. On the server:

```bash
sudo tail -n 3 ~/crowdsec-demo/logs/access.log | jq
```

> 🔎 **Why do scanners show up within minutes?** [Certificate Transparency](https://certificate.transparency.dev/). Every certificate is published in public logs, and scanners watch them. Want to keep subdomain names private? Use a **wildcard certificate**: only `*.example.com` shows up in the logs.

### 9. Teach CrowdSec to read Caddy logs

A **collection** bundles the parsers and scenarios for one service:

```bash
sudo cscli collections list -a
sudo cscli collections inspect crowdsecurity/caddy
sudo cscli collections install crowdsecurity/caddy
```

It includes the Caddy log parser and the web scenarios: probing, sensitive-file hunting, bad user agents, known exploits and more.

Now tell CrowdSec where the log is. Create `/etc/crowdsec/acquis.d/caddy-demo.yaml` (use your real home path):

```yaml
source: file
filenames:
  - /home/<you>/crowdsec-demo/logs/*.log
labels:
  type: caddy
```

Restart and check that lines are both **read** and **parsed**:

```bash
sudo systemctl restart crowdsec
curl https://whoami.example.com/      # from anywhere, to generate a log line
sudo cscli metrics show acquisition
```

> Lines read but **not parsed** almost always means a wrong `type` label or a log format that doesn't match the parser.

### 10. Attack: scan the site and watch the ban

On the server, watch the access log in a compact format (time, IP, path, status):

```bash
sudo tail -f -n 0 ~/crowdsec-demo/logs/access.log \
  | jq -r --unbuffered '[(.ts|floor|todate), .request.client_ip, .request.uri, .status] | @tsv'
```

On the **outside** machine, run `ffuf` against **your own** site:

```bash
ffuf -u https://whoami.example.com/FUZZ -w ~/wordlists/common.txt \
     -mc all -fc 404 -rate 20 -timeout 5 -maxtime 40
```

You'll see a wall of 404s... and then silence. ffuf keeps running, but every request now times out. Try the site again from the attacker:

```bash
curl --max-time 10 https://whoami.example.com/
```

No error page, no "connection refused". The packets are silently dropped. On the server:

```bash
sudo cscli alerts list                # e.g. http-crawl-non_statics, http-probing, http-sensitive-files
sudo cscli alerts inspect -d <ALERT_ID>
sudo cscli decisions list
sudo nft list set ip crowdsec crowdsec-blacklists-crowdsec
```

The attacker's IP is in the nftables set. CrowdSec read the log, caught the scan, and the bouncer dropped the IP, with no hand-written firewall rules.

Unban your test machine:

```bash
sudo cscli decisions delete --ip <ATTACKER_IP>
```

**Debug without banning anyone:** `cscli explain` replays a log through the parsers and scenarios and shows every step. It's very useful when you write your own parsers or scenarios.

```bash
sudo cscli explain --file ~/crowdsec-demo/logs/access.log --type caddy
```

### 11. Why some requests still got through

```bash
sudo jq -r 'select(.request.client_ip=="<ATTACKER_IP>") | .request.uri' ~/crowdsec-demo/logs/access.log | wc -l
```

Two delays add up:

1. **The bucket has to fill.** Detection happens *after* requests are logged.
2. **The bouncer polls on a timer.**

   ```bash
   sudo grep update_frequency /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml   # default: 10s
   ```

The goal isn't to stop request #1. The goal is that the scanner doesn't get to try all ~4,000 paths. To stop request #1 you need a WAF (Part 4).

---

## What to expose, and how

Rule #1 of self-hosting: **don't expose anything you don't need to.** When you do need remote access, pick a tier per service:

| Tier | How | Good for | Trade-offs |
|------|-----|----------|------------|
| **1. VPN only** | WireGuard (or Tailscale / ZeroTier) | SSH, router admin, databases, dashboards, Jupyter | Invisible to scanners: WireGuard doesn't answer without a valid key. Every device needs a client, so it's awkward for family and friends |
| **2. Cloudflare Tunnel** (+ optional Cloudflare Access) | No open ports, home IP hidden | Home Assistant, Open WebUI, SSO, ntfy, SearXNG | Cloudflare decrypts TLS at the edge. Free plan: 100 MB upload limit, no heavy video streaming, HTTP(S) only |
| **3. Direct port forward** | What this lab does | Big uploads (S3 / backups), video streaming (Jellyfin), non-HTTP services (game, mail) | Real client IP (CrowdSec works best), but your IP is public, gets scanned within minutes, and patching is all on you |

Think of it as an onion: the VPN is the hidden core, Cloudflare the protected middle, and direct forwarding the exposed outer layer. The further out a service sits, the more active defense (CrowdSec, fast patching) it needs.

---

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Attack from home never gets banned | Private ranges are whitelisted. Test from mobile data or a cloud VM |
| Decisions exist but nothing is blocked | No bouncer installed or running: `sudo cscli bouncers list`, `systemctl status crowdsec-firewall-bouncer` |
| Docker services not protected | Bouncer must hook the **forward** chain (the nftables bouncer does by default) |
| Acquisition shows lines read but not parsed | Wrong `type` label, or log format doesn't match. Use `cscli explain` |
| CrowdSec sees Cloudflare IPs, not real clients | DNS record is proxied (orange cloud). Use a grey cloud for this lab |
| Scanner never triggers web scenarios | App returns 200 for everything. Return 404 for unknown paths (see `Caddyfile`) |
| Console looks empty | Private IPs aren't sent; free plan keeps only 500 alerts |
| Caddy can't get a certificate | Port 443 not forwarded, or DNS doesn't point at your public IP yet |

---

## cscli cheat sheet

```bash
sudo cscli lapi status / capi status          # API health
sudo cscli console status                      # console enrollment
sudo cscli metrics show acquisition            # which logs are read and parsed
sudo cscli metrics show scenarios              # how often scenarios fired
sudo cscli metrics show decisions              # decisions by origin (incl. community list)
sudo cscli parsers|scenarios|collections list [-a]
sudo cscli collections install crowdsecurity/caddy
sudo cscli alerts list
sudo cscli alerts inspect -d <ID>
sudo cscli alerts delete --all
sudo cscli decisions list
sudo cscli decisions add --ip <IP> --type ban --duration 5m --reason "test"
sudo cscli decisions delete --ip <IP>
sudo cscli bouncers list
sudo cscli explain --file <log> --type <type>  # dry-run the pipeline
```

---

## Clean up

```bash
# Demo stack (-v also removes the Caddy certificate volume)
cd ~/crowdsec-demo && docker compose down -v

# CrowdSec and the bouncer
sudo apt purge crowdsec-firewall-bouncer-nftables crowdsec
sudo rm -rf /etc/crowdsec /var/lib/crowdsec /var/log/crowdsec*.log
```

Remove the router port-forward (MikroTik example):

```routeros
/ip firewall nat remove [find comment="crowdsec-demo"]
/ip firewall filter remove [find comment="crowdsec-demo"]
```

Also delete the DNS record, and the engine in the CrowdSec Console if you enrolled one.

---

**Next up, Part 2:** CrowdSec still decides on the host, but the bans are enforced on the router, so attackers are dropped before they ever enter your network.
