# CASTLE Lab Printing Setup

## Overview

The CASTLE Lab uses a centralized printing setup.

The Lexmark printer is connected directly to the server `lux` using USB. The server runs CUPS and shares the printer with the Debian lab computers.

The lab computers do not connect directly to the printer. Instead, they send print jobs to `lux`, and `lux` sends the jobs to the Lexmark printer over USB.

Architecture:

```text
Lab Computer
     |
     | IPP / CUPS
     v
lux.alfred.edu
     |
     | USB
     v
Lexmark MS310/MS312 Series
```

The shared printer is named:

```text
LabPrinter
```

The FreeIPA server and print server are both:

```text
lux.alfred.edu
```

The server currently uses the static IP:

```text
149.84.129.206
```

---

# 1. Server-Side Setup

All commands in this section are run on:

```text
lux.alfred.edu
```

## 1.1 Install CUPS

Install CUPS and the CUPS client utilities:

```bash
sudo apt update
sudo apt install cups cups-client
```

Enable and start CUPS:

```bash
sudo systemctl enable --now cups
```

Verify:

```bash
systemctl status cups
```

---

# 2. Verify the USB Printer

The Lexmark printer is physically connected to `lux` using USB.

Check that Linux sees the printer:

```bash
lsusb
```

The printer appeared as:

```text
Lexmark International, Inc. Lexmark MS312dn
```

To find the CUPS device URI:

```bash
sudo /usr/sbin/lpinfo -v
```

The Lexmark USB URI was:

```text
usb://Lexmark/MS310%20Series?serial=45147GLM3V8KL
```

The model may display as `MS310 Series` even though the physical printer is an MS312dn.

---

# 3. Find an Available Printer Driver

The Lexmark is connected over USB, so the CUPS `everywhere` driver cannot be used directly.

Running:

```bash
sudo /usr/sbin/lpinfo -m | grep -Ei 'Lexmark|MS310|MS312|Generic.*PostScript'
```

returned:

```text
drv:///sample.drv/generic.ppd Generic PostScript Printer
```

The Lexmark supports PostScript, so the Generic PostScript driver was used.

---

# 4. Create LabPrinter on lux

Create the printer:

```bash
sudo lpadmin \
  -p LabPrinter \
  -E \
  -v 'usb://Lexmark/MS310%20Series?serial=45147GLM3V8KL' \
  -m drv:///sample.drv/generic.ppd
```

CUPS may display:

```text
Printer drivers are deprecated and will stop working in a future version of CUPS.
```

This is a warning about future CUPS versions. It does not mean the current configuration failed.

Set `LabPrinter` as the server default:

```bash
sudo lpadmin -d LabPrinter
```

Verify:

```bash
lpstat -t
```

The important lines should include:

```text
system default destination: LabPrinter
device for LabPrinter: usb://Lexmark/MS310%20Series?serial=45147GLM3V8KL
printer LabPrinter is idle
```

---

# 5. Test Printing Directly from lux

Send a simple test job:

```bash
echo "CASTLE Lexmark printer test" | lp -d LabPrinter
```

CUPS should return a request ID similar to:

```text
request id is LabPrinter-1
```

Verify that the page physically prints.

---

# 6. Share LabPrinter

Mark the printer as shared:

```bash
sudo lpadmin -p LabPrinter -o printer-is-shared=true
```

Enable printer sharing globally:

```bash
sudo cupsctl --share-printers
```

Restart CUPS:

```bash
sudo systemctl restart cups
```

Verify:

```bash
lpstat -t
```

---

# 7. Configure CUPS Network Access

The CUPS configuration is:

```text
/etc/cups/cupsd.conf
```

The important server-side settings are:

```text
Port 631
Listen /run/cups/cups.sock
Browsing On
BrowseLocalProtocols dnssd
WebInterface Yes
```

Access for local network clients is allowed with:

```text
<Location />
  Order allow,deny
  Allow @LOCAL
</Location>
```

CUPS should be listening on TCP port 631.

Verify:

```bash
sudo ss -ltnp | grep 631
```

Expected:

```text
0.0.0.0:631
[::]:631
```

---

# 8. Fix the CUPS Hostname / Bad Request Issue

Initially, clients could reach `lux`, but CUPS returned:

```text
HTTP/1.1 400 Bad Request
```

and commands such as:

```bash
lpstat -h lux.alfred.edu:631 -p
```

returned:

```text
Error - add '/version=1.1' to server name.
```

The actual issue was that CUPS was rejecting the HTTP `Host:` header for:

```text
lux.alfred.edu
```

Edit:

```bash
sudo nano /etc/cups/cupsd.conf
```

Add near the top:

```text
ServerName lux.alfred.edu
ServerAlias lux.alfred.edu
```

For example:

```text
LogLevel warn
ServerName lux.alfred.edu
ServerAlias lux.alfred.edu

PageLogFormat
MaxLogSize 0
ErrorPolicy retry-job

Port 631
Listen /run/cups/cups.sock
```

Restart CUPS:

```bash
sudo systemctl restart cups
```

Clients should then be able to communicate normally with:

```text
lux.alfred.edu:631
```

---

# 9. Verify the Printer Is Shared

On `lux`:

```bash
sudo grep -A20 -B5 LabPrinter /etc/cups/printers.conf
```

The `LabPrinter` section should contain:

```text
DeviceURI usb://Lexmark/MS310%20Series?serial=45147GLM3V8KL
Accepting Yes
Shared Yes
```

The actual server configuration should remain:

```text
LabPrinter -> USB printer
```

Do not change `LabPrinter` on `lux` to point back to:

```text
ipp://lux.alfred.edu:631/printers/LabPrinter
```

That would create a loop where the print server points to itself.

---

# 10. Configure Lab Computers

The controlled CASTLE Debian computers use `lux` as their central CUPS server.

Current systems include:

```text
bedi.alfred.edu
alan.alfred.edu
ahmad.alfred.edu
daniel.alfred.edu
saron.alfred.edu
dagim.alfred.edu
garegin.alfred.edu
kumar.alfred.edu
```

Install the CUPS client if necessary:

```bash
sudo apt install -y cups-client
```

Create:

```text
/etc/cups/client.conf
```

with:

```text
ServerName lux.alfred.edu:631
```

This can be done using:

```bash
echo 'ServerName lux.alfred.edu:631' | sudo tee /etc/cups/client.conf
```

After this, CUPS client programs send their requests directly to `lux`.

---

# 11. Disable Automatic Printer Discovery

The Alfred University network contains other discoverable printers, including HP and Toshiba printers.

Without disabling discovery, the lab computers may display printers such as:

```text
HP_LaserJet_400_M401dne_1398E3
TOSHIBA_e_STUDIO3525AC_15969935
LabPrinter_lux
```

These may be automatically created by `cups-browsed`.

Disable automatic printer discovery on each CASTLE machine:

```bash
sudo systemctl disable --now cups-browsed
```

After disabling it, verify:

```bash
lpstat -p
lpstat -d
```

The desired output is:

```text
printer LabPrinter is idle
system default destination: LabPrinter
```

In practice, disabling `cups-browsed` caused the unwanted auto-discovered queues to disappear automatically.

---

# 12. Configure Multiple Lab Computers from lux

Once the FreeIPA account `admin` has sudo privileges on the lab machines, the CUPS client configuration can be pushed from `lux`.

Example:

```bash
for h in bedi alan ahmad daniel saron dagim garegin kumar; do
  ssh -t admin@$h.alfred.edu \
    "echo 'ServerName lux.alfred.edu:631' | sudo tee /etc/cups/client.conf"
done
```

This requires entering the FreeIPA password for SSH and sudo.

To disable `cups-browsed` on all machines:

```bash
for h in bedi alan ahmad daniel saron dagim garegin kumar; do
  ssh -t admin@$h.alfred.edu '
    sudo systemctl disable --now cups-browsed
    lpstat -p
    lpstat -d
  '
done
```

A machine that is powered off or unreachable may return:

```text
No route to host
```

It can simply be configured later.

---

# 13. FreeIPA Administrative Sudo Access

The FreeIPA account:

```text
admin
```

was added to the existing FreeIPA group:

```text
admins
```

From the FreeIPA server/container:

```bash
ipa group-add-member admins --users=admin
```

Verify:

```bash
ipa group-show admins
```

The `admins` group should list:

```text
admin
```

---

# 14. FreeIPA Sudo Rule

A FreeIPA sudo rule was created so members of the `admins` group can administer CASTLE machines.

Create the rule:

```bash
ipa sudorule-add castle-admins \
  --desc="Full sudo access for FreeIPA administrators"
```

Add the `admins` group:

```bash
ipa sudorule-add-user castle-admins --groups=admins
```

Allow all commands:

```bash
ipa sudorule-mod castle-admins --cmdcat=all
```

Allow commands to run as any user:

```bash
ipa sudorule-mod castle-admins --runasusercat=all
```

Allow the rule on all FreeIPA hosts:

```bash
ipa sudorule-mod castle-admins --hostcat=all
```

Verify:

```bash
ipa sudorule-show castle-admins --all
```

Important fields should include:

```text
Enabled: True
Host category: all
Command category: all
RunAs User category: all
User Groups: admins
```

---

# 15. Refresh SSSD After Sudo Changes

Some clients did not immediately receive the new FreeIPA sudo rule because SSSD had cached the old information.

On the affected client, become local root:

```bash
su
```

Clear the SSSD cache:

```bash
/usr/sbin/sss_cache -E
```

Restart SSSD:

```bash
systemctl restart sssd
```

Then completely log out and log back in as:

```text
admin
```

Verify group membership:

```bash
id
```

The output should include:

```text
admins
labstudents
```

Verify sudo:

```bash
sudo -l
```

Expected:

```text
User admin may run the following commands:
    (ALL) ALL
```

Then:

```bash
sudo whoami
```

should return:

```text
root
```

---

# 16. Important Note About `su` vs `sudo`

The local Linux root password is independent from FreeIPA.

Therefore:

```bash
su
```

may require a local root password that is different on each machine.

Once the FreeIPA `castle-admins` sudo rule is working, local root access is normally unnecessary.

Use:

```bash
ssh admin@machine.alfred.edu
sudo command
```

instead.

---

# 17. Testing a Client

On any CASTLE workstation:

```bash
lpstat -p
```

Expected:

```text
printer LabPrinter is idle
```

Check the default:

```bash
lpstat -d
```

Expected:

```text
system default destination: LabPrinter
```

Test printing:

```bash
echo "CASTLE printer test" | lp
```

The complete path is:

```text
Debian workstation
       |
       | CUPS / IPP
       v
lux.alfred.edu
       |
       | USB
       v
Lexmark MS310/MS312 Series
```

---

# 18. Printing from GUI Applications

Because `LabPrinter` is the system default printer, users should normally be able to open an application and press:

```text
Ctrl + P
```

The print dialog should show:

```text
LabPrinter
```

as the available/default printer.

This applies to applications such as:

- Firefox / Chromium
- PDF viewers
- LibreOffice
- text editors
- other applications using the system print dialog

---

# 19. Current Desired Client State

Each CASTLE lab machine should have:

```text
/etc/cups/client.conf
```

containing:

```text
ServerName lux.alfred.edu:631
```

`cups-browsed` should be disabled:

```bash
systemctl is-enabled cups-browsed
```

Expected:

```text
disabled
```

The printer check should return:

```bash
lpstat -p
lpstat -d
```

Expected:

```text
printer LabPrinter is idle
system default destination: LabPrinter
```

---

# 20. Current Desired Server State

On `lux`:

```bash
lpstat -v LabPrinter
```

should return:

```text
device for LabPrinter: usb://Lexmark/MS310%20Series?serial=45147GLM3V8KL
```

The printer must remain shared:

```text
Shared Yes
```

The server must listen on port 631:

```bash
sudo ss -ltnp | grep 631
```

and `/etc/cups/cupsd.conf` should include:

```text
ServerName lux.alfred.edu
ServerAlias lux.alfred.edu
Port 631
```

---

# 21. Troubleshooting

## `lpinfo: command not found`

`lpinfo` is normally located in:

```text
/usr/sbin/lpinfo
```

Use:

```bash
sudo /usr/sbin/lpinfo -v
```

---

## `IPP Everywhere driver requires an IPP connection`

This happens when attempting:

```bash
-m everywhere
```

against a USB printer.

For the USB Lexmark on `lux`, use the Generic PostScript driver instead:

```bash
-m drv:///sample.drv/generic.ppd
```

---

## `Unable to create PPD: No IPP attributes`

Do not create an unnecessary local `-m everywhere` queue if the workstation is configured to use `lux` through:

```text
/etc/cups/client.conf
```

The workstation can simply use the central CUPS server.

---

## `HTTP/1.1 400 Bad Request`

Make sure `/etc/cups/cupsd.conf` on `lux` contains:

```text
ServerName lux.alfred.edu
ServerAlias lux.alfred.edu
```

Then:

```bash
sudo systemctl restart cups
```

---

## `lpadmin: Forbidden`

If a workstation uses:

```text
ServerName lux.alfred.edu:631
```

then `lpadmin` may attempt to modify the CUPS configuration on `lux`.

Remote printer administration is intentionally restricted.

This does not indicate a printing problem.

If:

```bash
lpstat -d
```

already returns:

```text
system default destination: LabPrinter
```

nothing needs to be changed.

---

## Random HP/Toshiba Printers Appear

Disable `cups-browsed`:

```bash
sudo systemctl disable --now cups-browsed
```

Then check:

```bash
lpstat -p
```

Only `LabPrinter` should remain.

---

## FreeIPA User Is in `admins` but Cannot Use sudo

Clear the SSSD cache as local root:

```bash
/usr/sbin/sss_cache -E
systemctl restart sssd
```

Log completely out, reconnect, then:

```bash
sudo -l
sudo whoami
```

---

# 22. New Machine Enrollment

The final CASTLE enrollment script should automate:

```text
Fresh Debian installation
        |
        +--> Set hostname
        |
        +--> Install FreeIPA client
        |
        +--> Enroll into FreeIPA
        |
        +--> Configure SSSD
        |
        +--> Configure FreeIPA automount
        |
        +--> RAID/NFS roaming homes
        |
        +--> Configure /etc/cups/client.conf
        |
        +--> ServerName lux.alfred.edu:631
        |
        +--> Disable cups-browsed
        |
        +--> Install CASTLE wallpaper
        |
        +--> Verify LabPrinter
        |
        └--> Reboot
```

The custom deployment folder can contain:

```text
CASTLE-Setup/
├── castle-enroll.sh
└── castle-wallpaper.jpg
```

After running the enrollment script and rebooting, a FreeIPA student should be able to:

1. log in using their FreeIPA credentials;
2. receive their RAID-backed roaming home directory;
3. open a program and press `Ctrl+P`;
4. print using `LabPrinter`;
5. receive the CASTLE lab wallpaper.

---

# 23. Summary

The final CASTLE printing design is:

```text
                    CASTLE LAB
                         |
       +-----------------+-----------------+
       |                 |                 |
      bedi              alan             ahmad
       |                 |                 |
     daniel            saron             dagim
       |                 |                 |
     garegin            kumar             ...
       \                 |                /
        \                |               /
         +---------------+--------------+
                         |
                     CUPS / IPP
                         |
                         v
                  lux.alfred.edu
                         |
                  CUPS LabPrinter
                         |
                        USB
                         |
                         v
               Lexmark MS312dn Printer
```

The CASTLE workstations rely on `lux` for the printer queue, while `lux` handles the physical USB connection to the Lexmark printer.