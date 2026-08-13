# CASTLE Server Infrastructure
This page describes how to set up and maintain services used in the CASTLE Lab.

# OS installation
* Install latest Debian(13)
* 
# RAID

+ install mdadm:

```
sudo apt install mdadm
```
+ Clear the old partition tables from the disks:
```bash
sudo dd if=/dev/zero of=/dev/nvme0n1 bs=4096 count=1000
sudo dd if=/dev/zero of=/dev/nvme1n1 bs=4096 count=1000
sudo dd if=/dev/zero of=/dev/nvme2n1 bs=4096 count=1000
```

+ Then create the RAID 5 array:

```
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1
```
+Create a filesystem on your new RAID device:
```bash
sudo mkfs.ext4 -F /dev/md0
```
Create a mount point and mount the drive:
```bash
sudo mkdir -p /mnt/raid
sudo mount /dev/md0 /mnt/raid
```
Persist the configuration so it mounts automatically after a reboot:
```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
```

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

On each lab client machine (must have `freeipa-client` or equivalent installed):

Ensure `lux.alfred.edu` resolves to `149.84.129.206` (via `/etc/hosts` or campus DNS).

Run:

```bash
sudo ipa-client-install --server=lux.alfred.edu --domain=alfred.edu --realm=ALFRED.EDU --mkhomedir
```

Follow the interactive prompts, or pre-register the host on the server first with `ipa host-add` (see 6.4) and use a one-time password for unattended enrollment.

## 9. Firewall / Network Requirements

Make sure the Debian host's firewall allows these ports (already published via `-p` in the run command, but the host's own firewall — ufw/nftables/iptables — must also permit them):

| Port | Protocol | Purpose |
|---|---|---|
| 80, 443 | TCP | HTTP/HTTPS (web UI, ACME) |
| 389, 636 | TCP | LDAP / LDAPS |
| 88, 464 | TCP + UDP | Kerberos (KDC, kpasswd) |
| 53 | TCP + UDP | DNS (published but unused — this deployment does not run integrated DNS) |
| 123 | UDP | NTP (time sync, required for Kerberos) |

## 10. Known Issues Encountered During Setup (for future reference)

These are documented so future maintainers don't have to rediscover them:

- **Docker `--net=host` + userns-remap conflict.** Rootful Docker refuses to combine host networking with user namespace remapping. FreeIPA's read-only + systemd/cgroup requirements need userns-remap, so host networking was abandoned in favor of a dedicated bridge network.
- **Podman rootless port-binding restriction.** Rootless Podman cannot bind privileged ports (<1024) like 53, 80, 389, 88. Solution: run Podman rootful (`sudo podman ...`).
- **IPv6 loopback missing.** The CA setup step needs `::1` on the container's `lo` interface. Fixed with `--sysctl net.ipv6.conf.all.disable_ipv6=0 --sysctl net.ipv6.conf.lo.disable_ipv6=0`.
- **Hairpin NAT hang.** When `--add-host` pointed at the real external IP, pkispawn's LDAP connection attempt (`ldap://lux.alfred.edu:389`) hung indefinitely trying to loop back through the host's NAT/port-forwarding. Fixed by using a dedicated network with a static internal container IP instead.
- **Hostname/IP validation mismatch.** `ipa-server-install` requires the hostname to resolve to one of the `--ip-address` values given. Using the container's internal IP for both resolved this permanently.
- **Unattended mode requires explicit `-r`/`-p`/`-a`.** `--unattended` will not fall back to interactive prompts or environment-variable passwords — `--realm`, `--ds-password`, and `--admin-password` must all be explicitly present in the options file.
- **Options file lives in `/data`.** Wiping `/home/csadmin/ipa-data` (e.g., to retry a failed install) also deletes `ipa-server-install-options` — it must be recreated before each fresh attempt.
- **No restart policy by default.** A freshly-created Podman container has no restart policy (`podman inspect ... RestartPolicy.Name` returns `no`). Without fixing this, a host reboot or crash leaves the server down until someone manually runs `podman start`. See Section 5 for the systemd-based fix.