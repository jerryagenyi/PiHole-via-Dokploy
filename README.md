# Pi-hole + Tailscale on Docker

Deploy Pi-hole and Tailscale on a VPS with Docker Compose. The Tailscale node joins your tailnet via an auth key; Pi-hole runs in the same network stack as Tailscale so it gets a Tailscale IP. After you point your tailnet’s DNS at that IP (Step 5), **all** your Tailscale devices (phones, laptops, tablets) use Pi-hole for DNS—blocking ads and trackers everywhere without changing each device’s DNS manually. Pi-hole has no public ports; it is only reachable over Tailscale.

## Prerequisites

- A VPS or server with Docker and Docker Compose (Linux; `/dev/net/tun` is required for Tailscale).
- A [Tailscale](https://tailscale.com) account and an [auth key](https://login.tailscale.com/admin/settings/keys) (reusable recommended).

## Quick start

### 1. Clone and configure

```bash
cd /path/to/PiHole-via-Dokploy
cp .env.example .env
```

Edit `.env` and set your Tailscale auth key:

```bash
TS_AUTHKEY=tskey-auth-xxxxxxxxxxxx
```

### 2. Optional: change Pi-hole password and timezone

In `docker-compose.yml`, under the `pihole` service, set:

- `WEBPASSWORD` — admin password for the Pi-hole web UI (used only on **first** run; if the container already ran, use `pihole setpassword` inside the container—see [Wrong password](#issue-3-pi-hole-shows-wrong-password-even-though-i-set-webpassword)).
- `TZ` — e.g. `Europe/London` (use your timezone).

### 3. Start the stack

```bash
docker compose up -d
```

### 4. Confirm Tailscale and get the DNS IP

- In the [Tailscale admin console → Machines](https://login.tailscale.com/admin/machines), you should see a machine named **pihole-vpn**.
- If it does not appear after a minute, run once (replace with your key):

  ```bash
  docker exec tailscale tailscale up --auth-key=tskey-auth-xxxxxxxxxxxx --accept-dns=false
  ```

- Click the machine and note its **Tailscale IP** (e.g. `100.x.x.x`). This is the IP to use as DNS.

### 5. Use Pi-hole as DNS for your tailnet (block ads on all devices)

**To block ads (including Google ads) on every Tailscale device**, point your tailnet’s DNS at Pi-hole:

1. Open [Tailscale admin console → DNS](https://login.tailscale.com/admin/dns).
2. Under **Nameservers**, click **Add nameserver** → **Custom**.
3. Enter the **Tailscale IP** of the pihole-vpn machine (e.g. `100.x.x.x`).
4. Save and turn **Override DNS** on.

Once Override DNS is on, all devices on your tailnet will use Pi-hole for DNS, so ads and trackers are blocked everywhere (phones, laptops, tablets) without changing each device’s DNS manually. Open the Pi-hole admin UI at `http://<tailscale-ip>/admin` (e.g. `http://100.124.14.33/admin`) from any device on the tailnet to view stats and manage blocklists.

## Managing blocklists on the admin dashboard

Open the Pi-hole admin UI at `http://<tailscale-ip>/admin` (from a device on your tailnet). Use the left sidebar to manage what gets blocked.

### Adlists (blocklists)

These are the main lists of domains Pi-hole blocks (ads, trackers, malware).

- **Go to:** **Group management** → **Adlists** (or **Settings** → **Blocklists** in older UIs).
- **View:** You’ll see a table of adlist URLs. Pi-hole ships with some defaults; you can add more.
- **Add a list:** Click **Add** and enter the list URL, e.g.:
  - [Steven Black’s unified list](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts) (popular, blocks ads + trackers)
  - Or search for “Pi-hole blocklist” to find community lists.
- **Remove a list:** Select the list and delete it.
- **Apply changes:** After adding or removing lists, go to **Tools** → **Update Gravity** (or the **Update** button on the dashboard). This downloads the lists and applies them. Do this whenever you change adlists.

### Allowlist (whitelist)

When a site or app breaks because a domain is blocked, allow that domain.

- **Go to:** **Group management** → **Allowlist** (or **Whitelist**).
- **Add a domain:** Enter the exact domain (e.g. `s.youtube.com` if YouTube broke) and add it. Pi-hole will stop blocking that domain.
- **Bulk allow:** You can paste multiple domains, one per line.

### Denylist (blacklist)

Block a single domain that isn’t in your adlists.

- **Go to:** **Group management** → **Denylist** (or **Blacklist**).
- **Add a domain:** Enter the domain to always block.

### Regex (advanced)

Block domains by pattern (e.g. all subdomains of a tracker).

- **Go to:** **Group management** → **Regex** (or **Settings** → **Blocklists** → regex).
- **Add a pattern:** Use a regex, e.g. `(^|\.)doubleclick\.net$` to block doubleclick and its subdomains. Test with **Test** before saving.

### Update Gravity

After changing adlists, allowlist, or denylist, Pi-hole must reload the rules:

- **Dashboard:** Click **Update** (or **Update Gravity**).
- Or **Tools** → **Update Gravity**.

This fetches the latest adlist content and rebuilds the block database. Queries use the new rules immediately after the update.

### Quick reference

| Goal | Where to go | Action |
|------|-------------|--------|
| Block more ads/trackers | Group management → Adlists | Add list URL, then Update Gravity |
| A site/app broke | Group management → Allowlist | Add the domain |
| Block one domain | Group management → Denylist | Add the domain |
| Block by pattern | Group management → Regex | Add regex, test, save |
| Apply list changes | Dashboard or Tools | Update Gravity |

## Testing locally before the VPS

Testing on your PC first is a good idea: you can confirm the stack starts, the node appears in Tailscale, and Pi-hole is reachable at its Tailscale IP.

- **Tailscale auth is per machine.** The node that joins your tailnet is tied to where the stack runs. When you run the stack on your PC, *this PC* registers as **pihole-vpn** (using your auth key). When you later run the same stack on the VPS, the *VPS* will join as a (new) node. So you don’t “move” auth from PC to VPS — the VPS will authenticate again when you run `docker compose up` there (using the same or a new auth key).
- **After moving to the VPS:** In the [Tailscale admin console](https://login.tailscale.com/admin/machines) you’ll see two machines if you used the same reusable key (your PC and the VPS). Point your tailnet DNS at the **VPS node’s** Tailscale IP and, if you like, remove or disable the PC test node so only the VPS serves DNS.
- **On Windows (Docker Desktop):** The stack expects a Linux-style `/dev/net/tun` and kernel networking. Docker Desktop runs containers in a Linux VM, so it often works. If you get tun-related errors, try setting `TS_USERSPACE=true` in the tailscale service (and remove `SYS_MODULE`) for local testing only; use the default kernel mode on the VPS.

## Setup walkthrough & issues encountered

This section documents the full setup flow and the issues we hit during testing, with exact fixes.

### Flow (summary)

1. Create `.env` from `.env.example` and set `TS_AUTHKEY` (get a key from [Tailscale → Settings → Keys](https://login.tailscale.com/admin/settings/keys)).
2. Optionally set `WEBPASSWORD` and `TZ` in `docker-compose.yml` for Pi-hole.
3. Run `docker compose up -d`.
4. In [Tailscale → Machines](https://login.tailscale.com/admin/machines), confirm **pihole-vpn** appears and note its Tailscale IP (e.g. `100.124.14.33`).
5. Open Pi-hole at `http://<tailscale-ip>/admin` from a device on the tailnet.
6. In [Tailscale → DNS](https://login.tailscale.com/admin/dns), add the pihole-vpn IP as a custom nameserver and enable Override DNS so **all** Tailscale devices use Pi-hole for ad blocking.

### Issue 1: Tailscale node stays in “NeedsLogin”

**Symptom:** Containers start, but in [Machines](https://login.tailscale.com/admin/machines) there is no **pihole-vpn**, or Tailscale logs show `NeedsLogin`.

**Cause:** `TS_AUTHKEY` was empty in `.env` (or the stack was started before adding the key).

**Fix:** Put your auth key in `.env` as `TS_AUTHKEY=tskey-auth-xxxxxxxxxxxx`, then run `docker compose up -d` again. Alternatively, authenticate once with:

```bash
docker exec tailscale tailscale up --auth-key=tskey-auth-xxxxxxxxxxxx --accept-dns=false
```

(You must include `--accept-dns=false` because we set `TS_ACCEPT_DNS=false`; see Issue 2.)

### Issue 2: `tailscale up` says “requires mentioning all non-default flags”

**Symptom:** When running `docker exec tailscale tailscale up --authkey=...` you see:

```
Error: changing settings via 'tailscale up' requires mentioning all
non-default flags. To proceed, either re-run your command with --reset or
use the command below to explicitly mention the current value of
all non-default settings:

        tailscale up --auth-key=... --accept-dns=false
```

**Cause:** The container was started with `TS_ACCEPT_DNS=false`. Any later `tailscale up` must repeat that flag.

**Fix:** Use the full command:

```bash
docker exec tailscale tailscale up --auth-key=YOUR_KEY --accept-dns=false
```

### Issue 3: Pi-hole shows “Wrong password!” even though I set WEBPASSWORD

**Symptom:** You set `WEBPASSWORD` in `docker-compose.yml` but the Pi-hole login at `http://<tailscale-ip>/admin` rejects it with “Wrong password!”.

**Cause:** Pi-hole writes the admin password to its volume (`./pihole/etc-pihole`) on **first** run. If the container already ran once (e.g. with default or empty password), that stored value is used on later starts and `WEBPASSWORD` in the compose file is not re-applied.

**Fix:** Set the password inside the container. Use the **setpassword** subcommand (not `-a -p`):

- **Interactive (prompted for password):**
  ```bash
  docker exec -it pihole pihole setpassword
  ```
- **Non-interactive (password in single quotes):**
  ```bash
  docker exec pihole pihole setpassword 'YourNewPassword'
  ```

Then log in at `http://<tailscale-ip>/admin` with that password.

### Do I need Step 5?

**Yes, if you want ads (including Google ads) blocked on all your Tailscale devices.** Step 5 is what makes every phone, laptop, and tablet on your tailnet use Pi-hole for DNS automatically. If you skip it, only devices whose DNS you set manually to the pihole-vpn IP will use Pi-hole. Step 5 does not affect Pi-hole login or the web UI—only who uses Pi-hole for DNS.

## Directory layout

| Path | Purpose |
|------|--------|
| `tailscale-state/` | Tailscale state (created on first run; keep for persistent node identity). |
| `pihole/etc-pihole/` | Pi-hole config and blocklists. |
| `pihole/etc-dnsmasq.d/` | dnsmasq custom config. |

These are in `.gitignore` so secrets and state are not committed.

## Security notes

- Pi-hole has no public ports; it is only reachable over Tailscale.
- Set a strong admin password: use `WEBPASSWORD` in `docker-compose.yml` before first run, or `docker exec pihole pihole setpassword 'YourPassword'` after (see [Wrong password](#issue-3-pi-hole-shows-wrong-password-even-though-i-set-webpassword)).
- Keep your `.env` and auth key private; do not commit `.env`.

## Troubleshooting

- **Node not in Tailscale admin**: Ensure `TS_AUTHKEY` is set in `.env` and the key is valid. Run `docker exec tailscale tailscale up --auth-key=YOUR_KEY --accept-dns=false` if the image did not log in automatically.
- **Clients not using Pi-hole**: In the Tailscale admin DNS page, add the pihole-vpn Tailscale IP as a custom nameserver and enable Override DNS.
- **Pi-hole web UI**: Use `http://<tailscale-ip>/admin` from a device on the tailnet (same IP you set as DNS).
- **Wrong password**: Pi-hole stores the admin password in its volume on first run. If you set `WEBPASSWORD` in docker-compose after the first start, it may be ignored. Reset it with: `docker exec -it pihole pihole setpassword` (then enter your new password when prompted), or `docker exec pihole pihole setpassword 'YourNewPassword'` (password in single quotes).
- **Still seeing ads**: See [I’m still seeing ads](#still-seeing-ads) below.

### Still seeing ads

Work through this checklist:

1. **Is the device actually using Pi-hole for DNS?**
   - In [Tailscale → DNS](https://login.tailscale.com/admin/dns), confirm your pihole-vpn IP (e.g. `100.124.14.33`) is added as a nameserver and **Override DNS** is **on**.
   - On the device: turn Tailscale off and on (or reconnect to the tailnet) so it picks up the DNS setting.
   - Some browsers use their own DNS (e.g. Chrome’s “Secure DNS”). If it’s set to Google/Cloudflare, that bypasses Pi-hole. Either turn Secure DNS off or set it to use your system DNS.

2. **Is Pi-hole getting queries?**
   - Open the Pi-hole dashboard at `http://<tailscale-ip>/admin`. Do you see **queries** and **blocked** counts increasing when you browse? If not, that device’s traffic isn’t going through Pi-hole (go back to step 1).

3. **Are blocklists loaded and up to date?**
   - In Pi-hole: **Group management** → **Adlists**. Add a strong list if you haven’t (e.g. [Steven Black’s unified list](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts)).
   - Click **Update Gravity** (dashboard or **Tools** → **Update Gravity**) so Pi-hole fetches the latest lists.

4. **Many ads can’t be blocked by DNS alone**
   - A lot of ads are “first-party” (same domain as the site) or use encrypted DNS, so Pi-hole never sees them. For best results, use **Pi-hole + a browser ad blocker** (e.g. [uBlock Origin](https://ublockorigin.com/)). Pi-hole blocks a lot of trackers and ad domains network-wide; uBlock Origin blocks first-party and in-page ads in the browser.

### YouTube ads specifically

**Pi-hole cannot reliably block YouTube ads.** Pi-hole works at DNS level: it only sees domain names. YouTube serves both ads and videos from the same domains (e.g. `googlevideo.com`). Blocking those domains can stop videos from loading as well as ads. Pi-hole's own [Discourse](https://discourse.pi-hole.net/t/how-do-i-block-ads-on-youtube/253) states that DNS block lists won't reliably block YouTube ads and may break YouTube.

**If you still want to try a YouTube-focused block list:**

- **[youTube_ads_4_pi-hole](https://github.com/kboghdady/youTube_ads_4_pi-hole)** — Add this adlist in Pi-hole (**Group management** → **Adlists**): `https://raw.githubusercontent.com/kboghdady/youTube_ads_4_pi-hole/master/youtubelist.txt`
- **Important:** You must allowlist `s.youtube.com` in **Group management → Allowlist** (add it as a domain there). Do **not** add `s.youtube.com` to Adlists—Adlists are blocklist URLs (http/https only); adding a domain there causes "Invalid protocol specified". The [Steven Black hosts](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts) list already allows `s.youtube.com`.
- After adding the list, run **Update Gravity**. If you get YouTube loops or videos not loading, see the repo README for clearing googlevideo entries; use with caution.

**Recommended for YouTube:** Use a **browser extension** (e.g. [uBlock Origin](https://ublockorigin.com/)) or YouTube Premium. Pi-hole is best for network-wide trackers and ad domains; it is not a substitute for in-browser blocking on YouTube.

## Comparison with similar setups

- **[100dollarguy/pihole-tailscale-dns](https://github.com/100dollarguy/pihole-tailscale-dns)** — Pi-hole + Unbound on host, Tailscale on host; uses iptables to allow DNS only from `100.0.0.0/8`. We took the idea of not letting the DNS node use Tailscale DNS (`TS_ACCEPT_DNS=false`). Our stack exposes no ports, so no firewall rules are needed for this service.
- **[zzjiy/pihole-unbound-tailscale-dockerized](https://github.com/zzjiy/pihole-unbound-tailscale-dockerized)** — Full Docker Compose with Pi-hole, Unbound (recursive DNS + DNSSEC), and Tailscale. If you later want recursive resolution and no dependency on Google/Cloudflare, adding Unbound as Pi-hole’s upstream is a natural next step.

## References

- [Tailscale + Docker](https://tailscale.com/kb/1282/docker)
- [Tailscale + Pi-hole](https://tailscale.com/kb/1114/pi-hole)
- [Tailscale DNS](https://tailscale.com/kb/1054/dns)
