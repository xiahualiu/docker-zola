+++
title = "Ubuntu 24.04 LTS Cloud Server Quick Configuration"
date = 2024-09-28
draft = false
[taxonomies]
  tags = ["Linux"]
[extra]
  toc = true
  keywords = "Linux"
+++

This is a personal reference note for configuring a fresh cloud server instance running **Ubuntu 24.04 LTS** (Noble Numbat). It covers essential system updates, storage mounting, network hardening, and security best practices.

## System & Kernel Configuration

### Choose GA or HWE Kernel Track
Ubuntu LTS offers two main kernel tracks. Choosing the right one depends on your stability needs versus your hardware requirements.

* **GA (General Availability)**: The standard, most stable kernel (e.g., Linux 6.8). It is supported for the full lifespan of the LTS release without major version jumps. Best for: Critical production servers where stability is paramount.

* **HWE (Hardware Enablement)**: Newer kernels ported from interim Ubuntu releases. It updates roughly every 6 months. Best for: Servers needing support for the very latest hardware (e.g., newest GPUs, NVMe drives, or network cards).

#### Check your current kernel:

```bash
uname -r
```

#### To install the HWE (Newer) Kernel:

```bash
sudo apt install --install-recommends linux-generic-hwe-24.04
```

#### To install the GA (Stable) Kernel:

```bash
sudo apt install --install-recommends linux-generic
```

#### Clean Up Old Kernels

After rebooting into the new kernel, it is good practice to remove the old, unused kernel images to save disk space and keep the boot menu clean.

1. **Verify you are running the new kernel** Before removing anything, confirm you have successfully booted into the target kernel.

```bash
uname -r
```

2. **Configure GRUB default (Optional)** If the system didn't boot the new kernel automatically, check your GRUB config.

```bash
sudo vim /etc/default/grub
```

Ensure `GRUB_DEFAULT=0` (or saved if you prefer). If you changed it, update grub and reboot again.

```bash
sudo update-grub
```

3. **List installed kernel packages** Check which kernels are currently installed on your system.

```bash
apt list --installed "linux-*"
```

4. **Remove unused kernel packages** Ubuntu has a built-in safety mechanism in `apt autoremove` that keeps the current kernel and one previous version (as a fallback). To clean up everything else:

```bash
sudo apt autoremove --purge
```

## Storage Configuration (Fstab & RAID)

If you have attached block storage or need to mount a RAID array, handle this before configuring services.

### RAID Assembly (mdadm)

If your server uses software RAID, install `mdadm` and verify the array assembly.

```bash
sudo apt install mdadm
sudo mdadm --assemble --scan
```

#### Filesystem Mounting (fstab)

Always use UUIDs instead of device names (e.g., `/dev/vdb`) as device names can change between reboots.

1. Find the UUID of your disk:

```bash
lsblk -f
# Copy the UUID string (e.g., 1234-ABCD-...)
```

2. Edit the filesystem table:

```bash
sudo vim /etc/fstab
```

3. Add your entry at the bottom:

```
# Format: UUID=<uuid> <mount_point> <fstype> <options> <dump> <pass>
UUID=550e8400-e29b-41d4-a716-446655440000 /mnt/data ext4 defaults,noatime 0 2
```

4. Test the mount (to prevent boot failures):

```bash
sudo mount -a
```

#### Ext4 Mounting Flags (Performance Tuning)

For specific high-throughput scenarios (like build servers or temporary caches), you may want to trade some data safety for raw performance by relaxing the Ext4 journaling guarantees.

* `data=writeback`: Metadata is journaled, but data is not. (Fastest, but risks stale data on crash).
* `barrier=0`: Disables write barriers. (Improves write throughput, risks filesystem corruption on power loss).
* `commit=60`: Syncs metadata every 60 seconds instead of the default 5.

1. **Enable Writeback Mode on the Device** Before you can mount with `data=writeback`, you must enable the feature flag on the filesystem itself using `tune2fs`

```bash
# Replace /dev/sdX with your device
sudo tune2fs -o journal_data_writeback /dev/sdX
```

2. **Update Fstab** Add the flags to your mount options in `/etc/fstab`:

```
UUID=... /mnt/data ext4 defaults,noatime,data=writeback,barrier=0,commit=60 0 2
```

3. **(Required) For Root Filesystem** If you are applying this to the root (`/`) partition, `fstab` might be processed too late in the boot chain. You must pass these as kernel flags via GRUB.

Edit `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="... rootflags=data=writeback,barrier=0,commit=60"
```

Update GRUB and reboot:

```bash
sudo update-grub
sudo reboot
```

### Verify Filesystem Flags

After rebooting, it is crucial to verify that your performance flags were actually applied.

* **Check Active Mount Options**: Use `/proc/mounts` (which shows the kernel's view of mounts) rather than mount (which sometimes shows `/etc/mtab`):

```bash
cat /proc/mounts | grep ext4
```

* **Check Superblock Features**: To verify if the `journal_data_writeback` feature is enabled on the device permanently:

```bash
sudo tune2fs -l /dev/sdX | grep "Default mount options"
```

## User & Access Management

Avoid using the `root` user for daily tasks. Create a privileged user with `sudo` access.

```bash
# Create user and add to sudo group
sudo adduser <newuser>
sudo usermod -aG sudo <newuser>

# Switch to the new user
su - <newuser>
```

## Network Configuration

Ubuntu 24.04 relies heavily on `systemd-networkd` and Netplan.

### Netplan Setup

First, verify that `networkd` is the active backend.

```bash
sudo systemctl restart systemd-networkd.service
```

Check `/etc/netplan/` for existing configurations. If none exist, create a new one:

```bash
sudo vim /etc/netplan/default.yaml
```

YAML Configuration: *Note: YAML is strict about indentation (use 2 spaces).*

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      dhcp6: no
      addresses:
        - <server_ipv4>/24
        - <server_ipv6>/64
      routes:
        - to: default
          via: <server_gateway_ipv4>
        - to: default
          via: <server_gateway_ipv6>
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.0.0.1
          - 2606:4700:4700::1111
          - 2606:4700:4700::1001
```

> Tip: If your cloud provider supports DHCP, you can simply set dhcp4: yes and remove the addresses and routes sections.

Apply the settings:

```bash
sudo netplan try
sudo netplan apply
```

### DNS Security (DNSSEC & DNSOverTLS)

Enhance DNS privacy by enabling TLS.

```bash
sudo vim /etc/systemd/resolved.conf
```

Uncomment or add:

```toml
[Resolve]
DNSSEC=yes
DNSOverTLS=yes
```

Restart the resolver:

```bash
sudo systemctl restart systemd-resolved.service
```

## Security Hardening

### SSH Configuration

Ubuntu 22.10+ and 24.04 utilize socket-based activation for SSH. This requires a two-step configuration to change ports.

#### Add SSH Public Key

```bash
mkdir -p ~/.ssh/
vim ~/.ssh/authorized_keys # Paste your public key here
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

#### Configure `sshd_config`

Edit the main service file to restrict authentication methods.

```bash
sudo vim /etc/ssh/sshd_config
```

```
PermitRootLogin no
PasswordAuthentication no
# Port settings here are ignored by systemd socket activation
```

#### Change SSH Port (Socket Override)

To change the listening port, you must edit the socket definition, not just `sshd_config`.

```bash
sudo systemctl edit ssh.socket
```

Add the following lines **above** the commented section to clear the default listener and add your own:

```toml
[Socket]
ListenStream=
ListenStream=<new_port>
```

Reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

#### Update Client Config

On your **local machine**, update your `~/.ssh/config` to match:

```
Host myserver
    HostName <server_ip>
    User <username>
    Port <new_port>
    IdentityFile ~/.ssh/id_ed25519
```

### Nftables Firewall

We will replace `iptables` and `ufw` with the modern `nftables`.

#### Install & Clean up:

```bash
sudo apt install -y nftables
sudo apt autoremove -y iptables ufw
```

#### Configure Rules:

```bash
sudo vim /etc/nftables.conf
```

```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # Allow loopback (localhost)
        iifname lo accept

        # Allow established/related connections (crucial for responses)
        ct state vmap { established : accept, related : accept, invalid : drop }

        # Allow SSH (Ensure this matches your new port!)
        tcp dport <new_port> accept

        # ICMP & IPv6 essentials
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # DHCPv6 (required for some IPv6 cloud inits)
        udp dport 546 accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

#### Apply & Enable:

> Caution: Ensure you have console access in case you lock yourself out.

```bash
sudo systemctl restart nftables.service
sudo systemctl enable nftables.service
```

### Fail2Ban

Optional but recommended for brute-force protection.

```bash
sudo apt install -y fail2ban
```

Configure a local jail override:

```bash
sudo vim /etc/fail2ban/jail.d/defaults-debian.conf
```

Ensure the `backend` is set to systemd and `banaction` utilizes nftables:

```toml
[sshd]
enabled = true
port = <new_port>
backend = systemd
banaction = nftables-multiport
```

Restart the service:

```bash
sudo systemctl restart fail2ban.service
```