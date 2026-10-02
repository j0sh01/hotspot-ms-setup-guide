# OpenWrt 25.12.4 + Hotspot MS End-to-End Setup Guide

This guide explains how to build a working OpenWrt + Hotspot MS environment using the official OpenWrt 25.12.4 x86/64 image:

- `https://archive.openwrt.org/releases/25.12.4/targets/x86/64/`
- Image: `generic-squashfs-combined-efi.img.gz`

It is designed for a practical deployment where you keep access to both:

- LuCI web UI
- SSH terminal

The goal is to avoid locking yourself out while you configure the router, install the provisioning script, and enable the captive portal integration.

## 1. What this setup is for

This deployment is meant to integrate:

- OpenWrt router
- openNDS captive portal
- Hotspot MS backend (Frappe app)
- optional hardening and deauth flow

The router acts as the network gateway and portal enforcement point, while Hotspot MS manages vouchers, sessions, and API calls from the cloud.

## 2. Requirements

Before starting, make sure you have the following:

### Hardware

- Mini PC or x86/64 compatible machine with at least 1GB RAM (preferably 4GB+)
- Internal SSD or USB boot disk
- At least 2 NICs, ideally:
  - WAN: upstream internet
  - LAN: local hotspot network
- Optional: serial console or physical access for recovery

### Network

- One upstream internet connection
- Access to your Frappe / Hotspot MS site
- A local test laptop or desktop connected to the router LAN

### Software

- OpenWrt 25.12.4 x86/64 image:
  - `generic-squashfs-combined-efi.img.gz`
- BalenaEtcher, Rufus, or similar flashing tool
- SSH client such as PuTTY, Terminal, or OpenSSH
- Web browser for LuCI

### Frappe / Hotspot MS prerequisites

- Frappe site already running
- `hotspot_ms` app installed and migrated
- `Nas Device` DocType created in Frappe
- Router secret / shared secret available

## 3. Download and flash the OpenWrt image

### Step 3.1: Download the image

Go to:

`https://archive.openwrt.org/releases/25.12.4/targets/x86/64/`

Download:

`generic-squashfs-combined-efi.img.gz`

### Step 3.2: Decompress the image

If needed, extract the `.img` file from the `.gz` archive.

On Linux/macOS:

```bash
gunzip -c generic-squashfs-combined-efi.img.gz > openwrt-25.12.4-x86-64.img
```

### Step 3.3: Flash the image

Use a tool such as:

- BalenaEtcher
- Rufus
- dd on Linux

Example with `dd`:

```bash
sudo dd if=openwrt-25.12.4-x86-64.img of=/dev/sdX bs=4M status=progress conv=fsync
```

Important:

- Replace `/dev/sdX` with your target disk
- Double-check the disk before writing
- Keep a fallback boot method if the device has multiple drives

### Step 3.4: Boot the device

Connect the device to:

- WAN port to internet uplink
- LAN port to a laptop or switch
- Monitor/serial if available

Boot into OpenWrt.

## 4. Keep access to both LuCI and SSH before changing anything

This is the most important part of the deployment.

Do not disable SSH, DHCP, or LAN access until after the router is confirmed working.

### Step 4.1: Log in to the web interface

Open:

`http://192.168.1.1`

Default OpenWrt login is typically:

- Username: `root`
- Password: blank or default (if not changed)

### Step 4.2: Set a root password immediately

In LuCI:

- Go to `System` -> `Administration`
- Set a strong root password
- Keep SSH enabled
- Keep a second admin access method available

### Step 4.3: Enable SSH access if needed

In LuCI:

- `System` -> `Administration`
- Enable `SSH Access`
- Optionally allow `Password authentication`
- Keep `Dropbear` enabled unless you prefer `OpenSSH`

### Step 4.4: Test terminal access

From your laptop:

```bash
ssh root@192.168.1.1
```

If it works, do not change the network or firewall until you finish the initial validation.

## 5. Initial network layout and safe config

Before enabling hotspot features, set a stable network baseline.

### Recommended interface layout

- WAN: upstream internet
- LAN: local hotspot / client network

Typical LAN:

- IP: `192.168.1.1/24`
- DHCP enabled
- DNS forwarding enabled

### In LuCI

Go to:

- `Network` -> `Interfaces`

Configure:

- WAN: DHCP or static according to your ISP
- LAN: static IP `192.168.1.1`

### Validate connectivity

From terminal:

```bash
uci show network
ip addr
ip route
ping -c 3 8.8.8.8
```

Also verify that your laptop can access the internet through the router.

## 6. Create a safe recovery baseline

Before installing custom scripts or firewall rules, take a backup.

### Backup the configuration

From terminal:

```bash
firstboot
/etc/init.d/network restart
/etc/init.d/dropbear restart

uci export > /root/openwrt_uci_backup.txt
cp -r /etc/config /root/config_backup
```

You should also keep a note of:

- root password
- WAN settings
- LAN IP address
- SSH enablement status
- `nas_identifier`
- shared secret

This ensures you can recover if the captive portal or firewall configuration breaks.

## 7. Prepare the Frappe / Hotspot MS side

The router is not the source of truth; Frappe is.

### In the Frappe admin UI

- Open your Frappe site
- Search for `Nas Device`
- Create a new router record

Use fields similar to:

- `Short Name` / `Device Name`: `OpenWrt-Branch-1`
- `IP Address`: `192.168.10.254` or your actual LAN IP
- `NAS Type`: `OpenWrt`
- `Shared Secret`: a strong random password

Then click:

- `Generate FAS Key`

This creates the key used for secure portal/auth communication.

Also keep the router name and secret recorded. These will be used by the generated bootstrap script.

## 8. Generate the provisioning script from Frappe

On the saved `Nas Device` record:

- Click `Generate Provisioning Script`

You will receive a command similar to:

```bash
wget --no-check-certificate -qO- "https://<your-site>/api/method/hotspot_ms...download_provisioning_script?name=<Nas_Name>&secret=<Secret>" | sh
```

This is the key step that deploys the OpenWrt side of the integration.

## 9. Run the provisioning script from SSH

This step is required and should be done from the terminal.

### Step 9.1: Connect over SSH

```bash
ssh root@192.168.1.1
```

### Step 9.2: Run the generated provisioning command

Paste the generated command exactly.

Example:

```bash
wget --no-check-certificate -qO- "https://hotspot.example.com/api/method/hotspot_ms...download_provisioning_script?name=OpenWrt-Branch-1&secret=your-secret" | sh
```

This script typically does the following:

- install required network drivers
- configure LAN/WAN interfaces
- set up OpenNDS
- configure FAS settings and whitelist ports
- install custom worker scripts
- apply firewall hardening rules
- restart relevant services

### Step 9.3: Confirm the script finished successfully

Check for service status:

```bash
/etc/init.d/opennds status
ndsctl status
logread -e opennds | tail -n 80
```

If you see errors, check the script output and your Frappe values before continuing.

## 10. Validate the OpenWrt side with LuCI and terminal

This is where the UI and shell work together.

### In LuCI

Check the following:

- `Network` -> `Interfaces`
- `Network` -> `Wireless` (if used)
- `Services` -> `openNDS` or the relevant traffic capture page
- `Firewall` -> `Traffic Rules`

### In terminal

```bash
uci show opennds
cat /etc/config/opennds
/etc/init.d/opennds status
ndsctl status
```

If something is wrong, do not proceed to hardening until the basic captive portal is alive.

## 11. OpenWrt interface configuration for custom hardware

For x86/64 custom builds or USB Ethernet adapters, the hardware may not appear as expected.

For example, if WAN is on `eth0` and LAN is on `eth1`, configure:

```bash
uci set network.lan.device='eth1'
uci commit network
/etc/init.d/network restart
```

Then validate:

```bash
ip addr
```

This is a terminal-only task and is one of the main reasons you must keep SSH access available.

## 12. OpenNDS direct FAS configuration

The repo includes documentation for direct remote FAS configuration. You may need to ensure the settings match the domain and FAS key.

Typical `/etc/config/opennds` values look like:

```uci
config opennds
	option enabled '1'
	option gatewayinterface 'br-lan'
	option gatewayname 'OpenWrt-Branch-1'
	option gatewayport '2050'
	option fasremoteip 'your-public-ip-or-domain'
	option fasremotefqdn 'your-domain.com'
	option fasport '80'
	option faspath '/hotspot/login'
	option fassecureenabled '1'
	option faskey 'YOUR_GENERATED_FAS_KEY'
```

Then reload:

```bash
/etc/init.d/opennds restart
logread -e opennds | tail -n 80
```

This is where the portal redirect flow gets established.

## 13. LuCI-only tasks vs terminal-only tasks

Here is the practical separation to avoid confusion.

### Tasks you can do safely in LuCI

- Set root password
- Enable SSH
- Configure WAN/LAN interfaces
- Adjust DHCP
- View status pages
- Check firewall rules
- Configure wireless settings
- Monitor basic logs

### Tasks that require terminal

- `wget | sh` provisioning script
- `uci set ...` interface reconfiguration
- `ndsctl status` and debugging
- `nft` firewall rules
- custom script execution
- service restarts and log checks
- hardening bundle deployment
- anti-tethering rules

### Recommended workflow

Use LuCI for the basic UI tasks and SSH for the actual integration and hardening tasks.

This preserves both access methods and reduces the chance of locking yourself out.

## 14. Hardening and anti-tethering

Once the hotspot is functioning, you can apply advanced hardening.

### 14.1 Generate the hardening script

From the Frappe `Nas Device` form, generate the hardening bundle or use the helper script provided by the app.

Then run:

```bash
chmod +x /root/openwrt_hotspot_hardening.sh
HOTSPOT_IFACE=br-lan CONN_LIMIT=150 TTL_VALUE=64 \
  /root/openwrt_hotspot_hardening.sh /usr/share/nftables.d/table-pre/99-hotspot-hardening.nft
/etc/init.d/firewall restart
```

### 14.2 Anti-tethering TTL rules

```bash
nft add rule inet fw4 trian_professional iifname "eth1" ip ttl 63 counter drop
nft add rule inet fw4 trian_professional iifname "eth1" ip6 hoplimit 63 counter drop
```

### 14.3 Proxy/VPN blocking rules

```bash
nft add rule inet fw4 trian_professional tcp dport { 1080, 3128, 7777, 8080, 8243, 10808 } counter drop
nft add rule inet fw4 trian_professional udp dport { 1080, 3128, 7777, 8080, 8243, 10808 } counter drop
```

### Important

- Do not apply these in a live production environment until the basic portal flow is already working
- Keep SSH and LuCI enabled so that you can revert or refine the rules quickly

## 15. Verify the working flow end to end

After provisioning and hardening, verify the complete path.

### Check the router side

```bash
/etc/init.d/opennds status
ndsctl status
logread -e opennds | tail -n 80
logread -e hotspot-deauth | tail -n 80
```

### Check the Frappe side

- Confirm `Nas Device` matches the router
- Confirm `FAS Key` matches the router config
- Run the session expiry/deauth task from Frappe if needed

### Trigger manual tasks

```bash
bench --site <your-site> execute hotspot_ms.tasks.close_expired_or_used_sessions
```

### Test redirect flow

From a connected client:

- Connect to the guest network
- Try to reach a blocked site
- Confirm the captive portal page appears
- Validate voucher login flow

## 16. Do not lose LuCI or terminal access: best practices

Follow this checklist:

### Before making changes

- set root password
- enable SSH
- record WAN/LAN config
- test SSH from a laptop
- confirm you can still reach LuCI

### Before enabling firewall hardening

- keep a physical console connection if possible
- make sure DHCP continues to work
- confirm WAN connectivity still works
- do not disconnect the management network

### Before rebooting

- save configuration
- note interface names
- confirm the LAN IP remains reachable
- verify the default route is present

### Best recovery references

```bash
cat /etc/config/network
cat /etc/config/firewall
cat /etc/config/opennds
uci export
```

If you lose access:

- use console/serial access
- log into a direct-connected laptop with the LAN IP
- repair via SSH or local console
- restore prior config from `/etc/config` backup

## 17. Recommended final deployment workflow

This is the sequence I recommend for a stable deployment:

1. Flash OpenWrt 25.12.4 x86/64 image
2. Log in to LuCI
3. Set root password and enable SSH
4. Verify LAN/WAN connectivity
5. Save the initial config
6. Create the router record in Frappe
7. Generate the provisioning script
8. Run the provisioning command over SSH
9. Check `openNDS` and router services
10. Validate captive portal flow
11. Apply hardening only after validation
12. Keep both LuCI and SSH active permanently

## 18. Troubleshooting

### Cannot access LuCI

- Confirm the LAN IP is correct
- Check DHCP lease on your laptop
- Verify the device is connected to LAN
- Check if the router booted correctly
- Use serial/console access if necessary

### Cannot SSH into the router

- Ensure SSH is enabled in LuCI
- Check firewall rules
- Confirm the LAN address is correct
- Try from the same LAN subnet
- Check `dropbear`/SSH service status

```bash
/etc/init.d/dropbear status
ps | grep dropbear
logread | tail -n 50
```

### openNDS not working

```bash
/etc/init.d/opennds restart
logread -e opennds | tail -n 100
ndsctl status
```

### Portal not redirecting

- check `fasremoteip` and `fasremotefqdn`
- confirm `fasport` and `faspath`
- confirm website is reachable on port 80 or 443
- verify FAS key matches Frappe

## 19. Final note

This architecture is not designed to be fully managed from LuCI alone.

You can and should use LuCI for the router's web administration, but the actual Hotspot MS/OpenNDS integration requires shell/SSH work, especially for:

- provisioning script execution
- interface and UCI configuration
- openNDS setup
- firewall hardening
- service restart and verification

If you follow the sequence above, you will keep access to both the browser UI and the terminal throughout the deployment, while still completing the full setup safely.

## 20. Quick command checklist

Use these commands as your working checklist:

```bash
ssh root@192.168.1.1
uci show network
uci show opennds
cat /etc/config/opennds
/etc/init.d/opennds status
ndsctl status
logread -e opennds | tail -n 80
ip addr
ip route
ping -c 3 8.8.8.8
```

## 21. Recommended next step

After you finish the initial base install and confirm OpenNDS is active, proceed to:

- Frappe `Nas Device` configuration
- provisioning script execution
- portal validation
- hardening bundle deployment

Then verify the actual session and voucher flow using a client device connected to the LAN.

---

This document is intended as a practical, repeatable setup guide for your environment. If you want, I can also produce a second version optimized for:

1. a printable PDF-style runbook
2. a shorter checklist format for field deployment
3. a version tailored specifically for x86/64 mini PCs
4. a version with exact commands for your actual WAN/LAN topology
