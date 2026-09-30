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




# NFS,Roaming Home Directory


This document describes the CASTLE Linux lab architecture for centralized authentication, RAID-backed student storage, NFS roaming home directories, FreeIPA automount, quotas, and automatic student-home provisioning.

All deployment-specific and personally identifying information has been replaced with safe placeholders.


The examples below use documentation-only values such as:

```text
Server FQDN: ipa01.example.test
Domain: example.test
Realm: EXAMPLE.TEST
Server IP: 192.0.2.10
Student username: student01
Client IPs: 192.0.2.101, 192.0.2.102, ...
```

`example.test` and `192.0.2.0/24` are placeholders only.

---

# 1. System Goal

CASTLE allows a FreeIPA user to log into any configured Linux lab workstation and receive the same home directory and files.

The basic architecture is:

```text
                    FreeIPA
              ipa01.example.test
                       │
             Authentication / UID
                  / GID / home
                       │
                       ▼
                Lab workstation
                       │
                       │ autofs + NFSv4
                       ▼
          /home/students/<username>
                       │
                       ▼
    192.0.2.10:/mnt/raid/homes/<username>
                       │
                       ▼
                     RAID5
```

The home directory is not copied between computers.

Each workstation mounts the same centrally stored home directory from the server.

This means files written on one workstation are immediately available from another workstation using the same user account.

---

# 2. Main Components

CASTLE uses:

```text
FreeIPA
    Central authentication and identity

SSSD
    Client-side identity lookup and FreeIPA integration

autofs
    Automatic home-directory mounting

NFSv4
    Network filesystem transport

Linux md RAID5
    Centralized redundant storage

ext4
    RAID filesystem

ext4 user quotas
    Storage limits per student

systemd
    Automated RAID-home provisioning

Podman
    FreeIPA server container
```

---

# 3. Example Server Layout

Public documentation example:

```text
Host operating system:
Debian 13

FreeIPA container OS:
Rocky Linux 9

FreeIPA container:
freeipa-server-container

FreeIPA server:
ipa01.example.test

Domain:
example.test

Kerberos realm:
EXAMPLE.TEST

Storage server IP:
192.0.2.10
```

Replace these with private production values during deployment.

---

# 4. RAID Architecture

CASTLE uses a Linux software RAID5 array:

```text
/dev/md0
```

It consists of three NVMe drives:

```text
/dev/nvme0n1
/dev/nvme1n1
/dev/nvme2n1
```

Conceptually:

```text
NVMe 1 ─┐
NVMe 2 ─┼── RAID5 ── /dev/md0 ── ext4 ── /mnt/raid
NVMe 3 ─┘
```

RAID5 provides single-disk redundancy.

RAID is not a backup.

It does not protect against:

```text
accidental deletion
administrator mistakes
filesystem corruption
malware
multiple-disk failure
loss of the server
```

A separate backup system should still be used for important data.

---

# 5. Check RAID Health

Basic status:

```bash
cat /proc/mdstat
```

Detailed status:

```bash
sudo mdadm --detail /dev/md0
```

Device overview:

```bash
lsblk -f
```

A healthy RAID5 should show all expected member devices active.

---

# 6. Rebuilding RAID From Blank Drives

This section is destructive.

Do not run these commands on a working array containing data.

Install `mdadm`:

```bash
sudo apt update
sudo apt install -y mdadm
```

Create a three-disk RAID5:

```bash
sudo mdadm --create /dev/md0 \
    --level=5 \
    --raid-devices=3 \
    /dev/nvme0n1 \
    /dev/nvme1n1 \
    /dev/nvme2n1
```

Monitor RAID initialization:

```bash
watch cat /proc/mdstat
```

Create an ext4 filesystem:

```bash
sudo mkfs.ext4 /dev/md0
```

Get the new filesystem UUID:

```bash
sudo blkid /dev/md0
```

A newly formatted RAID will have a new UUID.

Do not copy a UUID from another system.

---

# 7. Mount the RAID

Create the mount point:

```bash
sudo mkdir -p /mnt/raid
```

Example `/etc/fstab` entry:

```fstab
UUID=<RAID_FILESYSTEM_UUID> /mnt/raid ext4 defaults,nofail,discard,usrquota 0 2
```

Replace:

```text
<RAID_FILESYSTEM_UUID>
```

with the actual value returned by:

```bash
sudo blkid /dev/md0
```

Reload systemd and mount:

```bash
sudo systemctl daemon-reload
sudo mount -a
```

Verify:

```bash
findmnt /mnt/raid
df -h /mnt/raid
```

---

# 8. Enable User Quotas

CASTLE uses filesystem quotas so one student cannot consume the entire RAID.

Current policy:

```text
Soft limit: 4 GiB
Hard limit: 5 GiB
```

Install quota utilities:

```bash
sudo apt update
sudo apt install -y quota e2fsprogs
```

Enable the ext4 internal user-quota feature:

```bash
sudo tune2fs -Q usrquota /dev/md0
```

If a filesystem check is required:

```bash
sudo umount /mnt/raid
sudo e2fsck -f /dev/md0
sudo mount /mnt/raid
```

Make sure `/etc/fstab` contains:

```text
usrquota
```

Check quota enforcement:

```bash
sudo quotaon -p /mnt/raid
```

Desired state:

```text
user quota: on
group quota: off
project quota: off
```

---

# 9. ext4 Internal Quotas

CASTLE uses ext4's internal quota inode.

Inspect the filesystem:

```bash
sudo tune2fs -l /dev/md0
```

The filesystem should show the `quota` feature and a user quota inode.

An old-style file such as:

```text
/mnt/raid/aquota.user
```

is not required.

Do not run `quotacheck` simply to create `aquota.user`.

---

# 10. Quota Values

The quota values passed to `setquota` are expressed in KiB:

```text
4 GiB = 4194304 KiB
5 GiB = 5242880 KiB
```

Apply a quota manually:

```bash
sudo setquota -u <UID> 4194304 5242880 0 0 /mnt/raid
```

Example using a fictional UID:

```bash
sudo setquota -u 1001001 4194304 5242880 0 0 /mnt/raid
```

View quotas:

```bash
sudo repquota -s -n /mnt/raid
```

---

# 11. Student Storage Root

All roaming student homes are stored under:

```text
/mnt/raid/homes
```

Create it:

```bash
sudo mkdir -p /mnt/raid/homes
sudo chown root:root /mnt/raid/homes
sudo chmod 0711 /mnt/raid/homes
```

The parent directory uses:

```text
owner: root:root
mode: 0711
```

Individual student homes use:

```text
owner: FreeIPA UID/GID
mode: 0700
```

Example:

```text
/mnt/raid/homes/student01
```

should be owned by the numeric UID/GID assigned to `student01` by FreeIPA.

---

# 12. Why `/mnt/raid/homes` Uses 0711

The parent directory:

```text
/mnt/raid/homes
```

uses mode:

```text
0711
```

This allows users to traverse the path to their own home while preventing ordinary users from listing the entire student-home directory.

Each user's own directory uses:

```text
0700
```

so other students cannot browse that user's files.

---

# 13. Install the NFS Server

On the storage server:

```bash
sudo apt update
sudo apt install -y nfs-kernel-server nfs-common
```

Enable NFS:

```bash
sudo systemctl enable --now nfs-server
```

Check:

```bash
systemctl status nfs-server
```

On Debian it may show:

```text
active (exited)
```

while the kernel NFS services are functioning normally.

Check port 2049:

```bash
sudo ss -lntp | grep ':2049'
```

Optional RPC inspection:

```bash
rpcinfo -p
```

---

# 14. NFS Export File

CASTLE uses:

```text
/etc/exports.d/castle-homes.exports
```

rather than placing the configuration directly in `/etc/exports`.

Example with several static workstation addresses:

```exports
/mnt/raid/homes 192.0.2.101(rw,sync,no_subtree_check,root_squash,sec=sys) 192.0.2.102(rw,sync,no_subtree_check,root_squash,sec=sys) 192.0.2.103(rw,sync,no_subtree_check,root_squash,sec=sys)
```

Add every authorized CASTLE workstation using its privately assigned static address.

Do not publish the production client IP list.

---

# 15. Reload NFS Exports

After editing the file:

```bash
sudo exportfs -rav
```

Check the active configuration:

```bash
sudo exportfs -v
```

Check the service:

```bash
systemctl is-active nfs-server
```

Expected:

```text
active
```

---

# 16. NFS Options

The current CASTLE export uses:

```text
rw
sync
no_subtree_check
root_squash
sec=sys
```

`rw` allows students to write to their home directories.

`sync` uses synchronous server-side NFS write behavior.

`no_subtree_check` disables subtree checking.

`root_squash` prevents root on a workstation from automatically becoming root on the NFS server.

`sec=sys` uses UNIX numeric UID/GID credentials.

Keep:

```text
root_squash
```

enabled.

Do not normally use:

```text
no_root_squash
```

for student workstations.

---

# 17. Security Limitation of `sec=sys`

With:

```text
sec=sys
```

the NFS server trusts the UID/GID credentials provided by authorized client machines.

FreeIPA ensures users receive consistent UID/GID values across the lab.

Students therefore should not have unrestricted root or sudo access on the CASTLE workstations.

A future hardening option is Kerberized NFS:

```text
sec=krb5p
```

which provides stronger authentication and privacy.

The current lab uses `sec=sys`.

---

# 18. Why Individual Static Addresses Are Used

The NFS export is restricted to specific CASTLE workstations.

The lab does not simply export storage to an entire building subnet.

For example, this broad configuration is avoided:

```exports
/mnt/raid/homes 192.0.2.0/24(...)
```

Instead, only managed systems receive access:

```text
lab01 → static IP
lab02 → static IP
lab03 → static IP
...
```

This reduces unnecessary exposure to unrelated devices on the same network.

---

# 19. Static Address Requirement

The CASTLE NFS ACL is IP-based.

Each workstation should therefore have a stable address assigned by the network administrators.

The Linux machine can use either:

```text
manually configured static addressing
```

or:

```text
DHCP reservation
```

provided the workstation reliably receives the same address.

FreeIPA authentication itself does not require the client IP to remain static.

The static address is required here because of the NFS authorization design.

---

# 20. FreeIPA and DNS

A FreeIPA host object can exist even if the corresponding hostname is not available through the organization's DNS infrastructure.

For example, a client may have:

```text
host/lab01.example.test@EXAMPLE.TEST
```

as a valid Kerberos principal without:

```text
lab01.example.test
```

resolving through DNS.

Because of this, CASTLE uses static IP addresses in the NFS export rather than client hostnames.

The FreeIPA server itself must still be resolvable/reachable by clients.

---

# 21. FreeIPA Default Home Directory

The default FreeIPA home-directory base is:

```text
/home/students
```

The default shell is:

```text
/bin/bash
```

Example configuration:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin

ipa config-mod \
    --homedirectory=/home/students \
    --defaultshell=/bin/bash
'
```

Verify:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin
ipa config-show
'
```

A new account such as:

```text
student01
```

should therefore receive:

```text
/home/students/student01
```

---

# 22. Why `/home/students` Is Used

CASTLE intentionally does not replace the entire:

```text
/home
```

directory.

Roaming student accounts use:

```text
/home/students
```

Local or special accounts may remain under paths such as:

```text
/home/localadmin
/home/localguest
```

This prevents CASTLE's autofs configuration from taking over every local home directory.

---

# 23. FreeIPA Automount Architecture

CASTLE uses the FreeIPA automount location:

```text
default
```

The layout is:

```text
auto.master

/home/students → auto.students
```

and:

```text
auto.students

* → -fstype=nfs4,vers=4.2,rw,hard 192.0.2.10:/mnt/raid/homes/&
```

Replace:

```text
192.0.2.10
```

with the private NFS server address.

---

# 24. Create `auto.students`

Example:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
set -e

kinit admin

ipa automountmap-show default auto.students >/dev/null 2>&1 || \
    ipa automountmap-add default auto.students
'
```

Link it into `auto.master`:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin

ipa automountkey-add default auto.master \
    --key=/home/students \
    --info=auto.students
'
```

Add the wildcard:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin

ipa automountkey-add default auto.students \
    --key="*" \
    --info="-fstype=nfs4,vers=4.2,rw,hard 192.0.2.10:/mnt/raid/homes/&"
'
```

If rebuilding an existing environment, inspect the keys first so duplicate entries are not added.

---

# 25. Critical Wildcard Detail

The automount key must be exactly:

```text
*
```

Do not store:

```text
\*
```

The wildcard:

```text
*
```

matches the requested subdirectory name.

The:

```text
&
```

in the NFS path is replaced with that value.

Example:

```text
/home/students/student01
```

becomes:

```text
192.0.2.10:/mnt/raid/homes/student01
```

---

# 26. Check the Automount Configuration

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin

echo "Automount maps:"
ipa automountmap-find default

echo
echo "Master map:"
ipa automountkey-find default auto.master

echo
echo "Student map:"
ipa automountkey-find default auto.students
'
```

Expected architecture:

```text
/home/students → auto.students
```

and:

```text
* → -fstype=nfs4,vers=4.2,rw,hard <SERVER_IP>:/mnt/raid/homes/&
```

---

# 27. CASTLE Student Group

Create a FreeIPA group:

```text
labstudents
```

Description:

```text
CASTLE lab students with roaming RAID home directories
```

Create it:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin

ipa group-show labstudents >/dev/null 2>&1 || \
    ipa group-add labstudents \
    --desc="CASTLE lab students with roaming RAID home directories"
'
```

This group is used as the scope for automatic home provisioning.

---

# 28. FreeIPA Automember Rule

Normal new students are automatically placed into:

```text
labstudents
```

based on their FreeIPA home directory.

Create the rule:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
set -e

kinit admin

ipa automember-show labstudents --type=group >/dev/null 2>&1 || \
    ipa automember-add labstudents \
    --type=group \
    --desc="Automatically add users whose home is under /home/students"

ipa automember-add-condition labstudents \
    --type=group \
    --key=homeDirectory \
    --inclusive-regex="^/home/students/.*$"
'
```

The rule matches users whose home looks like:

```text
/home/students/student01
```

---

# 29. Verify Automember

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin

echo "Student group:"
ipa group-show labstudents

echo
echo "Automember rule:"
ipa automember-show labstudents \
    --type=group \
    --all
'
```

Expected condition:

```text
homeDirectory=^/home/students/.*$
```

---

# 30. Rebuild Membership for an Existing User

If an existing user's home already matches the rule:

```bash
ipa automember-rebuild --users=<USERNAME>
```

Example:

```bash
ipa automember-rebuild --users=student01
```

This is useful when introducing the rule after accounts already exist.

---

# 31. Manual Provisioning Tool

CASTLE keeps a manual tool at:

```text
/usr/local/sbin/castle-home-add
```

It is useful for:

```text
manual provisioning
repairing an account
migrating an older account
troubleshooting automatic provisioning
```

Normal use:

```bash
sudo castle-home-add student01
```

Migration use:

```bash
sudo castle-home-add --migrate student01
```

---

# 32. What `castle-home-add` Does

The utility verifies:

```text
/mnt/raid is mounted
user quotas are enabled
FreeIPA is available
the FreeIPA account exists
the account has a valid UID and GID
the home path matches /home/students/<username>
```

It then:

```text
creates /mnt/raid/homes/<username>
sets FreeIPA UID/GID ownership
sets mode 0700
sets 4 GiB / 5 GiB quota
verifies the result
```

If an existing directory has unexpected ownership, the tool should refuse to silently take ownership of it.

---

# 33. Reliable Quota Check

The quota test used by the CASTLE provisioning tools is:

```bash
QUOTA_STATUS="$(quotaon -p "$RAID" 2>&1 || true)"

if ! awk '/^user quota on / && / is on$/ {ok=1} END {exit ok ? 0 : 1}' <<< "$QUOTA_STATUS"; then
    echo "User quotas are not enabled on $RAID."
    echo "Current quota status:"
    printf '%s\n' "$QUOTA_STATUS"
    exit 1
fi
```

This prevents provisioning when quota enforcement is unexpectedly disabled.

---

# 34. Existing Account Migration

Older accounts may have been configured with homes such as:

```text
/home/student01
```

instead of:

```text
/home/students/student01
```

Before migrating an established user:

```text
check the old local home
copy any important local files
confirm the user understands the migration
```

Then run:

```bash
sudo castle-home-add --migrate student01
```

Do not blindly migrate established accounts because mounting an NFS home can hide files that remain underneath the mount point.

---

# 35. Automatic Home Provisioning

Normal new accounts are automatically provisioned.

Workflow:

```text
Admin creates FreeIPA user
          │
          ▼
FreeIPA assigns
/home/students/<username>
          │
          ▼
Automember adds user to
labstudents
          │
          ▼
castle-home-sync runs
          │
          ▼
/mnt/raid/homes/<username>
          │
          ├── correct UID/GID
          ├── mode 0700
          └── quota 4 GiB / 5 GiB
```

---

# 36. Host-Keytab Authentication

The automatic provisioning system does not store the FreeIPA administrator password.

It uses the server host keytab.

Example principal:

```text
host/ipa01.example.test@EXAMPLE.TEST
```

Test access:

```bash
sudo podman exec -it freeipa-server-container bash -lc '
set -e

export KRB5CCNAME=FILE:/tmp/castle-provisioner-test.ccache
rm -f /tmp/castle-provisioner-test.ccache

echo "Keytab principals:"
klist -k /etc/krb5.keytab

echo
echo "Getting a temporary host ticket..."
kinit -k \
    -t /etc/krb5.keytab \
    host/ipa01.example.test@EXAMPLE.TEST

echo
echo "Current ticket:"
klist

echo
echo "Checking the student group:"
ipa group-show labstudents --all

rm -f /tmp/castle-provisioner-test.ccache
'
```

Replace the example principal with the private deployment principal.

Never publish the actual keytab.

---

# 37. Automatic Provisioning Script

Path:

```text
/usr/local/sbin/castle-home-sync
```

Recommended public-safe implementation:

```bash
#!/usr/bin/env bash
set -euo pipefail

RAID="/mnt/raid"
HOME_ROOT="/mnt/raid/homes"
IPA_CONTAINER="freeipa-server-container"
IPA_GROUP="labstudents"

# Replace with the real private host principal.
IPA_HOST_PRINCIPAL="host/ipa01.example.test@EXAMPLE.TEST"

# Values are in KiB.
SOFT_QUOTA=4194304
HARD_QUOTA=5242880

if [[ $EUID -ne 0 ]]; then
    echo "This needs to run as root."
    exit 1
fi

# Prevent timer and manual runs from overlapping.
exec 9>/run/castle-home-sync.lock

if ! flock -n 9; then
    echo "A CASTLE home sync is already running."
    exit 0
fi

echo "CASTLE student home sync"
echo "Started: $(date)"
echo

# Do not create directories if the RAID has failed to mount.
if ! mountpoint -q "$RAID"; then
    echo "$RAID is not mounted. Stopping."
    exit 1
fi

if ! podman container exists "$IPA_CONTAINER"; then
    echo "FreeIPA container '$IPA_CONTAINER' was not found."
    exit 1
fi

if [[ "$(podman inspect -f '{{.State.Running}}' "$IPA_CONTAINER")" != "true" ]]; then
    echo "FreeIPA container '$IPA_CONTAINER' is not running."
    exit 1
fi

QUOTA_STATUS="$(quotaon -p "$RAID" 2>&1 || true)"

if ! awk '/^user quota on / && / is on$/ {ok=1} END {exit ok ? 0 : 1}' <<< "$QUOTA_STATUS"; then
    echo "User quotas are not active on $RAID."
    printf '%s\n' "$QUOTA_STATUS"
    exit 1
fi

mkdir -p "$HOME_ROOT"
chown root:root "$HOME_ROOT"
chmod 0711 "$HOME_ROOT"

echo "Reading members of '$IPA_GROUP'..."

mapfile -t RECORDS < <(
    podman exec \
        -e IPA_HOST_PRINCIPAL="$IPA_HOST_PRINCIPAL" \
        "$IPA_CONTAINER" \
        bash -lc '
            set -euo pipefail

            export KRB5CCNAME="FILE:/tmp/castle-home-sync.ccache.$$"

            cleanup() {
                rm -f "$KRB5CCNAME"
            }

            trap cleanup EXIT

            kinit -k \
                -t /etc/krb5.keytab \
                "$IPA_HOST_PRINCIPAL"

            MEMBERS="$(
                ipa group-show labstudents |
                sed -n "s/^  Member users: //p" |
                tr "," "\n" |
                sed "s/^[[:space:]]*//;s/[[:space:]]*$//" |
                sed "/^$/d"
            )"

            for USERNAME in $MEMBERS; do
                DATA="$(ipa user-show "$USERNAME" --all --raw)"

                USER_UID="$(
                    printf "%s\n" "$DATA" |
                    awk -F": " "/^[[:space:]]*uidnumber:/ {print \$2; exit}"
                )"

                USER_GID="$(
                    printf "%s\n" "$DATA" |
                    awk -F": " "/^[[:space:]]*gidnumber:/ {print \$2; exit}"
                )"

                USER_HOME="$(
                    printf "%s\n" "$DATA" |
                    awk -F": " "/^[[:space:]]*homedirectory:/ {print \$2; exit}"
                )"

                printf "%s\t%s\t%s\t%s\n" \
                    "$USERNAME" \
                    "$USER_UID" \
                    "$USER_GID" \
                    "$USER_HOME"
            done
        '
)

if [[ ${#RECORDS[@]} -eq 0 ]]; then
    echo "No users are currently in $IPA_GROUP."
    exit 0
fi

echo "Found ${#RECORDS[@]} student account(s)."
echo

ERRORS=0
PROVISIONED=0
OK=0

for RECORD in "${RECORDS[@]}"; do
    IFS=$'\t' read -r \
        USERNAME USER_UID USER_GID USER_HOME \
        <<< "$RECORD"

    echo "$USERNAME"

    if [[ ! "$USERNAME" =~ ^[A-Za-z0-9._-]+$ ]]; then
        echo "  Username is not valid. Skipping."
        ERRORS=$((ERRORS + 1))
        echo
        continue
    fi

    if [[ ! "$USER_UID" =~ ^[0-9]+$ || ! "$USER_GID" =~ ^[0-9]+$ ]]; then
        echo "  FreeIPA did not return a valid UID/GID."
        ERRORS=$((ERRORS + 1))
        echo
        continue
    fi

    EXPECTED_HOME="/home/students/$USERNAME"
    RAID_HOME="$HOME_ROOT/$USERNAME"

    if [[ "$USER_HOME" != "$EXPECTED_HOME" ]]; then
        echo "  Home is $USER_HOME, not $EXPECTED_HOME."
        echo "  Leaving this account alone."
        echo
        continue
    fi

    if [[ -e "$RAID_HOME" ]]; then
        if [[ ! -d "$RAID_HOME" ]]; then
            echo "  $RAID_HOME exists but is not a directory."
            ERRORS=$((ERRORS + 1))
            echo
            continue
        fi

        CURRENT_OWNER="$(stat -c '%u:%g' "$RAID_HOME")"
        EXPECTED_OWNER="${USER_UID}:${USER_GID}"

        if [[ "$CURRENT_OWNER" != "$EXPECTED_OWNER" ]]; then
            echo "  Existing home has unexpected ownership."
            echo "  Current:  $CURRENT_OWNER"
            echo "  Expected: $EXPECTED_OWNER"
            echo "  Leaving it untouched."
            ERRORS=$((ERRORS + 1))
            echo
            continue
        fi

        chmod 0700 "$RAID_HOME"

        setquota -u "$USER_UID" \
            "$SOFT_QUOTA" \
            "$HARD_QUOTA" \
            0 \
            0 \
            "$RAID"

        echo "  Existing roaming home is OK."
        OK=$((OK + 1))
    else
        install -d \
            -m 0700 \
            -o "$USER_UID" \
            -g "$USER_GID" \
            "$RAID_HOME"

        setquota -u "$USER_UID" \
            "$SOFT_QUOTA" \
            "$HARD_QUOTA" \
            0 \
            0 \
            "$RAID"

        echo "  Created $RAID_HOME"
        echo "  Owner: $USER_UID:$USER_GID"
        echo "  Quota: 4 GiB soft / 5 GiB hard"

        PROVISIONED=$((PROVISIONED + 1))
    fi

    echo
done

echo "Sync finished."
echo "Existing homes checked: $OK"
echo "New homes created:      $PROVISIONED"
echo "Errors:                 $ERRORS"

if (( ERRORS > 0 )); then
    exit 1
fi
```

Install:

```bash
sudo chmod 0755 /usr/local/sbin/castle-home-sync
```

---

# 38. systemd Service

Create:

```text
/etc/systemd/system/castle-home-sync.service
```

Contents:

```ini
[Unit]
Description=Synchronize CASTLE FreeIPA labstudents roaming homes
After=network-online.target nfs-server.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/castle-home-sync
User=root
Group=root
```

A successful oneshot service normally becomes:

```text
inactive (dead)
```

after it finishes.

That is expected.

---

# 39. systemd Timer

Create:

```text
/etc/systemd/system/castle-home-sync.timer
```

Contents:

```ini
[Unit]
Description=Periodically provision CASTLE roaming student homes

[Timer]
OnBootSec=10min
OnUnitActiveSec=4h
AccuracySec=5min
Unit=castle-home-sync.service

[Install]
WantedBy=timers.target
```

The job therefore runs approximately:

```text
10 minutes after boot
```

and then:

```text
every 4 hours
```

This interval is appropriate for a small lab where accounts are not created continuously.

---

# 40. Enable Automatic Provisioning

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now castle-home-sync.timer
```

Check:

```bash
systemctl list-timers castle-home-sync.timer --no-pager
```

Run immediately if needed:

```bash
sudo systemctl start castle-home-sync.service
```

Check logs:

```bash
sudo journalctl \
    -u castle-home-sync.service \
    -n 100 \
    --no-pager
```

A healthy run should end with:

```text
Errors: 0
```

---

# 41. Normal Student Account Workflow

Once CASTLE is deployed, the normal account workflow is:

```text
FreeIPA UI
    │
    ▼
Create student
    │
    ▼
/home/students/<username>
    │
    ▼
Automember → labstudents
    │
    ▼
castle-home-sync
    │
    ▼
RAID home created automatically
```

The administrator normally does not need to manually:

```text
add the user to labstudents
create the RAID home
set ownership
set permissions
set quota
create an automount entry
```

If immediate provisioning is required:

```bash
sudo systemctl start castle-home-sync.service
```

## 42. Enrolling a New CASTLE Workstation

Use this procedure when adding a new Linux workstation to CASTLE.

> Existing CASTLE workstations that were enrolled under the older configuration do **not** need to be removed from FreeIPA or re-enrolled. If roaming homes are already working on those machines, leave them as they are.

### Set the Workstation Hostname

Before enrolling the workstation, configure its permanent hostname and network settings.

Example:

```bash
sudo hostnamectl set-hostname lab01.example.test
```

Verify the hostname:

```bash
hostname -f
```

The result should be the workstation's intended fully qualified domain name:

```text
lab01.example.test
```

Make sure the workstation can reach the FreeIPA server before continuing.

---

### Install the Required Packages

Install the FreeIPA client along with the packages required for CASTLE roaming home directories:

```bash
sudo apt update

sudo apt install -y \
    freeipa-client \
    nfs-common \
    autofs \
    sssd-tools
```

The packages provide:

- `freeipa-client` — FreeIPA enrollment and authentication
- `nfs-common` — NFS client support
- `autofs` — automatic mounting of roaming home directories
- `sssd-tools` — SSSD cache and troubleshooting utilities

---

### Enroll the Workstation in FreeIPA

Enroll the workstation:

```bash
sudo ipa-client-install \
    --server=ipa01.example.test \
    --domain=example.test \
    --realm=EXAMPLE.TEST
```

Replace the example server, domain, and realm with the private CASTLE deployment values.

### Important: Do Not Use `--mkhomedir`

New CASTLE clients should **not** be enrolled with:

```text
--mkhomedir
```

CASTLE no longer uses locally created student home directories.

Student homes are stored centrally on the CASTLE storage server and mounted through NFS and autofs.

The intended layout is:

```text
/home/students/<username>
        ↓
autofs
        ↓
NFS
        ↓
/mnt/raid/homes/<username>
```

Using `--mkhomedir` on new machines is unnecessary and may create local home directories when CASTLE expects the user's home to come from the central NFS server.

---

### Verify FreeIPA Enrollment

After enrollment, confirm that FreeIPA users can be resolved:

```bash
getent passwd student01
```

Use a valid test account from the private CASTLE environment.

The user's home directory should eventually appear as:

```text
/home/students/student01
```

At this point, FreeIPA authentication may already work, but the workstation still needs to be configured to retrieve the centralized automount maps.

---

## 43. Configure FreeIPA Automount

After the workstation has successfully joined FreeIPA, configure FreeIPA automount support.

Run:

```bash
sudo ipa-client-automount \
    --server=ipa01.example.test \
    --location=default
```

Replace `ipa01.example.test` with the private FreeIPA server hostname.

The client must already be enrolled in FreeIPA before this command is run.

This configures SSSD and autofs so the workstation can retrieve CASTLE's centrally managed automount maps from FreeIPA.

---

### Refresh SSSD and Autofs

Clear cached FreeIPA information:

```bash
sudo /usr/sbin/sss_cache -E || true
```

Restart SSSD:

```bash
sudo systemctl restart sssd
```

Enable and start autofs:

```bash
sudo systemctl enable --now autofs
```

Restart autofs:

```bash
sudo systemctl restart autofs
```

---

### Verify the Services

Check SSSD:

```bash
systemctl is-active sssd
```

Expected:

```text
active
```

Check autofs:

```bash
systemctl is-active autofs
```

Expected:

```text
active
```

---

### Verify the FreeIPA Automount Maps

Display the automount maps received by the workstation:

```bash
sudo /usr/sbin/automount -m
```

The CASTLE configuration should include a mount point similar to:

```text
Mount point: /home/students
instance type(s): sss
map: auto.students
```

The `auto.students` map should contain a wildcard entry similar to:

```text
* | -fstype=nfs4,vers=4.2,rw,hard <NFS_SERVER_IP>:/mnt/raid/homes/&
```

Replace `<NFS_SERVER_IP>` with the private address of the CASTLE NFS server.

The wildcard allows one automount rule to handle every student.

For example:

```text
/home/students/student01
```

maps to:

```text
<NFS_SERVER_IP>:/mnt/raid/homes/student01
```

---

### Verify the User Home Directory

Check a FreeIPA student account:

```bash
getent passwd student01
```

The home-directory field should point to:

```text
/home/students/student01
```

If it still shows an older value such as:

```text
/home/student01
```

clear the SSSD cache and restart the services again:

```bash
sudo /usr/sbin/sss_cache -E
sudo systemctl restart sssd
sudo systemctl restart autofs
```

Then check again:

```bash
getent passwd student01
```

---

### Verify NFS Connectivity

Confirm that the workstation can reach the CASTLE NFS server on TCP port 2049:

```bash
timeout 3 bash -c '</dev/tcp/<NFS_SERVER_IP>/2049'
```

If the command exits without an error, the NFS service is reachable.

The workstation's static or reserved IP address must also be listed in the CASTLE NFS export ACL on the server.

---

### Test the Roaming Home

Log into the workstation using a CASTLE student account.

Check the user's home:

```bash
echo "$HOME"
```

Expected:

```text
/home/students/student01
```

Check the actual mount:

```bash
findmnt -T "$HOME"
```

The source should point to the CASTLE NFS server.

Create a test file:

```bash
echo "CASTLE roaming-home test from $(hostname)" > ~/castle-test.txt
```

Verify it:

```bash
cat ~/castle-test.txt
```

Log into another configured CASTLE workstation with the same account and run:

```bash
cat ~/castle-test.txt
```

The file should appear immediately because both workstations are using the same NFS-backed home directory.

After testing:

```bash
rm -f ~/castle-test.txt
```

---

## 44. Legacy CASTLE Workstations

Some CASTLE workstations may have originally been enrolled using the older command:

```bash
sudo ipa-client-install \
    --server=ipa01.example.test \
    --domain=example.test \
    --realm=EXAMPLE.TEST \
    --mkhomedir
```

These machines do **not** need to be removed from FreeIPA or re-enrolled simply because CASTLE later moved to centralized roaming home directories.

If a legacy workstation already has:

- working FreeIPA authentication;
- `nfs-common` installed;
- `autofs` installed and running;
- SSSD retrieving the FreeIPA automount maps;
- access to the CASTLE NFS server;
- successful roaming-home tests;

then leave the workstation enrolled as-is.

For all **new** CASTLE workstations, use the current enrollment process **without `--mkhomedir`**.

---

## 45. New CASTLE Workstation Checklist

A newly deployed CASTLE workstation should pass all of the following checks:

```text
[ ] Permanent hostname configured
[ ] Correct static or reserved IP assigned
[ ] FreeIPA server reachable
[ ] freeipa-client installed
[ ] nfs-common installed
[ ] autofs installed
[ ] sssd-tools installed
[ ] Workstation enrolled in FreeIPA
[ ] --mkhomedir was NOT used
[ ] ipa-client-automount configured
[ ] SSSD active
[ ] autofs active
[ ] /home/students automount map visible
[ ] NFS server reachable on TCP 2049
[ ] Workstation IP authorized by the NFS server
[ ] FreeIPA user's home is /home/students/<username>
[ ] User can log in successfully
[ ] Files created on one workstation appear on another
```

# 46. Expected Client Results

A configured client should report:

```text
SSSD: active
autofs: active
```

`getent passwd <username>` should show:

```text
/home/students/<username>
```

The automount map should include:

```text
Mount point: /home/students
instance type(s): sss
map: auto.students
```

and a wildcard entry similar to:

```text
* | -fstype=nfs4,vers=4.2,rw,hard 192.0.2.10:/mnt/raid/homes/&
```

---

# 47. SSSD Cache Refresh

When a FreeIPA user attribute changes, clients may retain an older cached value.

Clear the cache:

```bash
sudo /usr/sbin/sss_cache -E
```

Restart:

```bash
sudo systemctl restart sssd
sudo systemctl restart autofs
```

Check the user again:

```bash
getent passwd student01
```

---

# 48. `/usr/sbin` and Root Shells

If root access is obtained with:

```bash
su
```

instead of:

```bash
su -
```

`/usr/sbin` may not be present in the shell's `PATH`.

Using full paths avoids confusion:

```bash
/usr/sbin/sss_cache
/usr/sbin/automount
```

---

# 49. Verify NFS Connectivity

From a client:

```bash
timeout 3 bash -c '</dev/tcp/192.0.2.10/2049'
```

If successful, TCP port 2049 is reachable.

The server must also authorize that client's static IP in:

```text
/etc/exports.d/castle-homes.exports
```

---

# 50. End-to-End SSH Test

A useful final test is to log into two different workstations using the same FreeIPA student account.

From an administrator system:

```bash
ssh student01@192.0.2.101 '
echo "Host: $(hostname)"
echo "User: $(whoami)"
echo "HOME: $HOME"

findmnt -T "$HOME" || true

echo "Created on $(hostname) at $(date)" > \
    "$HOME/castle-roaming-test.txt"

cat "$HOME/castle-roaming-test.txt"
'
```

Then connect to another workstation:

```bash
ssh student01@192.0.2.102 '
echo "Host: $(hostname)"
echo "User: $(whoami)"
echo "HOME: $HOME"

findmnt -T "$HOME" || true

cat "$HOME/castle-roaming-test.txt"
'
```

The second workstation should immediately see the file created on the first.

---

# 51. Why Files Appear Immediately

CASTLE is not syncing files between workstations.

Both workstations are accessing:

```text
the same directory on the NFS server
```

For example:

```text
workstation A:
/home/students/student01
        │
        ▼
server:
/mnt/raid/homes/student01
        ▲
        │
workstation B:
/home/students/student01
```

Therefore no synchronization service or waiting period is needed.

---

# 52. Downloads, Documents, Desktop, Etc.

Directories such as:

```text
Downloads
Documents
Desktop
Pictures
```

are normally stored inside the user's home.

For example:

```text
/home/students/student01/Downloads
```

actually resides on the server as:

```text
/mnt/raid/homes/student01/Downloads
```

A file downloaded on one CASTLE workstation should therefore appear when the same user logs into another CASTLE workstation.

Files saved outside the roaming home, such as:

```text
/tmp
/local-storage
another local disk
```

do not automatically roam.

---

# 53. Verify a Home Mount

While logged in as a student:

```bash
echo "$HOME"
```

Expected:

```text
/home/students/student01
```

Inspect the mount:

```bash
findmnt -T "$HOME"
```

The source should point to the central NFS server.

---

# 54. Test File Creation

On one workstation:

```bash
echo "CASTLE test from $(hostname)" > ~/castle-test.txt
cat ~/castle-test.txt
```

On another workstation using the same account:

```bash
cat ~/castle-test.txt
```

The contents should match immediately.

---

# 55. Cleaning Up Test Files

Delete a test file from the user's session:

```bash
rm -f ~/castle-test.txt
```

Or from the server:

```bash
sudo rm -f \
    /mnt/raid/homes/student01/castle-test.txt
```

Because it is the same centralized home, deleting it from the server removes it from every workstation.

Be careful when deleting directly from `/mnt/raid/homes`.

---

# 56. Finding Known Test Files

Before deleting multiple files, inspect them first:

```bash
sudo find /mnt/raid/homes \
    -maxdepth 2 \
    -name 'castle-*-test.txt' \
    -print
```

Only delete files after verifying the output contains exactly what is intended.

---

# 57. NFS Export Backup Before Changes

Before editing the NFS ACL:

```bash
sudo cp \
    /etc/exports.d/castle-homes.exports \
    "/etc/exports.d/castle-homes.exports.backup-$(date +%F-%H%M%S)"
```

This creates a timestamped rollback copy.

---

# 58. Restore the Latest NFS Export Backup

```bash
BACKUP="$(
    ls -1t /etc/exports.d/castle-homes.exports.backup-* \
    2>/dev/null |
    head -n1
)"

if [[ -z "$BACKUP" ]]; then
    echo "No NFS export backup was found."
    exit 1
fi

echo "Restoring $BACKUP"

sudo cp \
    "$BACKUP" \
    /etc/exports.d/castle-homes.exports

sudo exportfs -rav
sudo exportfs -v
```

---

# 59. Server RAID Health Check

```bash
echo "RAID status:"
cat /proc/mdstat

echo
echo "Detailed array information:"
sudo mdadm --detail /dev/md0
```

---

# 60. Filesystem Health Check

```bash
echo "RAID mount:"
findmnt /mnt/raid

echo
echo "Space:"
df -h /mnt/raid

echo
echo "Filesystem:"
sudo blkid /dev/md0
```

---

# 61. Quota Health Check

```bash
echo "Quota status:"
sudo quotaon -p /mnt/raid

echo
echo "Quota report:"
sudo repquota -s -n /mnt/raid
```

---

# 62. Student Home Health Check

```bash
sudo find /mnt/raid/homes \
    -mindepth 1 \
    -maxdepth 1 \
    -type d \
    -printf '%f UID=%U GID=%G MODE=%m\n' |
    sort
```

Each student directory should normally have:

```text
mode 700
```

and ownership matching the user's FreeIPA UID/GID.

---

# 63. NFS Health Check

```bash
echo "NFS service:"
systemctl is-active nfs-server

echo
echo "Active exports:"
sudo exportfs -v

echo
echo "NFS listener:"
sudo ss -lntp | grep ':2049' || true
```

---

# 64. Provisioning Health Check

```bash
echo "CASTLE timer:"
systemctl list-timers \
    castle-home-sync.timer \
    --no-pager

echo
echo "Recent provisioning log:"
sudo journalctl \
    -u castle-home-sync.service \
    -n 50 \
    --no-pager
```

---

# 65. FreeIPA CASTLE Health Check

```bash
sudo podman exec -it freeipa-server-container bash -lc '
kinit admin

echo "FreeIPA defaults:"
ipa config-show | grep -E \
"Home directory base|Default shell|Default users group"

echo
echo "CASTLE student group:"
ipa group-show labstudents

echo
echo "Automember rule:"
ipa automember-show \
    labstudents \
    --type=group \
    --all

echo
echo "Automount maps:"
ipa automountmap-find default

echo
echo "Master map:"
ipa automountkey-find \
    default \
    auto.master

echo
echo "Student map:"
ipa automountkey-find \
    default \
    auto.students
'
```

---

# 66. Check One User End to End

Example:

```bash
sudo podman exec -it freeipa-server-container \
    ipa user-show student01
```

Expected home:

```text
/home/students/student01
```

Check storage:

```bash
sudo stat \
    -c 'Path=%n UID=%u GID=%g Mode=%a' \
    /mnt/raid/homes/student01
```

Expected mode:

```text
700
```

Check quota:

```bash
sudo repquota -s -n /mnt/raid
```

Expected policy:

```text
soft: 4096M
hard: 5120M
```

---

# 67. Client Troubleshooting: Home Still Shows `/home/<user>`

If:

```bash
getent passwd student01
```

shows an old local-style home, clear SSSD:

```bash
sudo /usr/sbin/sss_cache -E
sudo systemctl restart sssd
sudo systemctl restart autofs
```

Check again:

```bash
getent passwd student01
```

---

# 68. Client Troubleshooting: Automount Map Missing

Run:

```bash
sudo /usr/sbin/automount -m
```

If `/home/students` is missing, verify:

```text
the machine is enrolled in FreeIPA
ipa-client-automount has been configured
SSSD is running
autofs is running
the FreeIPA automount location is "default"
```

Configure automount if required:

```bash
sudo ipa-client-automount \
    --server=ipa01.example.test \
    --location=default
```

Then restart:

```bash
sudo systemctl restart sssd
sudo systemctl restart autofs
```

---

# 69. Client Troubleshooting: NFS Access Denied

Check the client's IP:

```bash
ip -4 addr
```

Verify that exact address is listed in:

```text
/etc/exports.d/castle-homes.exports
```

on the server.

Then reload:

```bash
sudo exportfs -rav
```

Check:

```bash
sudo exportfs -v
```

---

# 70. Client Troubleshooting: NFS Port Unreachable

From the workstation:

```bash
timeout 3 bash -c '</dev/tcp/192.0.2.10/2049'
```

If it fails, check:

```text
routing
VLAN assignment
server reachability
NFS service
host firewall
network firewall
static IP configuration
```

---

# 71. Client Troubleshooting: Permission Denied as Root

This can be normal.

CASTLE uses:

```text
root_squash
```

so root on a client is not treated as server root over NFS.

Test access as the actual FreeIPA user instead.

---

# 72. Removing a Student

Removing an account requires care because its RAID home may contain data.

Do not automatically delete:

```text
/mnt/raid/homes/<username>
```

just because the FreeIPA account is disabled or removed.

Recommended process:

```text
disable account
confirm retention requirements
archive data if necessary
remove RAID home only after approval
clear quota if appropriate
```

---

# 73. Backups

RAID5 is not sufficient protection for student files.

Production deployments should consider backing up:

```text
/mnt/raid/homes
```

to storage independent of the RAID server.

The backup system should ideally protect against:

```text
accidental deletion
filesystem corruption
server loss
multiple-drive failure
ransomware
```

---

# 74. Security Notes

The current security model includes:

```text
FreeIPA authentication
central UID/GID assignment
private 0700 home directories
root_squash
IP-restricted NFS exports
no student sudo/root access
quota enforcement
host-keytab automation
no stored FreeIPA admin password
```

Potential future improvements include:

```text
Kerberized NFS using sec=krb5p
host firewall rules for TCP 2049
off-server backups
monitoring/alerting for RAID health
monitoring failed provisioning runs
```

---

# 75. Sensitive Files That Must Never Be Published

Do not commit any of the following to a public repository:

```text
/etc/krb5.keytab
Kerberos credential caches
admin passwords
SSH private keys
TLS private keys
production /etc/hosts
real /etc/exports client list
real filesystem UUIDs if considered internal
real DNS server addresses
student account exports
private network diagrams containing addresses
```

---

# 76. Public Documentation Placeholder Rules

Use values such as:

```text
student01
student02
lab01.example.test
ipa01.example.test
192.0.2.10
192.0.2.101
1001001
<RAID_FILESYSTEM_UUID>
```

Never replace these with live internal identifiers in the public version.

Keep real values in private operational documentation.

---

# 77. Normal Day-to-Day Administration

Creating a student normally consists only of:

```text
FreeIPA UI
    ↓
Add user
```

The account receives:

```text
/home/students/<username>
```

Automember adds:

```text
labstudents
```

The next automatic sync creates:

```text
/mnt/raid/homes/<username>
```

with:

```text
FreeIPA UID/GID
mode 0700
4 GiB soft quota
5 GiB hard quota
```

No per-user automount configuration is required.

---

# 78. Immediate Provisioning

If waiting for the four-hour timer is inconvenient:

```bash
sudo systemctl start castle-home-sync.service
```

Check:

```bash
sudo journalctl \
    -u castle-home-sync.service \
    -n 50 \
    --no-pager
```

---

# 79. Manual Recovery Provisioning

For a normal account:

```bash
sudo castle-home-add <username>
```

For an old account requiring home migration:

```bash
sudo castle-home-add --migrate <username>
```

Only use `--migrate` after checking local files.

---

# 80. Full Rebuild Order

If rebuilding CASTLE from an already functioning FreeIPA server:

```text
1. Install mdadm
2. Build or assemble RAID5
3. Create ext4 filesystem
4. Enable ext4 user quota
5. Configure /etc/fstab
6. Mount /mnt/raid
7. Verify quota enforcement
8. Create /mnt/raid/homes
9. Install NFS server
10. Assign stable client addresses
11. Configure NFS client ACL
12. Reload NFS exports
13. Set FreeIPA home base to /home/students
14. Create auto.students
15. Link /home/students in auto.master
16. Add wildcard * automount rule
17. Create labstudents
18. Create Automember condition
19. Install castle-home-add
20. Verify server host-keytab read access
21. Install castle-home-sync
22. Install systemd service
23. Install systemd timer
24. Enable provisioning timer
25. Enroll client workstations in FreeIPA
26. Configure ipa-client-automount
27. Install nfs-common/autofs/sssd-tools
28. Restart SSSD and autofs
29. Verify automount map
30. Verify NFS connectivity
31. Test login
32. Create file on one workstation
33. Verify file immediately from another
34. Configure backups
```

---

# 81. Final Architecture

```text
                     FreeIPA
                ipa01.example.test
                         │
                         ▼
                    User account
                         │
                 /home/students
                         │
                         ▼
                   labstudents
                         │
                    Automember
                         │
                         ▼
                castle-home-sync
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
    /mnt/raid/homes/user          quota
             │                  4G / 5G
             │
             ▼
          NFS server
             │
             ▼
       static client ACL
             │
             ▼
      CASTLE workstations
             │
        SSSD + autofs
             │
             ▼
 /home/students/<username>
```

---

# 82. Expected User Experience

A student can:

```text
log into workstation A
download or create files
log out
log into workstation B
see the same Downloads/Documents/Desktop/files
```

because both systems access the same NFS-backed home directory.

There is no file-copy or synchronization delay.

---

# 83. Final Operational State

A completed CASTLE deployment should have:

```text
Healthy RAID5
Mounted ext4 filesystem
User quotas enabled
/mnt/raid/homes configured
NFS server active
Only approved static clients exported
root_squash enabled
FreeIPA authentication working
/home/students as the student home base
FreeIPA wildcard automount configured
labstudents group configured
Automember configured
Automatic home provisioning enabled
4-hour systemd timer enabled
FreeIPA clients enrolled
FreeIPA automount configured on clients
SSSD active
autofs active
NFS connectivity working
roaming files confirmed across multiple clients
```

At that point, normal new-user administration is reduced to creating the student account in FreeIPA.