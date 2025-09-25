---
title: Config
layout: page
---

The EMMS4 backend is configured via a single JSON file:

/home/emms/emms4/config/production.json

This file is small but critical. It defines runtime behavior for security, database, SMTP, scheduling, and image handling.

## Main sections

### General

```json
"env": "production",
"pepper": "secret-random-string",
"hmac_key": "secret-hmac-key",
"jwt_duration": "1h",
"tz": "America/Mexico_City"
```

**env:** Always leave as "production".

**pepper, hmac_key:** Secret values. Change on install.

**jwt_duration:** Token expiration. Default 1 hour.

**tz:** System timezone. Must match your server setup.

### Server

```json
"server": {
  "ip": "127.0.0.1",
  "domain": "localhost",
  "port": 8443,
  "ssl": {
    "cert": "/etc/letsencrypt/live/api.example.com/fullchain.pem",
    "key": "/etc/letsencrypt/live/api.example.com/privkey.pem"
  }
}
```

#### Deployment modes

1. Real domain with SSL (direct HTTPS)
    * Example: "domain": "api.example.com".
    * EMMS4 will load the SSL cert/key from the configured paths.
    * If the files are missing or invalid, EMMS4 will refuse to start.
    * Nginx (or another proxy) is optional in this setup.
    * **Use case:** standalone HTTPS API without a reverse proxy.

2. Localhost + reverse proxy (default and recommended)
    * "domain": "localhost", "ip": "127.0.0.1", "port": 8443.
    * EMMS4 will not load SSL certs.
    * Nginx (or another proxy) handles SSL termination and forwards requests to EMMS4.
    * **Use case:** production deployment, secure and standard.

3. Localhost + direct non-SSL (testing/closed networks)
    * "domain": "localhost", "ip": "123.123.123.123", "port": 80.
    * EMMS4 will not load SSL certs.
    * Clients connect directly to the API without encryption.
    * **Use case:** quick tests, development environments, or trusted internal networks.
    * **Not recommended for production — no TLS protection.**

### Database

```json
"db": {
  "host": "localhost",
  "port": 3306,
  "user": "emms",
  "password": "CHANGE-ME",
  "name": "emms4",
  "max_open_conns": 5
}
```

* Use the emms user you created in DB setup.
* Replace the password.
* Keep max_open_conns low unless tuned for heavy load.

### SMTP

```json
"smtp": {
  "host": "smtp.example.com",
  "port": 465,
  "user": "me@example.com",
  "password": "CHANGE-ME-TOO",
  "retries": 3,
  "retry_interval": "1m",
  "timeout": 30000000000
}
```
* Used for email (password resets, notifications).
* Set proper SMTP credentials.
* Supports retry and timeout options.

### Account

```json
"account": {
  "pins_length": 5,
  "pins_emsil_subject": "Password Reset Request - Your PIN Inside",
  "pwd_validation": "required,min=8"
}
```

* Controls password reset pins and validation rules.
* Adjust subject line if needed.

### Parameters

```json
"param": { "icpp": 10, "icpp_max": 1000 }
```

* **icpp:** Default number of items per page (applied when the client does not specify a limit).
* **icpp_max:** Maximum number of items per page allowed in a request.

When calling an endpoint that returns a collection:

* If the client does not pass a limit parameter, the response will return up to icpp items (default).
* If the client does pass a limit, it will be honored up to a maximum of icpp_max. Any higher value is capped at icpp_max.

**Example:**

* Request:
```bash
GET /u/clients
```
Response includes up to 10 items (`icpp`).

* Request:
```bash
GET /u/clients?limit=20
```
Response includes up to 20 items.

* Request:
```bash
GET /u/clients?limit=5000
```
Response is capped at 1000 items (`icpp_max`).

### EMMS

```json
"emms": { "currency_local": 1, "currency_reference": 2 }
```

* Internal currency mapping. Don’t change unless you know your EMMS currency IDs.

### Image

```json
"image": {
  "aspect_ratio": 0.75,
  "width_resize": { "portrait": 480, "landscape": 1024 },
  "quality": 75
}
```

* Used for automatic resizing/compression of uploaded client/officer images.

### Cron

```json
"cron": [ "... list of procedures ..." ]
```

* Defines the stored procedures executed nightly by the cron binary.
* ~40 procedures covering arrears, portfolios, performance metrics, risk, etc.
* Don’t edit unless you know exactly what you’re doing.

## Notes for sysadmins

1. Always change default secrets (pepper, hmac_key, db.password, smtp.password).
2. Ensure cert paths point to your Let’s Encrypt or other SSL certs.
3. If cron fails, DB consistency may be affected. That’s why daily backups exist.
4. Keep a copy of production.json in your backup plan.
5. **Editing requires emms4 service restart.**