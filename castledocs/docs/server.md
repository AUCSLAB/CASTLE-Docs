# FreeIPA Lab Server Documentation

**Host:** lux.alfred.edu (149.84.129.206)  
**Realm:** ALFRED.EDU  
**Domain:** alfred.edu  
**Container engine:** Podman (rootful)  
**Base OS:** Debian (host), Rocky Linux 9 (container image: `freeipa/freeipa-server:rocky-9`)

## 1. Architecture Overview

FreeIPA runs inside a rootful Podman container on the Debian host lux. Because the host only has a single public IP address (149.84.129.206) bound directly to its physical NIC, the container cannot share the host's network directly (`--net=host` is blocked by required user-namespace remapping) and cannot use macvlan (would conflict with the host's own IP). Instead:

- The container runs on a dedicated Podman bridge network (`ipanet`, subnet `10.89.0.0/24`) with a static internal IP `10.89.0.10`.
- FreeIPA is installed and internally configured using that internal IP (`--ip-address=10.89.0.10`), so all of FreeIPA's self-referential checks (hostname resolution, CA subject, Kerberos principals) are internally consistent.
- The container's ports are published (`-p`) to the host's public IP, `149.84.129.206`, so external clients connect to the real static IP and Podman forwards that traffic into the container.
- The container runs read-only with tmpfs for `/run` and `/tmp`, and a bind-mounted `/data` directory holding all persistent FreeIPA state (databases, certs, configs, logs).

**Why not `--ip-address=149.84.129.206` directly?** FreeIPA's installer validates that `<hostname>` resolves to one of the `--ip-address` values you give it. With bridge networking, the container can't see the host's real IP on any of its own interfaces, and pointing `--add-host` at the real public IP causes a hairpin-NAT hang (the CA setup step tries to reach the directory server via `ldap://lux.alfred.edu:389`, which loops out through the host's NAT and back in — something Podman's bridge networking does not support). Using the internal container IP for both `--ip-address` and `--add-host` avoids this entirely, and does not affect external reachability, since Kerberos/LDAP/HTTPS all authenticate by hostname, not by baked-in IP address.

## 2. Prerequisites on the Debian Host

```bash
sudo apt update
sudo apt install -y podman
```

Enable Docker/Podman user namespace remapping in `/etc/docker/daemon.json` if not already present (required for the FreeIPA container's read-only + systemd/cgroup setup):

```json
{ "userns-remap": "default" }
```

(This applies to Docker; Podman rootful mode with `--read-only` + explicit cgroup/tmpfs mounts as shown below does not require this setting to be duplicated for Podman itself.)

Ensure `lux.alfred.edu` resolves to `149.84.129.206` for anyone connecting to the server (via campus DNS A record, or `/etc/hosts` on each client machine, since this deployment does not run FreeIPA's integrated DNS server).

## 3. One-Time Setup Commands

### 3.1 Create the dedicated Podman network

```bash
sudo podman network create --subnet 10.89.0.0/24 --disable-dns ipanet
```

(`--disable-dns` avoids a port-53 conflict between Podman's internal aardvark-dns helper and FreeIPA's own DNS-related services.)

### 3.2 Create the data directory and install-options file

```bash
sudo mkdir -p /home/csadmin/ipa-data

sudo tee /home/csadmin/ipa-data/ipa-server-install-options <<'EOF'
--ip-address=10.89.0.10
--realm=ALFRED.EDU
--ds-password=YOUR_DS_PASSWORD_HERE
--admin-password=YOUR_ADMIN_PASSWORD_HERE
--unattended
EOF

sudo chmod 600 /home/csadmin/ipa-data/ipa-server-install-options
```

**Important:** Replace both password placeholders with real, unique passwords before running. This file is only read the first time the container initializes `/data` — if `/data` already contains a configured FreeIPA instance, this file is ignored on subsequent starts.

### 3.3 Run the container

```bash
sudo podman run --name freeipa-server-container -d \
  -h lux.alfred.edu \
  --read-only \
  -v /home/csadmin/ipa-data:/data:Z \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  --tmpfs /run --tmpfs /tmp \
  --network ipanet --ip 10.89.0.10 \
  --add-host lux.alfred.edu:10.89.0.10 \
  --sysctl net.ipv6.conf.all.disable_ipv6=0 \
  --sysctl net.ipv6.conf.lo.disable_ipv6=0 \
  -p 53:53/udp -p 53:53 \
  -p 80:80 -p 443:443 \
  -p 389:389 -p 636:636 \
  -p 88:88 -p 88:88/udp \
  -p 464:464 -p 464:464/udp \
  -p 123:123/udp \
  docker.io/freeipa/freeipa-server:rocky-9
```

The first startup runs `ipa-server-install` automatically using the options file above. This takes 5–10 minutes. Do not interrupt it.

### 3.4 Watch progress

```bash
sudo podman logs -f freeipa-server-container
```

Look for the final banner:

```
==============================================================================
Setup complete
```

If it fails partway through, see the Troubleshooting section below.

## 4. Day-to-Day Operations

### 4.1 Check container status

```bash
sudo podman ps
```

Should show `freeipa-server-container` as Up with all the port mappings listed.

### 4.2 If the container is down or crashed: restart it

This is the command you'll use 95% of the time when something goes wrong. As long as nobody has deleted the container (see below), it still exists — it's just stopped — and all its state is intact.

```bash
sudo podman start freeipa-server-container
```

Confirm it's back up:

```bash
sudo podman ps
```

Watch it come back online, especially useful if it crashed mid-operation and you want to see what happened:

```bash
sudo podman logs -f freeipa-server-container
```

Manual stop (e.g., for host maintenance):

```bash
sudo podman stop freeipa-server-container
```

Data persists across stop/start since it's stored in the bind-mounted `/home/csadmin/ipa-data` directory. Do not `rm -rf` that directory unless you intend to wipe and reinstall from scratch.

### 4.3 If the container was deleted (`podman rm`): recreate it

`podman start` only works if the container object still exists. If someone ran `podman rm` or `podman rm -f` on it, the container itself is gone (though your data in `/home/csadmin/ipa-data` is untouched, since it lives outside the container). In that case, you need to run the full `podman run` command again — see Section 3.3 or Section 5 (systemd unit) below. It will not re-run `ipa-server-install`, since `/home/csadmin/ipa-data` already contains a fully configured FreeIPA instance — it just detects that and starts the existing services directly.

Quick summary:

| Situation | What to do |
|---|---|
| Container stopped/crashed but still exists | `podman start freeipa-server-container` |
| Container was deleted (`podman rm`) | Full `podman run ...` command again (Section 3.3), or restart the systemd unit if configured (Section 5) |
| Host rebooted | Automatic, only if the systemd unit in Section 5 is set up — otherwise, manual `podman start` needed after boot |

### 4.4 View logs

```bash
sudo podman logs freeipa-server-container | tail -100
```

## 5. Making the Container Survive Crashes and Reboots

By default, a freshly-created Podman container has no restart policy — check with:

```bash
sudo podman inspect freeipa-server-container --format '{{.HostConfig.RestartPolicy.Name}}'
```

If this prints `no`, then:

- If the container crashes, Podman will not bring it back automatically.
- If the host reboots (power outage, kernel update, maintenance), the container will not start automatically — it just sits stopped until someone manually runs `podman start`.

For a lab server other students depend on for authentication, this should be fixed so the server is self-healing.

### 5.1 Recommended: generate a systemd unit (survives host reboots)

This is the most reliable way to make a Podman container persistent on a Linux host, since it hooks into the same init system that manages everything else on boot.

```bash
sudo podman stop freeipa-server-container
sudo podman rm freeipa-server-container
```

Re-run the original container command from Section 3.3 (without `-d` is fine too, systemd will manage backgrounding), then generate and install the unit:

```bash
sudo podman generate systemd --new --name freeipa-server-container --files
sudo mv container-freeipa-server-container.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now container-freeipa-server-container.service
```

This creates a systemd service that:

- Starts the container automatically on host boot
- Restarts it automatically if it crashes

Can be managed with standard systemd commands going forward:

```bash
sudo systemctl status container-freeipa-server-container.service
sudo systemctl restart container-freeipa-server-container.service
sudo systemctl stop container-freeipa-server-container.service
```

### 5.2 Simpler alternative: `--restart=always` flag

If you don't want to deal with systemd, adding `--restart=always` to the `podman run` command (see Section 3.3) will make Podman restart the container automatically if it crashes while the Podman service itself is running. This is easier to set up but is less robust across actual host reboots than the systemd approach above — Podman's own documentation recommends the systemd route for anything meant to survive a reboot reliably.

## 6. Accessing the Server

### 6.1 Web UI (from any machine on the network)

```
https://149.84.129.206/ipa/ui/
```

Your browser will show a self-signed certificate warning — this is expected, since the CA was configured with Chaining: self-signed. Accept the exception, or better, distribute the CA cert (`/root/cacert.p12` inside the container) to client machines' trust stores.

Log in with username `admin` and the admin password you set in the options file.

### 6.2 Root/terminal access inside the container

```bash
sudo podman exec -it freeipa-server-container bash
```

This drops you into a root shell inside the running container. From here you can run all standard IPA CLI tools.

### 6.3 Authenticate to IPA via Kerberos (inside the container)

```bash
kinit admin
```

Enter the admin password when prompted. Verify with:

```bash
klist
```

You should see a valid ticket for `admin@ALFRED.EDU`.

### 6.4 Common IPA CLI commands (run after `kinit admin`)

```bash
ipa user-find admin        # look up a user
ipa user-add jdoe --first=John --last=Doe --password   # add a new user
ipa group-add students     # create a group
ipa host-add client1.alfred.edu --ip-address=<client-ip>   # pre-register a client
```

### 6.5 Exit the container shell

```bash
exit
```

This returns you to the Debian host shell. The container keeps running in the background.

## 7. Backups

Back up the CA certificate bundle — required to create replicas or recover from disaster:

```bash
sudo podman cp freeipa-server-container:/root/cacert.p12 /home/csadmin/cacert-backup.p12
```

Store this somewhere safe outside the container/host (e.g., encrypted off-host storage). The password protecting this file is the Directory Manager (`--ds-password`) password.

Full data backup (stop the container first for consistency):

```bash
sudo podman stop freeipa-server-container
sudo tar czf /home/csadmin/ipa-data-backup-$(date +%F).tar.gz -C /home/csadmin ipa-data
sudo podman start freeipa-server-container
```

## 8. Enrolling Client Machines

GO TO SECTION 10 IF YOU WANT THE SCRIPT TO DO ALL OF THE ENROLLING

### 8.1 Prerequisites on the client (manual steps)

Type `su` and enter the admin password.

**Step 1 — Set the hostname as a lowercase FQDN.** `ipa-client-install` requires the hostname to be a fully qualified domain name (`<name>.alfred.edu`, not a short name like `debian`), and it must be **all lowercase** — it will reject a hostname like `CLIENT1.alfred.edu` or `Client1.alfred.edu` outright with `Invalid hostname '...', must be lower-case`.

```bash
hostnamectl set-hostname <clientname>.alfred.edu
```

Always use a lowercase `<clientname>` (e.g. `client1`, `comp1`, `castle` — not `Client1` or `CLIENT1`).

**Step 2 — Fix `/etc/hosts`.** Right after setting the hostname, `hostname -f` will almost always fail with `hostname: Name or service not known` — this is expected. Setting the hostname with `hostnamectl` does **not** create a DNS/hosts entry for it; `hostname -f` needs something to actually resolve the name to an IP, and nothing does that yet.

Open the file for editing:

```bash
nano /etc/hosts
```

You'll typically see something like:

```text
127.0.0.1   localhost
127.0.1.1   debian
```

**Edit the `127.0.1.1` line** to match your new FQDN and short name (replace `debian` — or whatever stale short name is there — with your new hostname), and **add a line for the FreeIPA server**. The file should end up looking like:

```text
127.0.0.1   localhost
127.0.1.1   client1.alfred.edu client1

149.84.129.206  lux.alfred.edu lux

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Save and exit (in `nano`: `Ctrl+O`, `Enter`, then `Ctrl+X`).

**Step 3 — Verify `hostname -f` now resolves.**

```bash
hostname -f
```

This should print your FQDN cleanly (e.g. `client1.alfred.edu`) with no error. Do not proceed to enrollment until this works — if `ipa-client-install` is run while this is broken, it will fail early with a hostname-related error.

**Step 4 — Verify the FreeIPA server resolves and is reachable.**

```bash
getent hosts lux.alfred.edu
ping -c 2 149.84.129.206
nc -zv -w5 149.84.129.206 389
```

If ping or the port check fails, do not proceed — see Section 11, item 9 (Docker/Podman iptables conflict) and Section 11 (firewall ports) before troubleshooting further.

### 8.2 Install the FreeIPA client and CASTLE roaming-home packages

```bash
sudo apt update
sudo apt install -y freeipa-client nfs-common autofs sssd-tools
```

### 8.3 Run the installer

```bash
sudo ipa-client-install --server=lux.alfred.edu --domain=alfred.edu --realm=ALFRED.EDU
```

> **Do not use `--mkhomedir` on new CASTLE machines.** CASTLE student home directories are stored centrally on `lux` and mounted through NFS and autofs. Local student home directories should not be created by PAM.

### 8.4 Prompts you will see, and exactly what to type

```text
WARNING: conflicting time&date synchronization service 'ntp' will be disabled in favor of chronyd
```

Informational only — no action needed, it proceeds automatically.

```text
Autodiscovery of servers for failover cannot work with this configuration.
...
Proceed with fixed values and no DNS discovery? [no]: yes
```

Type **`yes`**. This is expected since the server doesn't run integrated DNS for SRV-record autodiscovery.

```text
Do you want to configure chrony with NTP server or pool address? [no]:
```

Just press **Enter** (accept the default `no`). The default chrony configuration is sufficient.

```text
Client hostname: <clientname>.alfred.edu
Realm: ALFRED.EDU
DNS Domain: alfred.edu
IPA Server: lux.alfred.edu
BaseDN: dc=alfred,dc=edu

Continue to configure the system with these values? [no]: yes
```

Type **`yes`**. Confirm the values shown match what you expect.

```text
User authorized to enroll computers: admin
```

Type **`admin`**.

```text
Password for admin@ALFRED.EDU:
```

Enter the FreeIPA admin password.

Follow the interactive prompts, or pre-register the host on the server first with `ipa host-add` (see 6.4) and use a one-time password for unattended enrollment.

### 8.5 Configure CASTLE roaming-home automount

After `ipa-client-install` completes successfully, configure the client to retrieve the CASTLE automount maps from FreeIPA:

```bash
sudo ipa-client-automount \
    --server=lux.alfred.edu \
    --location=default
```

Clear any cached FreeIPA information:

```bash
sudo /usr/sbin/sss_cache -E || true
```

Restart SSSD:

```bash
sudo systemctl restart sssd
```

Enable and restart autofs:

```bash
sudo systemctl enable --now autofs
sudo systemctl restart autofs
```

Verify both services are running:

```bash
systemctl is-active sssd
systemctl is-active autofs
```

Both should return:

```text
active
```

Verify the FreeIPA automount maps:

```bash
sudo /usr/sbin/automount -m
```

You should see the CASTLE student-home map:

```text
Mount point: /home/students
instance type(s): sss
map: auto.students
```

and a wildcard entry similar to:

```text
* | -fstype=nfs4,vers=4.2,rw,hard 149.84.129.206:/mnt/raid/homes/&
```

Verify a FreeIPA student's home directory:

```bash
getent passwd <username>
```

The home field should be:

```text
/home/students/<username>
```

Verify that the client can reach the NFS server:

```bash
timeout 3 bash -c '</dev/tcp/149.84.129.206/2049'
```

If the command exits without an error, NFS port 2049 is reachable.

The workstation's static IP must also be listed in the NFS export configuration on `lux`.

### 8.6 Legacy clients enrolled with `--mkhomedir`

Some existing CASTLE machines were originally enrolled using:

```bash
sudo ipa-client-install \
    --server=lux.alfred.edu \
    --domain=alfred.edu \
    --realm=ALFRED.EDU \
    --mkhomedir
```

These existing machines **do not need to be removed from FreeIPA or re-enrolled**.

If the machine has already been configured with:

- `nfs-common`
- `autofs`
- `sssd-tools`
- FreeIPA automount
- working NFS access to `lux`
- working `/home/students/<username>` roaming homes

then leave the existing enrollment alone.

For all **new** CASTLE machines, use the enrollment process in this section without `--mkhomedir`.
## 9. Firewall / Network Requirements

Make sure the Debian host's firewall allows these ports (already published via `-p` in the run command, but the host's own firewall — ufw/nftables/iptables — must also permit them):

| Port | Protocol | Purpose |
|---|---|---|
| 80, 443 | TCP | HTTP/HTTPS (web UI, ACME) |
| 389, 636 | TCP | LDAP / LDAPS |
| 88, 464 | TCP + UDP | Kerberos (KDC, kpasswd) |
| 53 | TCP + UDP | DNS (published but unused — this deployment does not run integrated DNS) |
| 123 | UDP | NTP (time sync, required for Kerberos) |




## 10. Script to enroll client machines into FreeIPA

```bash
#READ THISSSSSSS

## Copy the script over (scp, USB, git clone, whatever's convenient), then:
chmod +x ipa-client-setup.sh

#USAGE: 
    sudo ./ipa-client-setup.sh

#It'll prompt for the hostname interactively. Or skip the prompt:

    sudo ./ipa-client-setup.sh client5
#PROMPTS YOU HAVE TO FOLLOW

#Proceed with fixed values and no DNS discovery? [no]: yes

#Do you want to configure chrony with NTP server or pool address? [no]:  (just press Enter)

#Continue to configure the system with these values? [no]: yes

#User authorized to enroll computers: admin

#Password for admin@ALFRED.EDU: <type the actual password> FOR THE SERVER

# ipa-client-setup.sh
#
# Prepares a Debian/Linux machine and enrolls it as a FreeIPA client
# for the ALFRED.EDU lab realm (server: lux.alfred.edu / 149.84.129.206).
#
# What it does:
#   1. Asks for the short hostname you want for this machine (e.g. "client1")
#   2. Sets the FQDN hostname (lowercase, enforced)
#   3. Fixes /etc/hosts: removes stale/incorrect self-entries, adds a clean
#      one, and ensures the lux.alfred.edu entry is present
#   4. Verifies `hostname -f` resolves before doing anything else
#   5. Installs the freeipa-client package if missing
#   6. Runs ipa-client-install with --mkhomedir
#
# Run as root (or with sudo) on the client machine, while it's on the
# same network as lux.alfred.edu.
#!/usr/bin/env bash
#
# ipa-client-setup.sh
#
# Prepares a Debian/Linux machine and enrolls it as a FreeIPA client
# for the ALFRED.EDU lab realm (server: lux.alfred.edu / 149.84.129.206).
#
# What it does:
#   1. Asks for the short hostname you want for this machine (e.g. "client1")
#   2. Sets the FQDN hostname (lowercase, enforced)
#   3. Fixes /etc/hosts: removes stale/incorrect self-entries, adds a clean
#      one, and ensures the lux.alfred.edu entry is present
#   4. Verifies `hostname -f` resolves before doing anything else
#   5. Installs the FreeIPA/NFS/autofs client packages if needed
#   6. Runs ipa-client-install
#   7. Configures FreeIPA automount for CASTLE roaming home directories
#
# Run as root (or with sudo) on the client machine, while it's on the
# same network as lux.alfred.edu.
#
# Usage:
#   sudo ./ipa-client-setup.sh
#   sudo ./ipa-client-setup.sh client1        # skip the hostname prompt

set -euo pipefail

# ---- Config: edit these if your lab's values ever change ----
IPA_SERVER="lux.alfred.edu"
IPA_SERVER_IP="149.84.129.206"
IPA_DOMAIN="alfred.edu"
IPA_REALM="ALFRED.EDU"
AUTOMOUNT_LOCATION="default"
# ----------------------------------------------------------------

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

info()  { echo -e "${GREEN}==>${NC} $*"; }
warn()  { echo -e "${YELLOW}!!${NC} $*"; }
fail()  { echo -e "${RED}XX${NC} $*"; exit 1; }

# --- Must be root ---
if [[ $EUID -ne 0 ]]; then
    fail "Please run this script as root (e.g. sudo $0)"
fi

# --- Get desired short hostname ---
if [[ -n "${1:-}" ]]; then
    SHORT_NAME="$1"
else
    read -rp "Enter the short hostname for this machine (e.g. client1): " SHORT_NAME
fi

# Lowercase it, strip any accidental domain suffix the user might type
SHORT_NAME=$(echo "$SHORT_NAME" | tr '[:upper:]' '[:lower:]' | sed "s/\.${IPA_DOMAIN}$//")
FQDN="${SHORT_NAME}.${IPA_DOMAIN}"

if [[ -z "$SHORT_NAME" ]]; then
    fail "No hostname provided. Aborting."
fi

info "Target FQDN: $FQDN"

# --- Detect this machine's primary IP ---
info "Detecting this machine's IP address..."
CLIENT_IP=$(ip addr show | grep "inet " | grep -v '127.0.0.1' | awk '{print $2}' | cut -d/ -f1 | head -n1)

if [[ -z "$CLIENT_IP" ]]; then
    warn "Could not auto-detect this machine's IP address."
    read -rp "Enter this machine's IP address manually: " CLIENT_IP
fi

info "Using client IP: $CLIENT_IP"

# --- Set the hostname (lowercase FQDN) ---
info "Setting hostname to $FQDN ..."
hostnamectl set-hostname "$FQDN"

# --- Fix /etc/hosts ---
info "Cleaning up /etc/hosts ..."

HOSTS_FILE="/etc/hosts"
BACKUP_FILE="/etc/hosts.bak.$(date +%s)"
cp "$HOSTS_FILE" "$BACKUP_FILE"
info "Backed up existing /etc/hosts to $BACKUP_FILE"

# Remove any existing 127.0.1.1 line (old self-hostname entries, however malformed)
sed -i '/^127\.0\.1\.1/d' "$HOSTS_FILE"

# Remove any existing lux.alfred.edu line, so we can re-add a clean one
sed -i "/${IPA_SERVER}/d" "$HOSTS_FILE"

# Remove any stale line that already mentions this short name or FQDN
sed -i "/[[:space:]]${SHORT_NAME}\$/d" "$HOSTS_FILE"
sed -i "/${FQDN}/d" "$HOSTS_FILE"

# Add clean entries
{
    echo "127.0.1.1       ${FQDN} ${SHORT_NAME}"
    echo "${IPA_SERVER_IP}  ${IPA_SERVER} lux"
} >> "$HOSTS_FILE"

info "Updated /etc/hosts:"
cat "$HOSTS_FILE"

# --- Verify hostname -f resolves ---
info "Verifying hostname -f resolves correctly..."
RESOLVED_FQDN=$(hostname -f 2>&1) || true

if [[ "$RESOLVED_FQDN" != "$FQDN" ]]; then
    fail "hostname -f returned '$RESOLVED_FQDN', expected '$FQDN'. Check /etc/hosts manually and re-run."
fi
info "hostname -f resolves correctly: $RESOLVED_FQDN"

# --- Verify lux.alfred.edu resolves ---
info "Verifying ${IPA_SERVER} resolves..."
if ! getent hosts "$IPA_SERVER" > /dev/null; then
    fail "${IPA_SERVER} does not resolve. Check /etc/hosts manually and re-run."
fi
info "${IPA_SERVER} resolves correctly."

# --- Basic connectivity check before attempting enrollment ---
info "Checking connectivity to ${IPA_SERVER} (${IPA_SERVER_IP})..."
if ping -c 2 -W 3 "$IPA_SERVER_IP" > /dev/null 2>&1; then
    info "Ping to ${IPA_SERVER_IP} succeeded."
else
    warn "Ping to ${IPA_SERVER_IP} failed. Enrollment will likely fail too."
    warn "Make sure this machine is on the same network as ${IPA_SERVER}."
fi

if command -v nc > /dev/null 2>&1; then
    if nc -zv -w5 "$IPA_SERVER_IP" 389 2>&1 | grep -qi "succeeded\|open"; then
        info "Port 389 (LDAP) reachable."
    else
        warn "Port 389 (LDAP) is not reachable. Enrollment will likely fail."
        warn "See the lab documentation's 'Known Issues' section (Docker/Podman iptables conflict) if this is unexpected."
    fi
fi

# --- Install required client packages ---
info "Installing required FreeIPA/NFS/autofs client packages..."
apt update
apt install -y \
    freeipa-client \
    nfs-common \
    autofs \
    sssd-tools

# --- Run the enrollment ---
info "Starting ipa-client-install ..."
info "You will be prompted to confirm settings and enter the FreeIPA admin password."
echo

if ipa-client-install \
    --server="$IPA_SERVER" \
    --domain="$IPA_DOMAIN" \
    --realm="$IPA_REALM"
then
    info "Enrollment succeeded! This machine is now a member of ${IPA_REALM}."
    info "Note: DNS warnings about missing A/AAAA/SSHFP records are expected and harmless"
    info "(this deployment does not run FreeIPA's integrated DNS server)."
else
    fail "ipa-client-install failed. Check /var/log/ipaclient-install.log for details."
fi

# --- Configure CASTLE roaming-home automount ---
info "Configuring FreeIPA automount for CASTLE roaming home directories..."

if ipa-client-automount \
    --server="$IPA_SERVER" \
    --location="$AUTOMOUNT_LOCATION"
then
    info "FreeIPA automount configuration succeeded."
else
    fail "ipa-client-automount failed. Check the client logs and FreeIPA automount configuration."
fi

# --- Refresh SSSD/autofs ---
info "Refreshing SSSD and autofs..."

if [[ -x /usr/sbin/sss_cache ]]; then
    /usr/sbin/sss_cache -E || true
fi

systemctl restart sssd
systemctl enable --now autofs
systemctl restart autofs

# --- Verify services ---
info "Checking SSSD..."
if systemctl is-active --quiet sssd; then
    info "SSSD is active."
else
    warn "SSSD is not active."
fi

info "Checking autofs..."
if systemctl is-active --quiet autofs; then
    info "autofs is active."
else
    warn "autofs is not active."
fi

# --- Verify NFS connectivity ---
info "Checking NFS connectivity to ${IPA_SERVER_IP}:2049..."

if timeout 3 bash -c "</dev/tcp/${IPA_SERVER_IP}/2049" 2>/dev/null; then
    info "NFS port 2049 is reachable."
else
    warn "NFS port 2049 is not reachable."
    warn "Make sure this workstation's static IP is authorized in the CASTLE NFS export."
fi

# --- Show automount configuration ---
info "Automount maps available on this client:"
/usr/sbin/automount -m || warn "Could not display automount maps."

echo
info "CASTLE client setup is complete."
info "This workstation is enrolled in FreeIPA and configured for roaming NFS home directories."
echo
info "Verify a student account with:"
echo "    getent passwd <username>"
echo
info "The home should be:"
echo "    /home/students/<username>"
echo
info "You can also verify the mount after login with:"
echo '    findmnt -T "$HOME"'

```



## 11. Known Issues Encountered During Setup (for future reference)

These are documented so future maintainers don't have to rediscover them:

- **Docker `--net=host` + userns-remap conflict.** Rootful Docker refuses to combine host networking with user namespace remapping. FreeIPA's read-only + systemd/cgroup requirements need userns-remap, so host networking was abandoned in favor of a dedicated bridge network.
- **Podman rootless port-binding restriction.** Rootless Podman cannot bind privileged ports (<1024) like 53, 80, 389, 88. Solution: run Podman rootful (`sudo podman ...`).
- **IPv6 loopback missing.** The CA setup step needs `::1` on the container's `lo` interface. Fixed with `--sysctl net.ipv6.conf.all.disable_ipv6=0 --sysctl net.ipv6.conf.lo.disable_ipv6=0`.
- **Hairpin NAT hang.** When `--add-host` pointed at the real external IP, pkispawn's LDAP connection attempt (`ldap://lux.alfred.edu:389`) hung indefinitely trying to loop back through the host's NAT/port-forwarding. Fixed by using a dedicated network with a static internal container IP instead.
- **Hostname/IP validation mismatch.** `ipa-server-install` requires the hostname to resolve to one of the `--ip-address` values given. Using the container's internal IP for both resolved this permanently.
- **Unattended mode requires explicit `-r`/`-p`/`-a`.** `--unattended` will not fall back to interactive prompts or environment-variable passwords — `--realm`, `--ds-password`, and `--admin-password` must all be explicitly present in the options file.
- **Options file lives in `/data`.** Wiping `/home/csadmin/ipa-data` (e.g., to retry a failed install) also deletes `ipa-server-install-options` — it must be recreated before each fresh attempt.
- **No restart policy by default.** A freshly-created Podman container has no restart policy (`podman inspect ... RestartPolicy.Name` returns `no`). Without fixing this, a host reboot or crash leaves the server down until someone manually runs `podman start`. See Section 5 for the systemd-based fix.


