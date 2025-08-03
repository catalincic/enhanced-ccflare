# Configuring a Reverse Proxy with Nginx

Using a reverse proxy in front of `ccflare` is highly recommended for production or any public-facing deployment. It provides a critical layer of security and simplifies handling HTTPS/SSL certificates.

This guide will walk you through setting up Nginx as a reverse proxy for `ccflare`.

## Why Use a Reverse Proxy?

-   **SSL/TLS Termination**: Secure your traffic with HTTPS. Nginx will handle the SSL handshake, and you can run `ccflare` internally over plain HTTP.
-   **Security**: Keep `ccflare` from being directly exposed to the internet. You can run `ccflare` on `localhost` and let Nginx handle all public traffic.
-   **Load Balancing**: While `ccflare` already load balances across Claude accounts, Nginx can load balance across multiple `ccflare` instances for high availability.
-   **Centralized Logging & Rate Limiting**: Implement global logging and rate limiting rules in one place.

## Prerequisites

1.  **A cloud server** with `ccflare` installed.
2.  **Nginx installed** on your server.
3.  **A domain name** (e.g., `claude.your-domain.com`) pointing to your server's public IP address.
4.  **Certbot installed** for easy SSL certificate management with Let's Encrypt.

## Step 1: Configure `ccflare`

First, ensure `ccflare` is running on your server. For a reverse proxy setup, it's best to have it listen only on `localhost`.

You can start it with the default settings:

```bash
# Start ccflare on localhost:8080
bun run ccflare
```

No need to set `HOSTNAME=0.0.0.0` as Nginx will be handling public traffic.

## Step 2: Configure Nginx

Next, create a new Nginx configuration file for your `ccflare` proxy.

1.  Create a new file in `/etc/nginx/sites-available/`:

    ```bash
    sudo nano /etc/nginx/sites-available/ccflare
    ```

2.  Add the following configuration. This sets up a basic HTTP reverse proxy. We'll add SSL in the next step.

    ```nginx
    server {
        listen 80;
        server_name claude.your-domain.com;

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Support for streaming responses
            proxy_buffering off;
            proxy_cache off;
            proxy_http_version 1.1;
            proxy_set_header Connection '';
        }
    }
    ```

3.  Enable this configuration by creating a symbolic link to `sites-enabled`:

    ```bash
    sudo ln -s /etc/nginx/sites-available/ccflare /etc/nginx/sites-enabled/
    ```

4.  Test the Nginx configuration:

    ```bash
    sudo nginx -t
    ```

5.  If the test is successful, restart Nginx:

    ```bash
    sudo systemctl restart nginx
    ```

At this point, you should be able to access `ccflare` at `http://claude.your-domain.com`.

## Step 3: Secure with SSL using Certbot

1.  Run Certbot to automatically obtain and install an SSL certificate for your domain:

    ```bash
    sudo certbot --nginx -d claude.your-domain.com
    ```

    Certbot will update your Nginx configuration to handle HTTPS and set up automatic certificate renewal.

2.  After Certbot finishes, your Nginx configuration will look something like this:

    ```nginx
    server {
        server_name claude.your-domain.com;

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Support for streaming responses
            proxy_buffering off;
            proxy_cache off;
            proxy_http_version 1.1;
            proxy_set_header Connection '';
        }

        listen 443 ssl; # managed by Certbot
        ssl_certificate /etc/letsencrypt/live/claude.your-domain.com/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/claude.your-domain.com/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    }

    server {
        if ($host = claude.your-domain.com) {
            return 301 https://$host$request_uri;
        } # managed by Certbot

        listen 80;
        server_name claude.your-domain.com;
        return 404; # managed by Certbot
    }
    ```

## Step 4: Configure Your Clients

Now, you can securely connect to your `ccflare` instance from anywhere. Update your client configuration to use the new HTTPS endpoint:

```bash
export ANTHROPIC_BASE_URL="https://claude.your-domain.com"
```

All your Claude API traffic will now be securely routed through your Nginx reverse proxy to your `ccflare` instance.
