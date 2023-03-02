# Setting Up Epicbox for Epic Cash Blockchain & Wallets

The goal of this document is to provide an easily readable and reproducible workflow for setting up Epicbox at a domain of the operator's choice.

This document assumes operator is running Ubuntu 20.04.

It will detail processes for:

1. [Setup and configuration of `nginx`](#nginx) webserver/reverse proxy for use with websockets
2. [SSL certificate generation using `certbot`](#ssl)
3. [Build `epic-wallet` from source, listen for `epicbox`](#epic-wallet)
4. [Build `epicbox` from source, and launch](#epicbox)

---

<h2 id="nginx">Setup and Configuration of nginx</h2>

**Step 1: Install nginx**

```
sudo apt-get install nginx
```

Once installed, a directory located at `/etc/nginx` will be populated with files for configuration.

**Step 2: Modify nginx Configuration for Epicbox**

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

**Step 1: Install `certbot`, create symbolic link for `certbot` command**

```
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
**Step 2: Stop `nginx` service if it is running**

Certbot requires an open port at 80, with no other processes using it, for communication with the Let's Encrypt API.  Nginx automatically starts, and is running on port 80 after install, so we need to stop the service.

```
sudo systemctl stop nginx
```

**Step 3: Add DNS A Record for your chosen domain**

Through your hosting provider, Cloudflare, etc, configure your DNS settings to have a subdomain (i.e. epicbox.epic.tech), and point the A record to your server's IP address.

Steps required to do this will not be covered here.


**Step 4: Generate SSL certificate, and relevant files**

```
sudo certbot certonly --standalone -d <domain>
```
This will generate SSL certificate files and place them in `/etc/letsencrypt/<domain>/` directory.

**Step 5: Retrieve `options-ssl-nginx.conf` file from certbot GitHub (step not needed if you used `nginx` option during certificate generation).**

If you don't have `wget` installed, do so with `sudo apt-get install wget`

```
wget https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
sudo mv ./options-ssl-nginx.conf /etc/letsencrypt/
```

**Step 6: Generate Diffie-Hellman params with `openssl`**

If you don't have openssl installed, do so with `sudo apt-get install openssl`

Then, use `openssl` to generate your `ssl-dhparams.pem` file:

```
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
```
You may optionally increase `2048` value to `3072` for added security.

**Step 7: Finally, start restart your `nginx` service**

```
sudo systemctl restart nginx.service
```

You can also do this with `sudo /etc/init.d/nginx restart`.

---

<h2 id="epic-wallet">Setup and Configure Epic-Wallet for Epicbox</h2>

**Step 1: Clone and Build `epic-wallet` from source**

We will use my personal repository here, since it has a commit awaiting PR merge for a fix.

First, clone repository and branch of your choice:

```
git clone https://github.com/who-biz/epic-wallet.git --branch epicbox-address
cd epic-wallet
cargo build --release
```
Wait for `epic-wallet` to compile.

**Step 2: Run `epic-wallet` in listening mode with `epicbox` method**

Once that is finished building, we will want to run `epic-wallet` in listening mode.  The most effective way to do this is with a `tmux` session or similar tool, so we can analyze stdout logs as well as logging to file.


To start a new `tmux` session: `tmux new -s epic-wallet`


Once in the session, we can start our epic wallet:

```
cd epic-wallet/target/release
./epic-wallet listen -m epicbox -i 5
```
The `-i 5` argument specifies a 5 second refresh interval for epicbox listener.

You can then detach from this session with `Ctrl+B` then `D`.

If you need to re-attach to this session, `tmux a -t epic-wallet` will get you back in.

---

<h2 id="epicbox">Setup and Configure Epicbox</h2>

**Step 1: Install dependencies**

1. Install `rustup` from https://rustup.rs/.
  - Rust toolchains up to `1.61` are confirmed to be functional.  Anything newer has not been tested and may result in issues.
2. Install `rabbitmq`
  - This step is best completed by using the **Cloudsmith Quick Start Script**, located here: https://www.rabbitmq.com/install-debian.html#apt-cloudsmith
3. Enable STOMP Plugin for RabbitMQ:
```
rabbitmq-plugins enable rabbitmq_stomp
```

**Step 2: Clone & build your desired epicbox repo/branch**

In this step, we'll be using my personal fork of `epicbox` and will clone a branch awaiting PR merge, with a fix for domain discernment.

The official Epic Cash Epicbox repository is [located here](https://github.com/EpicCash/epicbox).

```
git clone https://github.com/who-biz/epicbox.git --branch ws-localhost-fix
cd epicbox
cargo build --release
```

**Step 3: Run `epicbox`**

This step is best performed in a `tmux` session or similar, so that logs can be more easily analyzed when printed to stdout.

To start a tmux session: `tmux new -s epicbox`

Once in session, navigate to directory containing epicbox binary, and launch it.  Be sure to replace `<domain>` with your information, omitting brackets:

```
cd epicbox/target/release
RUST_LOG=debug EPICBOX_DOMAIN=<domain> BIND_ADDRESS=0.0.0.0:3423 ./epicbox
```
