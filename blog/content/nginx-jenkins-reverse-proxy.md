+++
title = "Set up Jenkins and Nginx reverse proxy in Docker containers"
date = 2024-04-20
draft = false
[taxonomies]
  tags = ["Docker", "Jenkins", "Nginx"]
[extra]
  toc = true
  keywords = "Docker, Jenkins, Nginx, Reverse Proxy, CI/CD, Container"
+++
This post demonstrates how to set up Jenkins in a Docker container and configure an Nginx container to act as a reverse proxy for the Jenkins web interface.

## Requirements

- Nginx is Ready: You have configured your Nginx container and verified it can serve HTTP/HTTPS content. See my previous post Set up Let's Encrypt (Certbot) and Nginx in Docker Containers for details on setting up Nginx with SSL.

- Same Host: The Nginx container and the Jenkins container must run on the same machine to communicate via a local Docker network.

## Docker Compose YAML

Here is an example docker-compose.yml file that orchestrates both services:

```yaml
services:
  # Nginx Service
  webserver:
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/conf/:/etc/nginx/conf.d/:ro   # Nginx config folder
      - ./nginx/log/:/var/log/nginx:rw        # Log folder
      - ./certbot/www/:/var/www/certbot/:ro   # Certbot challenge folder
      - ./certbot/conf/:/etc/letsencrypt/:ro  # Certbot SSL certificates
    networks:
      - local-net 
    depends_on:
      - jenkins   # Ensures Jenkins starts before Nginx tries to proxy to it

  # Jenkins Service
  jenkins:
    image: jenkins/jenkins:lts
    user: root # Avoids permission issues with volumes (not recommended for high-security envs)
    volumes:
      - ./jenkins/config/:/var/jenkins_home:rw # Persist Jenkins data
    networks:
      - local-net

networks:
  local-net: 
    driver: bridge
```

## Note on Networks

While Docker Compose automatically creates a default network for all services in a file to communicate[^1], explicitly defining local-net allows for better isolation. It restricts communication so that only containers attached to local-net can talk to each other, which is useful if you add more unrelated containers to this host later.

## Nginx Host Configuration

The following configuration assumes you are setting up HTTPS (highly recommended).

If you must use HTTP only (not recommended), simply remove the SSL directives and the 301 redirection block, then move the location / block into the port 80 server block.

Save this file in your ./nginx/conf/ directory (e.g., as jenkins.conf):

```nginx
# HTTP Block - Redirect to HTTPS
server {
    listen 80;
    listen [::]:80;

    server_name jenkins.example.com; # Replace with your domain

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Block - Jenkins Proxy
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    # Enable HTTP/2 (Requires Nginx 1.25.1+)
    http2 on;

    server_name jenkins.example.com; # Replace with your domain

    # SSL Config
    ssl_certificate /etc/letsencrypt/live/jenkins.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jenkins.example.com/privkey.pem;

    access_log /var/log/nginx/jenkins.access.log;
    error_log  /var/log/nginx/jenkins.error.log;

    location / {
        # Standard Proxy Headers
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Forward to the Jenkins container name on internal port 8080
        proxy_pass http://jenkins:8080;
        
        # Increase timeout for long-running Jenkins operations
        proxy_read_timeout 90s;

        # Fix "It appears that your reverse proxy setup is broken" error in Jenkins
        # This rewrites the Location headers in responses from Jenkins
        proxy_redirect http://jenkins:8080 https://jenkins.example.com;
    }
}
```

## Restart Nginx Service

Once the docker-compose.yml is updated and the Nginx config file is created, restart the webserver to apply changes.

```bash
docker compose restart webserver
```

Open your browser and navigate to your Jenkins domain. You should see the login screen.

## Postscripts: Security Warning

For most use cases, it is NOT recommended to expose Jenkins directly to the public internet. Jenkins has a history of security vulnerabilities, and while the developers patch them swiftly, there is always a window of opportunity for malicious actors to exploit zero-day vulnerabilities[^2].

Best Practice: Host your Jenkins instance behind a VPN (Virtual Private Network). This ensures that only users authenticated to your private network can access the Jenkins interface.

In my setup, Jenkins is accessible only through a WireGuard VPN. You can read my post Make Jenkins Accessible Only through WireGuard VPN to learn how to implement this.

[^1]: Reference: Docker Compose Networking from docs.docker.com.

[^2]: A zero-day (or 0-day) is a vulnerability in a computer system that is unknown to its owners or developers, making it difficult to mitigate before an attack occurs.