# NGINX Application Gateway Setup on Ubuntu 24.04

## Overview

This guide provides a detailed, step-by-step process for installing and configuring an Ubuntu 24.04 server to act as a secure, high-performance Layer 7 application gateway. We'll use the community edition of NGINX for load balancing and reverse proxying.

Security is enhanced by integrating ModSecurity v3 (libmodsecurity) with the OWASP Core Rule Set (CRS) to provide robust Web Application Firewall (WAF) capabilities against common threats like SQL injection and Cross-Site Scripting (XSS).

The configuration supports multiple API endpoints routed to various backend server farms or standalone instances, with session persistence. It also details how to use a password-protected SSL/TLS certificate for HTTPS without compromising security by storing the password in a plaintext configuration file.

---

## Prerequisites

* **Operating System**: A clean installation of Ubuntu 24.04 LTS.
* **Access**: `sudo` or root privileges.
* **Domain Name**: A fully qualified domain name (FQDN) pointing to your server's public IP address.
* **Backend Servers**: At least four backend application servers ready to receive traffic.

---

## Variable Definitions

Before you begin, replace the following placeholders throughout the guide with your actual values.

| Variable | Description | Example |
| :--- | :--- | :--- |
| `{FQDN}` | Your fully qualified domain name. | `api.example.com` |
| `{YOUR_EMAIL}`| Your email address for Let's Encrypt notifications. | `admin@example.com`|
| `{CERT_PASSWORD}` | The password for your SSL private key. | `YourStrongPassword!23` |
| `{BACKEND_IP_1}` | IP address of the first backend server. | `10.0.1.10` |
| `{BACKEND_PORT_1}`| Port for the first backend server. | `8081` |
| `{BACKEND_IP_2}` | IP address of the second backend server. | `10.0.1.11` |
| `{BACKEND_PORT_2}`| Port for the second backend server. | `8082` |
| `{BACKEND_IP_3}` | IP address of the third backend server. | `10.0.1.12` |
| `{BACKEND_PORT_3}`| Port for the third backend server. | `8083` |
| `{BACKEND_IP_4}` | IP address of the fourth backend server. | `10.0.1.13` |
| `{BACKEND_PORT_4}`| Port for the fourth backend server. | `8084` |

---

## Installation and Setup

### Step 1: System Preparation

First, update your system's package list and upgrade existing packages to their latest versions.

```bash
sudo apt update && sudo apt upgrade -y
````

Install essential tools needed for compilation and system management.

```bash
sudo apt install -y build-essential libpcre3-dev libssl-dev zlib1g-dev git libtool autoconf automake pkg-config
```

### Step 2: Install NGINX Community Edition

Ubuntu 24.04 includes a recent version of NGINX in its default repositories.

```bash
sudo apt install -y nginx
```

Verify the installation and get the exact version number, which is needed to compile the dynamic module for ModSecurity.

```bash
nginx -v
# Example output: nginx/1.26.0
```

### Step 3: Build and Install ModSecurity v3 from Source

The recommended way to use ModSecurity with NGINX is by compiling `libmodsecurity` and the NGINX connector module from source.

**1. Install libmodsecurity (the WAF Engine)**

```bash
# Clone the repository
git clone --depth 1 -b v3/master --single-branch [https://github.com/owasp-modsecurity/ModSecurity.git](https://github.com/owasp-modsecurity/ModSecurity.git)
cd ModSecurity

# Configure and compile
git submodule init
git submodule update
./build.sh
./configure
make
sudo make install

# Return to home directory
cd ..
```

**2. Compile the ModSecurity v3 NGINX Connector**

This connector links `libmodsecurity` with NGINX. We compile it as a dynamic module against the NGINX version you installed.

```bash
# Get the exact NGINX version string from your installation
NGINX_VERSION=$(nginx -v 2>&1 | cut -d'/' -f2)

# Download the exact same version of NGINX source code
wget [http://nginx.org/download/nginx-$](http://nginx.org/download/nginx-$){NGINX_VERSION}.tar.gz
tar -zxvf nginx-${NGINX_VERSION}.tar.gz

# Clone the ModSecurity connector source code
git clone --depth 1 [https://github.com/owasp-modsecurity/ModSecurity-nginx.git](https://github.com/owasp-modsecurity/ModSecurity-nginx.git)

# Navigate into the NGINX source directory and compile the module
cd nginx-${NGINX_VERSION}
./configure --with-compat --add-dynamic-module=../ModSecurity-nginx
make modules

# Copy the compiled module to the NGINX modules directory
sudo cp objs/ngx_http_modsecurity_module.so /usr/share/nginx/modules/

# Return to home directory
cd ..
```

### Step 4: Install OWASP Core Rule Set (CRS)

The OWASP CRS provides the rules that ModSecurity uses to detect and block attacks.

```bash
# Clone the latest CRS repository
sudo git clone [https://github.com/coreruleset/coreruleset.git](https://github.com/coreruleset/coreruleset.git) /etc/nginx/modsec/
sudo mv /etc/nginx/modsec/crs-setup.conf.example /etc/nginx/modsec/crs-setup.conf
sudo mv /etc/nginx/modsec/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example /etc/nginx/modsec/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```

### Step 5: Create a Password-Protected SSL/TLS Certificate

For production, you should use a certificate from a trusted authority like Let's Encrypt. For this guide, we'll create a self-signed certificate with a password.

**1. Create the Certificate Files**

```bash
sudo mkdir /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/{FQDN}.key.insecure \
    -out /etc/nginx/ssl/{FQDN}.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/CN={FQDN}"

# Encrypt the private key with a password
sudo openssl rsa -in /etc/nginx/ssl/{FQDN}.key.insecure -aes256 -out /etc/nginx/ssl/{FQDN}.key
# You will be prompted to enter and verify a new password. Use `{CERT_PASSWORD}`.

# Remove the insecure, unencrypted key
sudo rm /etc/nginx/ssl/{FQDN}.key.insecure
```

**2. Create the SSL Password File**

NGINX will read the password from this file at startup.

```bash
echo "{CERT_PASSWORD}" | sudo tee /etc/nginx/ssl/cert_pass.txt
```

**3. Set Secure Permissions**

Only the root user should be able to read these sensitive files.

```bash
sudo chmod 600 /etc/nginx/ssl/*
sudo chown root:root /etc/nginx/ssl/*
```

-----

## Configuration

### Step 1: Configure NGINX to Load the ModSecurity Module

Edit the main NGINX configuration file to load the dynamic module.

```bash
sudo nano /etc/nginx/nginx.conf
```

Add the following line at the top of the file, above the `http` block:

```nginx
load_module modules/ngx_http_modsecurity_module.so;
```

### Step 2: Configure ModSecurity and OWASP CRS

Create a central configuration file for ModSecurity that will be included in your NGINX site configuration.

**1. Create `modsecurity.conf`**

This file enables the rule engine and sets basic directives.

```bash
sudo nano /etc/nginx/modsec/modsecurity.conf
```

Paste the following content into the file:

```text
# Basic ModSecurity configuration
# SecRuleEngine On: Detect and block attacks
# SecRuleEngine DetectionOnly: Only log, do not block
SecRuleEngine On
SecRequestBodyAccess On
SecResponseBodyAccess On
SecResponseBodyMimeType text/plain text/html text/xml application/json
SecDataDir /tmp/
SecAuditLog /var/log/nginx/modsec_audit.log
SecAuditLogParts ABIJDEFHZ
SecAuditLogType Serial
```

**2. Create `main.conf`**

This file will tie everything together—the ModSecurity configuration and the OWASP rules.

```bash
sudo nano /etc/nginx/modsec/main.conf
```

Add the following lines:

```text
# Include the main ModSecurity config
Include /etc/nginx/modsec/modsecurity.conf

# Include the OWASP CRS setup file
Include /etc/nginx/modsec/crs-setup.conf

# Include the OWASP CRS rules
Include /etc/nginx/modsec/rules/*.conf
```

### Step 3: Configure NGINX Application Gateway

Now, create the NGINX server block (virtual host) that defines the application gateway's behavior.

```bash
sudo nano /etc/nginx/sites-available/{FQDN}.conf
```

Paste the following configuration, making sure to replace the placeholders.

```nginx
# Upstream group of servers for load balancing
upstream api_farm {
    # ip_hash ensures a client is always directed to the same server
    ip_hash;
    server {BACKEND_IP_1}:{BACKEND_PORT_1};
    server {BACKEND_IP_2}:{BACKEND_PORT_2};
    # You can add more servers to this farm
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name {FQDN};
    return 301 https://$host$request_uri;
}

# Main Application Gateway Server Block
server {
    listen 443 ssl http2;
    server_name {FQDN};

    # SSL/TLS Configuration
    ssl_certificate /etc/nginx/ssl/{FQDN}.crt;
    ssl_certificate_key /etc/nginx/ssl/{FQDN}.key;
    ssl_password_file /etc/nginx/ssl/cert_pass.txt;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers off;

    # ModSecurity WAF Configuration
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    # Proxy Headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # --- API Endpoint Routing ---

    # Location for API v1 & v2 (Load Balanced Farm)
    location /api/v1/ {
        proxy_pass http://api_farm;
    }

    location /api/v2/ {
        proxy_pass http://api_farm;
    }

    # Location for API v3 (Single Backend)
    location /api/v3/ {
        proxy_pass http://{BACKEND_IP_3}:{BACKEND_PORT_3};
    }

    # Location for API v4 (Single Backend)
    location /api/v4/ {
        proxy_pass http://{BACKEND_IP_4}:{BACKEND_PORT_4};
    }
    
    # Location for API v5 (Load Balanced Farm)
    location /api/v5/ {
        proxy_pass http://api_farm;
    }

    # Access and Error Logs
    access_log /var/log/nginx/{FQDN}.access.log;
    error_log /var/log/nginx/{FQDN}.error.log;
}
```

Enable this new site by creating a symbolic link.

```bash
sudo ln -s /etc/nginx/sites-available/{FQDN}.conf /etc/nginx/sites-enabled/

# It's good practice to remove the default site to avoid conflicts
sudo rm /etc/nginx/sites-enabled/default
```

-----

## Security Considerations

### Web Application Firewall (WAF)

ModSecurity is now configured with the OWASP Core Rule Set. By default (`SecRuleEngine On`), it will block suspicious requests and log them to `/var/log/nginx/modsec_audit.log`. For initial deployment, you might set `SecRuleEngine` to `DetectionOnly` in `/etc/nginx/modsec/modsecurity.conf` to monitor for false positives before enabling blocking.

### AppArmor

Ubuntu 24.04 uses AppArmor to confine applications. The default NGINX profile may not allow reading the SSL password file or writing to the ModSecurity log.

**1. Check AppArmor Status**

```bash
sudo aa-status
```

**2. Update the NGINX AppArmor Profile**

If you encounter permission errors, you may need to update the NGINX profile.

```bash
sudo nano /etc/apparmor.d/usr.sbin.nginx
```

Add the following rules inside the `/usr/sbin/nginx { ... }` block to grant the necessary permissions:

```
  # Allow reading the SSL password file
  /etc/nginx/ssl/cert_pass.txt r,

  # Allow writing to the ModSecurity audit log
  /var/log/nginx/modsec_audit.log w,
```

After saving the file, reload the AppArmor profile for NGINX.

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```

As a temporary debugging step, you can put the profile in "complain" mode, which logs violations without enforcing them: `sudo aa-complain /usr/sbin/nginx`.

### Other Security Tools

  * **SafeLine WAF / lua-resty-t1k**: These are powerful, modern WAFs that require **OpenResty** (a superset of NGINX with LuaJIT integration). While highly effective, they are beyond the scope of this NGINX Community guide. Integrating them would involve replacing NGINX with OpenResty.
  * **AI-Based Tools**: Most AI-powered threat detection tools are commercial products. True open-source, community-driven AI security projects for NGINX are not yet mature or widely adopted. Sticking with the battle-tested ModSecurity/OWASP CRS combination provides an excellent security baseline.

-----

## Finalizing the Setup

### Test Configuration and Start Services

Before restarting NGINX, always test its configuration for syntax errors.

```bash
sudo nginx -t
```

If the test is successful, restart the NGINX service to apply all changes.

```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```

Check its status to ensure it started without issues.

```bash
sudo systemctl status nginx
```

-----

## Testing and Validation

### Testing API Endpoints

Use `curl` to test each API endpoint. The `-k` flag is used to ignore certificate warnings for our self-signed cert.

```bash
# Test a load-balanced endpoint
curl -k https://{FQDN}/api/v1/some/path

# Test a single-backend endpoint
curl -k https://{FQDN}/api/v3/some/path
```

You can check your backend server logs to verify that requests are being received and that load balancing with `ip_hash` is working as expected.

### Testing the WAF

Simulate a simple XSS attack. The WAF should block it and return a **403 Forbidden** error.

```bash
curl -k "https://{FQDN}/?param=<script>alert('xss')</script>"
```

Check the ModSecurity audit log to see the details of the blocked request.

```bash
sudo tail -f /var/log/nginx/modsec_audit.log
```

You should see an entry detailing the blocked attempt, the rule that triggered it (e.g., rule ID 941100 from OWASP CRS), and the reason.

-----

## Maintenance

### Service Management

  * **Restart NGINX**: `sudo systemctl restart nginx`
  * **Reload NGINX (graceful, no downtime)**: `sudo systemctl reload nginx`
  * **Stop NGINX**: `sudo systemctl stop nginx`
  * **Check Status**: `sudo systemctl status nginx`

### Log Management

  * **NGINX Access Log**: `/var/log/nginx/{FQDN}.access.log`
  * **NGINX Error Log**: `/var/log/nginx/{FQDN}.error.log`
  * **ModSecurity Audit Log**: `/var/log/nginx/modsec_audit.log`

Remember to set up log rotation (e.g., using `logrotate`) to prevent logs from consuming all your disk space.

### Updating OWASP CRS

To keep your WAF effective, periodically update the rule set.

```bash
cd /etc/nginx/modsec/
sudo git pull
sudo systemctl reload nginx
```

-----

## References

  * **NGINX Documentation**: [https://nginx.org/en/docs/](https://nginx.org/en/docs/)
  * **ModSecurity v3 GitHub**: [https://github.com/owasp-modsecurity/ModSecurity](https://github.com/owasp-modsecurity/ModSecurity)
  * **OWASP Core Rule Set**: [https://github.com/coreruleset/coreruleset](https://github.com/coreruleset/coreruleset)
  * **Ubuntu 24.04 Server Guide**: [https://help.ubuntu.com/lts/serverguide/index.html](https://help.ubuntu.com/lts/serverguide/index.html)

<!-- end list -->

```
```
