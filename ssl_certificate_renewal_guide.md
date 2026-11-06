# Automated SSL/TLS Certificate Renewal and Deployment
- Let's Encrypt • acme.sh • DNS-01 GoDaddy • HAProxy • Nginx • Mailcow
Centralized Architecture • Automatic Renewal • HAProxy • Mailcow • High Availability

---

## 1. Objective and Scope
This guide describes a centralized architecture for automatically issuing and renewing a wildcard certificate (`*.domain.com`) using Let’s Encrypt, and then distributing it across multiple servers: four HAProxy reverse proxies, one Nginx server, and a Mailcow mail platform.

DNS-01 validation is performed via the GoDaddy API. It validates the domain without exposing any HTTP ports to the internet and supports wildcard certificates.

The central server (`certs`) holds `acme.sh`, the DNS API credentials, and the deployment SSH key. The target servers do not store the GoDaddy API key.

- Centralized certificate issuance for `domain.com` and `*.domain.com`.
- Secure deployment via SSH/SCP across six target servers.
- Configuration validation before reloading services.
- Backup of previous certificates and rollback procedure.
- Expiration and renewal monitoring.

---

## 2. Target Architecture
**Logical Flow:** GoDaddy DNS (`_acme-challenge` TXT record) ↔ DNS API ↔ `certs` server ↔ SSH/SCP ↔ HAProxy ×4, Nginx ×1, and Mailcow ×1.

The central server triggers deployment after each issuance or renewal. Each target server applies the certificate using a local script executed via `sudo`, restricted to an authorized command.

---

## 3. DNS Preparation and Local Resolution (`/etc/hosts`)
On the central `certs` server, define the internal names and their IP addresses so that SSH can reach the target servers. The addresses below are examples to be replaced with your infrastructure's actual IPs.

```text
# /etc/hosts
127.0.0.1       localhost
10.10.10.10     certs
10.10.10.11     haproxy1
10.10.10.12     haproxy2
10.10.10.13     haproxy3
10.10.10.14     haproxy4
10.10.10.15     nginx1
10.10.10.16     mailcow
```

*Note:* `/etc/hosts` entries are only required if these names are not already resolved by your internal DNS. They do not replace public DNS records used by clients.

---

## 4. Prerequisites and Checks
- A domain managed in GoDaddy and a dedicated Production API key for DNS automation.
- A Linux server (`certs`) with outbound HTTPS access to `api.godaddy.com` and `acme-v02.api.letsencrypt.org`.
- SSH access from the `certs` server to the six target servers.
- HAProxy, Nginx, and Mailcow services fully operational before proceeding.
- A backup of existing certificates and configurations.
- A maintenance window scheduled for restarting Mailcow services.

> **Wildcard Warning:** `*.domain.com` covers `app.domain.com` and `mail.domain.com`, but *not* `domain.com` itself (which must be added explicitly), nor `app.client.domain.com`. The requested certificate must therefore include both `domain.com` and `*.domain.com`.

---

## 5. Preparing the Target Servers
On each of the four HAProxy servers, the Nginx server, and the Mailcow server, create a dedicated user account for depositing files:

```bash
adduser --disabled-password --gecos "" certdeploy
install -d -o certdeploy -g certdeploy -m 700 /home/certdeploy/incoming /home/certdeploy/.ssh
touch /home/certdeploy/.ssh/authorized_keys
chown certdeploy:certdeploy /home/certdeploy/.ssh/authorized_keys
chmod 600 /home/certdeploy/.ssh/authorized_keys
```

The public key of the `certs` server will be added to `authorized_keys` in Step 8. The `certdeploy` account must not have a privileged shell or general `sudo` access.

Verify resolution and connectivity before proceeding:
```bash
getent hosts haproxy1 haproxy2 haproxy3 haproxy4 nginx1 mailcow
ssh certdeploy@haproxy1 true
```

---

## 6. Installation Scripts on Target Servers

### 6.1 HAProxy — `/usr/local/bin/install-haproxy-cert`
```bash
#!/usr/bin/env bash
set -euo pipefail
NEW=/home/certdeploy/incoming/domain.com.pem
DST=/etc/haproxy/certs/domain.com.pem
CFG=/etc/haproxy/haproxy.cfg

test -s "$NEW"
openssl x509 -in "$NEW" -noout -checkend 86400
install -d -m 700 /etc/haproxy/certs
[ -f "$DST" ] && cp -p "$DST" "$DST.bak"
install -m 600 -o root -g root "$NEW" "$DST"
rm -f "$NEW"
if haproxy -c -q -f "$CFG"; then
    systemctl reload haproxy
    echo "HAProxy reloaded"
else
    echo "Invalid configuration: rolling back to previous certificate" >&2
    if [ -f "$DST.bak" ]; then cp -p "$DST.bak" "$DST"; fi
    exit 1
fi
```

Configure the HAProxy bind to point to the PEM file combining the full chain and private key:
```haproxy
bind :443 ssl crt /etc/haproxy/certs/domain.com.pem
```

---

### 6.2 Nginx — `/usr/local/bin/install-nginx-cert`
Nginx generally uses two separate files: the full chain and the private key. Adjust the SSL server block paths to match those below.

```bash
#!/usr/bin/env bash
set -euo pipefail
IN=/home/certdeploy/incoming
DEST=/etc/nginx/certs
BASE=valomnia.com

# Sanity check: the key must match the cert
c=$(openssl x509 -in "$IN/fullchain.pem" -noout -pubkey | sha256sum)
k=$(openssl pkey -in "$IN/key.pem" -pubout | sha256sum)
[ "$c" = "$k" ] || { echo "cert/key mismatch" >&2; exit 1; }

install -d -m 750 -o root -g root "$DEST"
[ -f "$DEST/$BASE.fullchain.pem" ] && cp -p "$DEST/$BASE.fullchain.pem" "$DEST/$BASE.fullchain.pem.bak"
[ -f "$DEST/$BASE.key.pem" ]       && cp -p "$DEST/$BASE.key.pem"       "$DEST/$BASE.key.pem.bak"

install -m 644 -o root -g root "$IN/fullchain.pem" "$DEST/$BASE.fullchain.pem"
install -m 600 -o root -g root "$IN/key.pem"       "$DEST/$BASE.key.pem"
rm -f "$IN/fullchain.pem" "$IN/key.pem"

nginx -t
systemctl reload nginx
```

Example Nginx configuration snippet:
```nginx
server {
    listen 443 ssl http2;
    server_name domain.com *.domain.com;

    ssl_certificate     /etc/nginx/certs/domain.com.fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/domain.com.key;

    # Existing application configuration...
}
```

---

### 6.3 Mailcow — `/usr/local/bin/install-mailcow-cert`
Adjust `MC` if Mailcow is installed in a different directory. The script backs up files before replacing them.

```bash
#!/usr/bin/env bash
set -euo pipefail
IN=/home/certdeploy/incoming
MC=/opt/mailcow-dockerized
SSL=$MC/data/assets/ssl

test -s "$IN/fullchain.pem" && test -s "$IN/key.pem"
openssl x509 -in "$IN/fullchain.pem" -noout -checkend 86400
cp -p "$SSL/cert.pem" "$SSL/cert.pem.bak"
cp -p "$SSL/key.pem"  "$SSL/key.pem.bak"
install -m 644 -o root -g root "$IN/fullchain.pem" "$SSL/cert.pem"
install -m 600 -o root -g root "$IN/key.pem" "$SSL/key.pem"
rm -f "$IN/fullchain.pem" "$IN/key.pem"
cd "$MC"
docker compose restart postfix-mailcow dovecot-mailcow nginx-mailcow
echo "Mailcow certificate installed"
```

In `mailcow.conf`, set `SKIP_LETS_ENCRYPT=y` to prevent Mailcow's built-in ACME mechanism from overriding the deployed certificate. Apply the change with `docker compose up -d`. (If using an older binary version, use `docker-compose`).

---

## 7. Restricting Sudo Privileges
After creating each script on the respective server, apply permissions and allow only the specific command. 

On HAProxy servers:
```bash
chown root:root /usr/local/bin/install-haproxy-cert
chmod 755 /usr/local/bin/install-haproxy-cert
echo 'certdeploy ALL=(root) NOPASSWD: /usr/local/bin/install-haproxy-cert' > /etc/sudoers.d/certdeploy
chmod 440 /etc/sudoers.d/certdeploy
visudo -c
```

On Nginx, replace the path and command name with `/usr/local/bin/install-nginx-cert`. On Mailcow, use `/usr/local/bin/install-mailcow-cert`. Verify with `visudo -c` on each machine.

---

## 8. Installing `acme.sh` on the Central Server
```bash
curl https://get.acme.sh | sh -s email=admin@domain.com
source ~/.bashrc
acme.sh --set-default-ca --server letsencrypt
```

Create a dedicated GoDaddy Production API key. Enter the credentials interactively without hardcoding them into a script or putting them in a support ticket:
```bash
read -s -p "GoDaddy API Key: " GD_Key; echo
read -s -p "GoDaddy Secret: " GD_Secret; echo
export GD_Key GD_Secret
```

Issue the wildcard certificate via DNS-01 validation:
```bash
acme.sh --issue --dns dns_gd -d domain.com -d '*.domain.com'
```

The initial issuance should end with `Cert success`. `acme.sh` stores the required DNS parameters for future renewals inside its account directory, which is accessible only to `root`. Protect this directory and restrict access to the `certs` server.

---

## 9. SSH and Central Deployment Script
Create a dedicated SSH key on the `certs` server, then add its public key to `authorized_keys` on all six target servers:
```bash
ssh-keygen -t ed25519 -f /root/.ssh/certdeploy -N '' -C 'certdeploy@certs'
cat /root/.ssh/certdeploy.pub
```

Test SSH access to each target after accepting and verifying its fingerprint:
```bash
for h in haproxy1 haproxy2 haproxy3 haproxy4 nginx1 mailcow; do
  ssh -i /root/.ssh/certdeploy certdeploy@$h true && echo "OK $h"
done
```

Create `/root/deploy-domain-cert.sh`:
```bash
#!/usr/env bash
set -uo pipefail
SRC=/root/certs                          # populated by acme.sh --install-cert
HAPROXY_HOSTS="haproxy1 haproxy2 haproxy3 haproxy4"
MAIL_HOST="mailcow"
U=certdeploy
PORT=22
NGINX_HOST="nginx1"      # IP or actual hostname of the nginx server
NGINX_PORT=22            # actual SSH port of this server

NGINX_SCP_OPT=(-i /root/.ssh/certdeploy
               -P "$NGINX_PORT"
               -o BatchMode=yes
               -o ConnectTimeout=15
               -o StrictHostKeyChecking=yes)

NGINX_SSH_OPT=(-i /root/.ssh/certdeploy
               -p "$NGINX_PORT"
               -o BatchMode=yes
               -o ConnectTimeout=15
               -o StrictHostKeyChecking=yes)

SCP_SSHOPT=(-i /root/.ssh/certdeploy
            -P "$PORT"
            -o BatchMode=yes
            -o ConnectTimeout=15
            -o StrictHostKeyChecking=yes)

SSH_SSHOPT=(-i /root/.ssh/certdeploy
            -p "$PORT"
            -o BatchMode=yes
            -o ConnectTimeout=15
            -o StrictHostKeyChecking=yes)

SSHOPT=(-i /root/.ssh/certdeploy -o BatchMode=yes -o ConnectTimeout=15)

umask 077
TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT
cat "$SRC/fullchain.pem" "$SRC/key.pem" > "$TMP/domain.com.pem"   # format expected by HAProxy

fail=0
for h in $HAPROXY_HOSTS; do
  if scp -q "${SSHOPT[@]}" "$TMP/domain.com.pem" "$U@$h:incoming/domain.com.pem" \
     && ssh "${SSHOPT[@]}" "$U@$h" sudo /usr/local/bin/install-haproxy-cert; then
    echo "OK     $h"
  else
    echo "FAILED  $h" >&2; fail=1
  fi
done

if scp -q "${NGINX_SCP_OPT[@]}" \
       "$SRC/fullchain.pem" \
       "$U@$NGINX_HOST:incoming/fullchain.pem" \
   && scp -q "${NGINX_SCP_OPT[@]}" \
       "$SRC/key.pem" \
       "$U@$NGINX_HOST:incoming/key.pem" \
   && ssh "${NGINX_SSH_OPT[@]}" \
       "$U@$NGINX_HOST" \
       sudo /usr/local/bin/install-nginx-cert
then
    echo "OK     $NGINX_HOST"
else
    echo "FAILED  $NGINX_HOST" >&2
    fail=1
fi

if scp -q "${SCP_SSHOPT[@]}" \
       "$SRC/fullchain.pem" \
       "$U@$MAIL_HOST:incoming/fullchain.pem" \
   && scp -q "${SCP_SSHOPT[@]}" \
       "$SRC/key.pem" \
       "$U@$MAIL_HOST:incoming/key.pem" \
   && ssh "${SSH_SSHOPT[@]}" \
       "$U@$MAIL_HOST" \
       sudo /usr/local/bin/install-mailcow-cert
then
    echo "OK     $MAIL_HOST"
else
    echo "FAILED  $MAIL_HOST" >&2
    fail=1
fi
exit $fail
```

Secure the script and prepare the output directory:
```bash
chmod 700 /root/deploy-domain-cert.sh
install -d -m 700 /root/certs
```

---

## 10. Linking Deployment to `acme.sh` Renewal
Install the certificate files into a stable directory and register the deployment script as the reload command:
```bash
acme.sh --install-cert -d domain.com \
  --fullchain-file /root/certs/fullchain.pem \
  --key-file /root/certs/key.pem \
  --reloadcmd "/root/deploy-domain-cert.sh"
```

`acme.sh` will automatically execute this command after each successful renewal. The script returns a non-zero exit code if at least one server fails to update. Any partial failure should trigger an alert and a manual review of the affected servers.

---

## 11. Post-Deployment Verification

### HTTPS Web Traffic — Test across multiple subdomains and endpoints
```bash
echo | openssl s_client -connect app.domain.com:443 -servername app.domain.com 2>/dev/null \
  | openssl x509 -noout -issuer -subject -dates -ext subjectAltName
```

### Mail Services — Test the protocols actually in use
```bash
echo | openssl s_client -connect mail.domain.com:993 -servername mail.domain.com 2>/dev/null \
  | openssl x509 -noout -issuer -dates

echo | openssl s_client -starttls smtp -connect mail.domain.com:587 -servername mail.domain.com 2>/dev/null \
  | openssl x509 -noout -issuer -dates
```

Complete testing by logging into the webmail, sending and receiving a message via Outlook or a mobile device, and checking service logs.