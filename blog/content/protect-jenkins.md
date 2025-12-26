++
title = "Locking Down Jenkins: Access via WireGuard VPN Only"
date = 2024-04-19
draft = false
[taxonomies]
  tags = ["Docker"]
[extra]
  toc = true
  keywords = "Docker"
++

In the previous post [Set up Jenkins and Nginx Reverse Proxy in Docker Containers](/blog/nginx-jenkins-reverse-proxy), we successfully deployed a Jenkins controller and exposed it via Nginx.

However, directly exposing a CI/CD tool like Jenkins to the public internet is a security risk. It presents a large attack surface for anyone looking to infiltrate your build pipeline.

The industry standard solution is to hide Jenkins behind a VPN, making it accessible only to authorized users on a private network. In this post, I will show you how to use [WireGuard VPN](https://www.wireguard.com/)—a modern, fast, and secure VPN protocol—to lock down your Jenkins instance.

## Requirements

- A server running Jenkins and Nginx (as set up in the previous post).
- Root/Sudo access to the server.
- A client machine (your laptop/desktop).

## Step 1: Install WireGuard

Installing WireGuard is straightforward on most modern Linux distributions. Refer to the [official install documentation](https://www.wireguard.com/install/) for OS-specific commands.

**A note on Docker:**
You can run WireGuard inside a container, but it requires privileged access or host kernel access and is more complex. Installing WireGuard on the host is simpler and more performant.

## Step 2: Configure the WireGuard Server

We will create a private network where the server is `10.66.66.1` and your client is `10.66.66.2`.

### Generate keys

WireGuard uses public-key cryptography. Generate a key pair for the server and another for the client:

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

Save the generated `privatekey` and `publickey` for each peer.

### Server configuration (create `/etc/wireguard/wg0.conf`)

```ini
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = <INSERT SERVER PRIVATE KEY HERE>

# Client Peer
[Peer]
PublicKey = <INSERT CLIENT PUBLIC KEY HERE>
AllowedIPs = 10.66.66.2/32
```

Start the interface and enable it on boot:

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

## Step 3: Configure the Client

Install the WireGuard client on your laptop/desktop and create a configuration file like this:

```ini
[Interface]
Address = 10.66.66.2/24
PrivateKey = <INSERT CLIENT PRIVATE KEY HERE>

[Peer]
PublicKey = <INSERT SERVER PUBLIC KEY HERE>
Endpoint = <YOUR_SERVER_PUBLIC_IP>:51820
AllowedIPs = 10.66.66.1/32
PersistentKeepalive = 25
```

Activate the connection in your client. You should then be able to ping `10.66.66.1`.

> Note: Setting `AllowedIPs` to the server's VPN IP implements split-tunneling for just that destination. Adjust `AllowedIPs` if you want to route more traffic through the tunnel.

## Step 4: Update DNS Records

If you control DNS for `jenkins.example.com`, you can point the hostname to the VPN IP so that clients using the VPN resolve the name to the private address. How you do this depends on your DNS setup — some teams use internal DNS or split-horizon DNS to avoid exposing private addresses to the public internet.

In a simple setup, you would change the A record for the Jenkins subdomain to `10.66.66.1` at your DNS provider. Be careful: publishing private IPs in public DNS may have unintended consequences; prefer split DNS when possible.

When a client on the VPN looks up the hostname and gets `10.66.66.1`, the WireGuard client will route traffic for that IP through the tunnel to the server.

## Step 5: Enforce IP Filtering in Nginx

DNS changes are convenient but not a hard barrier. To enforce that only VPN traffic reaches Jenkins, configure Nginx to accept requests only from the WireGuard subnet.

Edit your site's Nginx configuration (for example in `sites-available`) and add `allow`/`deny` rules:

```nginx
location / {
  # ... existing proxy settings ...

  allow 10.66.66.0/24;  # Allow anyone on the WireGuard subnet
  deny all;             # Reject everyone else
}
```

Restart the webserver container to apply the change:

```bash
docker compose restart webserver
```

## Summary

You have created a secure, private tunnel for your CI/CD operations:

- WireGuard provides a private network (`10.66.66.0/24`).
- DNS can point your Jenkins hostname to the private address for VPN clients.
- Nginx restricts access to requests originating from the WireGuard subnet.

## Hardening tips

For additional protection, block ports `80`/`443` on the server's public interface (for example with `ufw` or `iptables`) and allow them only on the WireGuard interface (`wg0`). This ensures traffic never reaches Nginx unless it originates from the VPN.