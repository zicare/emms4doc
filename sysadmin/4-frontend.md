---
title: Frontend
layout: page
---

The EMMS4 frontends are Angular applications delivered as static builds.
Sysadmins are not expected to build Angular apps — the development team delivers pre-built folders.
Your task is only to deploy them with nginx and set up SSL.

## Users frontend

1. Copy the Angular build folder delivered by the development team into:

```bash
sudo mkdir -p /var/www/ux.u.example.com
sudo cp -r /path/to/build/* /var/www/ux.u.example.com/
```

2. Create a minimal nginx config at /etc/nginx/sites-available/ux.u.example.com:

```bash
server {
    listen 80;
    server_name ux.u.example.com;

    root /var/www/ux.u.example.com;
    index index.html;

    location / {
        try_files $uri /index.html;
    }
}
```

3. Enable the site and reload nginx:

```bash
sudo ln -s /etc/nginx/sites-available/ux.u.example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

4. Run certbot to obtain and install SSL:

```bash
sudo certbot --nginx -d ux.u.example.com
```

This command will:

* Request a Let’s Encrypt certificate,
* Edit the nginx config to listen on 443 ssl,
* Add the correct ssl_certificate and ssl_certificate_key lines.

## Other frontends

Repeat the same procedure for the other builds:

* Clients: /var/www/ux.c.example.com → https://ux.c.example.com
* Sponsors: /var/www/ux.s.example.com → https://ux.s.example.com
* Public: /var/www/ux.p.example.com → https://ux.p.example.com

NOTES:

* Each has its own folder under /var/www and its own nginx config.
* Run certbot --nginx -d <domain> for each.

## Accessing the frontends

End users log into EMMS using their browser:

* Recommended browser: Google Chrome (tested)
* Works on both desktop and mobile (mobile-first UI design)

URLs:

* Users → https://ux.u.example.com
* Clients → https://ux.c.example.com
* Sponsors → https://ux.s.example.com
* Public → https://ux.p.example.com