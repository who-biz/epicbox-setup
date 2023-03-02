# Setting Up Epicbox for Epic Cash Blockchain & Wallets

The goal of this document is to provide an easily readable and reproducible workflow for setting up Epicbox at a domain of the operator's choice.

This document assumes operator is running Ubuntu 20.04.

It will detail processes for:

1. Setup and configuration of an `nginx` webserver/reverse proxy for use with websockets
2. SSL certificate generation using `certbot`
3. Building from source, and running, an instance of `[epicbox](https://github.com/EpicCash/epicbox)`.

---

<h2 id="nginx">Setup and Configuration of nginx</h2>

**Step 1:** Install nginx

```
sudo apt-get install nginx
```

Once installed, a directory located at `/etc/nginx` will be populated with files for configuration.

**Step 2:** Modify nginx Configuration for Epicbox

1. Open file located at `/etc/nginx/sites-enabled/default`:

```
sudo nano /etc/nginx/sites-enabled/default
```
2. Add the following server configuration to this file. Substitute your domain, anywhere you see `<domain>`, omitting brackets.

```
server {

        server_name <domain>;

        root /var/www/html/epicbox/;
        index index.html index.htm;

        location / {
           proxy_set_header        Host $host;
           proxy_set_header        X-Real-IP $remote_addr;
           proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header        X-Forwarded-Proto $scheme;


            # set this to the BIND_ADDRESS=0.0.0.0:3423
            proxy_pass http://0.0.0.0:3423;
            proxy_read_timeout  90;

            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        listen 80;
        listen 443 ssl default_server;
        ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
```

Next, we will generate our SSL certificates using `certbot` by letsencrypt.  If you attempt to start your `nginx` server before these files are generated and in their proper locations, `nginx` will fail to start.

---

<h2 id="ssl">Generate and Configure SSL</h2>

**Step 1:** Install `certbot`, create symbolic link for `certbot` command

```
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
**Step 2:** Stop `nginx` service if it is running

Certbot requires an open port at 80, with no other processes using it, for communication with the Let's Encrypt API.  Nginx automatically starts, and is running on port 80 after install, so we need to stop the service.

```
sudo systemctl stop nginx
```

**Step 3:** Add DNS A Record for your chosen domain

Through your hosting provider, Cloudflare, etc, configure your DNS settings to have a subdomain (i.e. epicbox.epic.tech), and point the A record to your server's IP address.

Steps required to do this will not be covered here.


**Step 4:** Generate SSL certificate, and relevant files

```
sudo certbot certonly --standalone -d <domain>
```
This will generate SSL certificate files and place them in `/etc/letsencrypt/<domain>/` directory.

**Step 5:** Retrieve `options-ssl-nginx.conf` file from certbot GitHub (step not needed if you used `nginx` option during certificate generation).

If you don't have `wget` installed, do so with `sudo apt-get install wget`

```
wget https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
sudo mv ./options-ssl-nginx.conf /etc/letsencrypt/
```

**Step 6:** Generate Diffie-Hellman params with `openssl`

If you don't have openssl installed, do so with `sudo apt-get install openssl`

Then, use `openssl` to generate your `ssl-dhparams.pem` file:

```
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
```
You may optionally increase `2048` value to `3072` for added security.

**Step 7:** Finally, start restart your `nginx` service

```
sudo systemctl restart nginx.service
```

You can also do this with `sudo /etc/init.d/nginx restart`.

---

<h2 id="epicbox">Setup and Configure Epicbox</h2>

