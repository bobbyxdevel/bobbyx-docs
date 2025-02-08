# VPS Setup Guide

## Table of Contents
1. [Initial Setup: Create a Non‑Root User](#initial-setup-create-a-non-root-user)
2. [Install Docker on Debian](#install-docker-on-debian)
3. [Install Portainer for Docker Management](#install-portainer-for-docker-management)
4. [Deploy Nextcloud via Portainer](#deploy-nextcloud-via-portainer)
5. [Configure Your Domain and Reverse Proxy](#configure-your-domain-and-reverse-proxy)
6. [Conclusion](#conclusion)
7. [References](#references)

---

## Initial Setup: Create a Non‑Root User

A fresh Debian installation may only have the root account enabled. For better security and daily use, create a non‑root user with sudo privileges.

### Steps:
1. **Log in as root:**
   - At the console or via SSH, log in with the root credentials provided during installation.

2. **Create a new user (e.g., `admin`):**
   ```sh
   adduser admin
   ```
   - Follow the prompts to set a password and fill in any optional details.

3. **Grant the new user sudo privileges:**
   ```sh
   usermod -aG sudo admin
   ```

4. **Log out and log back in as the new user:**
   - You can now perform administrative tasks using `sudo`.

---

## Install Docker on Debian

(Based on best practices from guides such as Shapehost’s tutorial and HowtoForge)

### Steps:
1. **Update the system and install prerequisites:**
   ```sh
   sudo apt update
   sudo apt install apt-transport-https ca-certificates curl gnupg2 software-properties-common -y
   ```

2. **Add Docker’s official GPG key and repository:**
   ```sh
   curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

3. **Install Docker CE:**
   ```sh
   sudo apt update
   sudo apt install docker-ce -y
   ```

4. **Verify Docker is installed:**
   ```sh
   docker --version
   ```
   - You should see output like `Docker version 20.10.xx, build xxxxxxx`.

---

## Install Portainer for Docker Management

Portainer provides a web-based interface to easily manage your Docker containers.

### Steps:
1. **Create a Docker volume for Portainer’s data:**
   ```sh
   sudo docker volume create portainer_data
   ```

2. **Deploy the Portainer container:**
   ```sh
   sudo docker run -d \
     -p 8000:8000 \
     -p 9000:9000 \
     --name=portainer \
     --restart=always \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v portainer_data:/data \
     portainer/portainer-ce:latest
   ```

3. **Access Portainer’s Web UI:**
   - Open your browser and go to `http://<your-vps-public-ip>:9000`
   - Create an admin account when prompted and select “Local” to manage your Docker host.

---

## Deploy Nextcloud via Portainer

Using Portainer’s “Stacks” feature you can deploy a multi‑container Docker Compose file. In this example, Nextcloud is deployed with a MariaDB backend.

### Example Docker Compose File

Create a stack (either via Portainer’s UI or by saving this YAML file locally) similar to the following. Adjust host paths and environment variables as needed.

```yaml
version: "3.8"

services:
  nextcloud:
    image: linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000            # Ensure this matches your non-root user (e.g., admin)
      - PGID=1000
      - TZ=Europe/Berlin      # Adjust timezone as required
      # Additional Nextcloud settings (upload limits, memory, etc.) can be added here
    volumes:
      - /srv/nextcloud/config:/config    # Persistent config directory on the host
      - /srv/nextcloud/data:/data          # Persistent data directory on the host
    ports:
      - "8080:443"           # Map container HTTPS port to host port 8080 (for initial setup)
    restart: unless-stopped
    depends_on:
      - nextcloud-db

  nextcloud-db:
    image: linuxserver/mariadb:latest
    container_name: nextcloud-db
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Berlin
      - MYSQL_ROOT_PASSWORD=YourRootPassword
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=YourDBPassword
    volumes:
      - /srv/nextcloud/db:/config
    restart: unless-stopped

networks:
  default:
    driver: bridge
```

### Deployment Steps:
1. In Portainer, navigate to Stacks and click Add stack.
2. Paste the YAML above (or upload your file), give the stack a name (e.g., “Nextcloud”), and click Deploy the stack.
3. Once deployed, access Nextcloud by navigating to `http://<your-vps-public-ip>:8080`
4. Complete the web installer by entering the database details you specified.

For more examples and custom templates, see community templates such as the GitHub repository by tv0ll.

---

## Configure Your Domain and Reverse Proxy

To make Nextcloud available at `nextcloud.bobbyxdev.xyz` and secure it with HTTPS, follow these steps:

### DNS Setup

1. **Log in to your DNS provider’s control panel.**
2. **Create an A record for the subdomain:**
   - **Name/Host:** nextcloud
   - **Type:** A
   - **Value:** Your VPS public IP

This points `nextcloud.bobbyxdev.xyz` to your VPS.

### (Optional) Reverse Proxy and SSL with SWAG

For production use, it’s recommended to front Nextcloud with a reverse proxy that handles SSL:

1. **Deploy a reverse proxy container such as SWAG (Secure Web Application Gateway by linuxserver.io) or Nginx Proxy Manager.** For example, SWAG can be deployed like this:
   ```sh
   sudo docker run -d \
     --name=swag \
     --cap-add=NET_ADMIN \
     -e PUID=1000 \
     -e PGID=1000 \
     -e TZ=Europe/Berlin \
     -e URL=bobbyxdev.xyz \
     -e SUBDOMAINS=nextcloud \
     -e VALIDATION=http \
     -p 80:80 \
     -p 443:443 \
     -v /path/to/swag/config:/config \
     --restart unless-stopped \
     linuxserver/swag
   ```

2. **Configure SWAG:**
   - Set up a reverse proxy rule so that requests to `nextcloud.bobbyxdev.xyz` are forwarded to your Nextcloud container (e.g., to `localhost:8080`). Consult the SWAG documentation for instructions on customizing your setup.

3. **Update Nextcloud’s trusted domains:**
   - Edit the Nextcloud configuration file (found in your mounted `/srv/nextcloud/config` directory) to include your subdomain:
     ```php
     'trusted_domains' =>
       array (
         0 => 'localhost',
         1 => 'nextcloud.bobbyxdev.xyz',
       ),
     ```

This ensures Nextcloud accepts connections from your domain.

---

## Conclusion

This guide has taken you from a fresh Debian installation (starting with only a root account) through creating a new administrative user, installing Docker and Portainer, deploying a Nextcloud stack, configuring your domain, and setting up a reverse proxy with SSL.

By following these steps, you now have a robust self-hosted Nextcloud instance managed via Portainer on your Debian VPS. For further customization or troubleshooting, consult the official documentation.

---

## References

- Portainer Installation Guides – Shapehost and HowtoForge
- Nextcloud Docker deployments and templates – GitHub: tv0ll/portainer-nextcloud and community discussions (Nextcloud AIO thread)