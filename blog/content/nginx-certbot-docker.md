+++
title = "Set up Let's Encrypt (Certbot) and Nginx in Docker containers"
date = 2024-04-18
draft = false
[taxonomies]
  tags = ["Docker", "Nginx", "DevOps"]
[extra]
  toc = true
  keywords = "Docker, Nginx, Certbot, Let's Encrypt, SSL, HTTPS"
+++
This post demonstrates how to obtain Let's Encrypt SSL certificates for your self-hosted website using Nginx and Docker containers.

## Requirements

- **SSH Access:** You have access to your server's command line.
- **Domain Name:** You have at least one active domain name, and the DNS records are configured correctly.

## DNS Configuration

If you bought a domain like example.com, you must ensure your DNS provider has A records (IPv4) or AAAA records (IPv6) pointing to your server's public IP address.

You should set up records for every subdomain you intend to use, such as:

- blog.example.com
- www.example.com
- jenkins.example.com

> Note for Cloudflare Users: If you manage your DNS via Cloudflare, you must temporarily disable the DNS Proxy (the orange cloud icon). Ensure the Proxy status column shows DNS only - reserved IP.

This is required because we are using the webroot verification method (HTTP-01 challenge), which requires Let's Encrypt to connect directly to your server IP, bypassing Cloudflare's proxy network.

## Writing Docker Compose

Create a docker-compose.yml file. This defines the interaction between the Nginx web server and the Certbot client.

```yaml
services:
  # Nginx Service
  webserver:
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/conf/:/etc/nginx/conf.d/:ro   # Nginx configuration
      - ./nginx/log/:/var/log/nginx:rw        # Logs
      - ./certbot/www/:/var/www/certbot/:ro   # Certbot challenge folder (Read-only for Nginx)
      - ./certbot/conf/:/etc/letsencrypt/:ro  # Certificates (Read-only for Nginx)
  
  # Certbot Service
  certbot:
    image: certbot/certbot:latest
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw   # Certbot challenge folder (Read-write)
      - ./certbot/conf/:/etc/letsencrypt/:rw  # Certificates (Read-write)
    depends_on:
      - webserver # Certbot runs after Nginx is up
```

## Testing Nginx

First, verify the Nginx container works.

```bash
docker compose up -d webserver
```

Note: You cannot access the default Nginx welcome page yet. We mounted ./nginx/conf/ to the container, overriding the default configuration. Since your local folder is likely empty, Nginx has no sites configured.

Create a simple configuration file at ./nginx/conf/app.conf:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;

    # Required for Let's Encrypt verification
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Test connectivity (Return 404 for now)
    location / {
        return 404;
    }
}
```

(Replace example.com with your actual domain)

Restart the webserver:

```bash
docker compose restart webserver
```

Navigate to http://example.com in your browser. If you see a 404 Not Found nginx error page, congratulations! Your server is accessible.

## Troubleshooting

If you do not see the 404 page, check your Firewall and DNS settings.

1. Firewall Issues
Ensure ports 80 (HTTP) and 443 (HTTPS) are open.

Cloud Providers (AWS/GCP/Azure): Check the "Security Group" or "VPC Firewall" settings in your cloud console.

Linux Firewall (UFW):

```bash
sudo ufw status
# If active and 80/443 are missing:
sudo ufw allow 80,443/tcp
```

Linux Firewall (iptables):

```bash
# Check current rules
sudo iptables -L INPUT -n
# If traffic is dropped, add accept rules:
sudo iptables -I INPUT 1 -m state --state NEW -m multiport -p tcp --dports 80,443 -j ACCEPT
```

2. DNS Issues
Use dig (Linux/Mac) or Resolve-DnsName (Windows) to verify your domain points to the correct IP.

```bash
dig example.com +short
# Or on Windows PowerShell:
Resolve-DnsName example.com
```

If the IP is incorrect, update your DNS records. Note that propagation can take up to 24 hours (though it is usually much faster).

## Run Certbot
Once HTTP access is confirmed, we can request certificates.

1. The Dry Run
Run a test first to ensure everything is configured correctly without hitting Let's Encrypt rate limits.

```bash
docker compose run --rm certbot certonly --webroot --webroot-path /var/www/certbot/ --dry-run -d example.com -d www.example.com
```

Follow the prompts (enter email, accept TOS). If the output says "The dry run was successful", proceed to the next step.

2. The Real Request
Remove the --dry-run flag to get your actual certificates:

```bash
docker compose run --rm certbot certonly --webroot --webroot-path /var/www/certbot/ -d example.com -d www.example.com
```

Your certificates will be saved in ./certbot/conf/live/example.com/.

Permissions Note: The certificates are created by the container (often as root). You may need to change ownership to your local user to manage the files locally (though Nginx inside the container will read them fine as root).

```bash
sudo chown -R $USER ./certbot
```

## Renewing Certificates
Let's Encrypt certificates expire every 90 days. To renew them, simply run:

```bash
docker compose run --rm certbot renew
```

## Common Questions
Why run certbot with certonly instead of the --nginx plugin?

The --nginx plugin attempts to modify Nginx configuration files automatically. Since Nginx is running in a separate container, Certbot cannot see or restart the Nginx process easily. Using certonly with webroot is the cleanest "Docker-way" to separate concerns: Certbot handles files, Nginx handles serving.

## Update Nginx HTTPS Configuration
Now that you have the certificates, update ./nginx/conf/app.conf to enable HTTPS.

```nginx
# 1. HTTP Redirect Block
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;

    # Needed for Let's Encrypt renewal
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect all other traffic to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# 2. HTTPS Block (Static Site Example)
server {
    listen 443 default_server ssl;
    listen [::]:443 default_server ssl;
    
    # Enable HTTP/2 (Requires Nginx version 1.25.1+)
    http2 on;

    server_name example.com;

    root /var/www/public/; 
    index index.html;

    # SSL Config
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        try_files $uri $uri/ =404;
    }
}

# 3. HTTPS Block (Reverse Proxy Example)
# Use this if you are proxying to another app (like Node.js, Python, or Jenkins)
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        proxy_pass http://app-container:8080; # Replace with your app service name
        
        # Standard Proxy Headers
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Apply the changes:

```bash
docker compose restart webserver
```

## Security & Postscripts
Improving SSL Security
You can further harden your SSL configuration by disabling weak ciphers or enabling HSTS.

Test your configuration at Qualys SSL Labs or ImmuniWeb.

Use the Mozilla SSL Configuration Generator to generate industry-standard Nginx configs.

## Cloudflare Users
Now that your SSL is working:

You can re-enable the Cloudflare DNS Proxy (Orange cloud).

Crucial: In Cloudflare dashboard, go to SSL/TLS -> Overview and set the encryption mode to Full (Strict).

If you leave it on "Flexible", you will get infinite redirect loops because Cloudflare talks to your server via HTTP, but your server redirects back to HTTPS.

Future Renewals: When it is time to renew certificates using the webroot method, you generally do not need to disable Cloudflare proxy again, provided your HTTP-to-HTTPS redirect allows the .well-known/acme-challenge path to pass through.