# Pterodactyl Panel & Wings Setup with Cloudflare Zero Trust

This guide provides instructions for securely exposing your Pterodactyl Panel and Wings daemon behind a **Cloudflare Zero Trust Tunnel** using **NGINX** and self-signed SSL certificates for local security.

This guide assumes you have **Pterodactyl Wings & Panel** already installed with **NGINX**, as well as the **Cloudflared** daemon (**Cloudflare Zero Trust**). For help with setting up the Cloudflare Tunnel, I recommend reading this external resource: [How to Setup a Cloudflare Tunnel and Expose Your Local Service or Application](https://medium.com/design-bootcamp/how-to-setup-a-cloudflare-tunnel-and-expose-your-local-service-or-application-497f9cead2d3).

---

## 1. Panel Instructions

### 1.1. Generate SSL Certificates

First, you'll need to generate self-signed SSL certificates for NGINX to use on the local network.

1.  **Create a folder for the certificates:**

    ```bash
    mkdir -p /etc/certs
    ```

2.  **Navigate into the folder:**

    ```bash
    cd /etc/certs
    ```

3.  **Generate the SSL certificates:**

    ```bash
    openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/C=NA/ST=NA/L=NA/O=NA/CN=Generic SSL Certificate" -keyout privkey.pem -out fullchain.pem
    ```

### 1.2. Configure NGINX for the Panel

1.  **Go into the NGINX configuration directory:**

    ```bash
    cd /etc/nginx/sites-enabled
    ```

2.  Create or replace the content of `pterodactyl.conf` with the configuration below. **Important**: You must replace the placeholders in the comments (`<ip>`, `<domain>`) with your specific local IP address and the correct certificate paths.

    ```nginx
    server {
        # Replace the example <ip> with your local IP address
        listen 80;
        server_name <ip>;
        return 301 https://$server_name$request_uri;
    }

    server {
        # Replace the example <ip> with your local IP address
        listen 443 ssl http2;
        server_name <ip>;

        root /var/www/pterodactyl/public;
        index index.php;

        access_log /var/log/nginx/pterodactyl.app-access.log;
        error_log  /var/log/nginx/pterodactyl.app-error.log error;

        client_max_body_size 100m;
        client_body_timeout 120s;

        sendfile off;

        # SSL Configuration - Replace ssl_certificate with '/etc/certs/fullchain.pem' and ssl_certificate_key with '/etc/certs/privkey.pem'
        ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
        ssl_session_cache shared:SSL:10m;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
        ssl_prefer_server_ciphers on;

        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header Content-Security-Policy "frame-ancestors 'self'";
        add_header X-Frame-Options DENY;
        add_header Referrer-Policy same-origin;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/run/php/php8.3-fpm.sock;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param HTTP_PROXY "";
            fastcgi_intercept_errors off;
            fastcgi_buffer_size 16k;
            fastcgi_buffers 4 16k;
            fastcgi_connect_timeout 300;
            fastcgi_send_timeout 300;
            fastcgi_read_timeout 300;
            include /etc/nginx/fastcgi_params;
        }

        location ~ /\.ht {
            deny all;
        }
    }
    ```

3.  **Restart NGINX** to apply the changes:

    ```bash
    systemctl restart nginx
    ```

> After restarting NGINX, you should be able to access your panel at the local IP. If it shows an **"invalid SSL certificates"** warning, the local setup is correct.

### 1.3. Configure Cloudflare Zero Trust Tunnel (Panel Route)

1.  In your browser, navigate to **Cloudflare Zero Trust** > **Networks** > **Tunnels**.
2.  **Edit your tunnel** > **Published application routes**.
3.  **Add a published application route** for the panel.
4.  Reference the image for the correct settings:

![Image](https://temp)

5.  Tick **'No TLS Verify'** under **'Additional application settings'**.

![Image](https://temp)

6.  Save, wait a few minutes, and you should be able to access your Pterodactyl Panel via your chosen (sub)domain! 🎉

---

## 2. Wings Instructions

### 2.1. Generate SSL Certificates

If you already completed **Section 1.1**, skip this step. Otherwise, run the following commands:

```bash
mkdir -p /etc/certs
cd /etc/certs
openssl req -new -newkey rsa:4096 -days 3650 -nodes -x509 -subj "/C=NA/ST=NA/L=NA/O=NA/CN=Generic SSL Certificate" -keyout privkey.pem -out fullchain.pem
```

### 2.2. Configure NGINX for Wings

Wings listens on port `8080` by default. This NGINX config will proxy requests on port `443` to the Wings daemon.

1.  **Go into the NGINX configuration directory:**

    ```bash
    cd /etc/nginx/sites-enabled
    ```

2.  **Create or edit the Wings configuration file:**

    ```bash
    nano wings.conf
    ```

3.  **Copy the content** into the file. **Important**: Replace `<ip>` with your local IP address.

    ```nginx
    server {
        listen 443 ssl http2;
        server_name <ip>;  # Replace with your local IP

        ssl_certificate /etc/certs/fullchain.pem;
        ssl_certificate_key /etc/certs/privkey.pem;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-Content-Type-Options "nosniff";

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_buffering off;
        }
    }
    ```

4.  **Restart NGINX** to apply the Wings configuration:

    ```bash
    systemctl restart nginx
    ```

### 2.3. Configure Cloudflare Zero Trust Tunnel (Wings Route)

1.  In your browser, navigate to **Cloudflare Zero Trust** > **Networks** > **Tunnels**.
2.  **Edit your tunnel** > **Published application routes**.
3.  **Add a published application route** for Wings.
4.  Reference the image for the correct settings:

![Image](https://temp)

5.  Tick **'No TLS Verify'** under **'Additional application settings'**.

![Image](https://temp)

### 2.4. Final Pterodactyl Panel Configuration

1.  On your Pterodactyl panel, go to **'Admin'** > **Nodes**.
2.  **Create a new node** (or edit your existing one).
    * Set the **FQDN** as the (sub)domain you set up in Cloudflare for Wings.
    * Ensure **'Not Behind Proxy'** is ticked under the 'Behind Proxy' section.

3.  Go to the node's **'Configuration'** tab, copy the configuration file, and paste it into the correct directory on your Wings server:

    ```bash
    nano /etc/pterodactyl/config.yml
    ```
    **Critical**: In the `config.yml` file, ensure the directories for the SSL credentials are set correctly:
    * `cert`: `'/etc/certs/fullchain.pem'`
    * `key`: `'/etc/certs/privkey.pem'`

4.  **Run the debug tool for Wings** to check for configuration errors:

    ```bash
    wings --debug
    ```

5.  If no errors are shown, return to the node settings on the Panel and **change the daemon port to `443`**. Save the changes. You should now see a **green heart** next to your node! ✅
