+++
title = 'Certbot'
date = 2024-08-07T14:48:33+08:00
draft = false
tags = ['linux', 'ssl', 'letsencrypt', 'certbot']
+++

## install

```bash
sudo apt install nginx certbot python3-certbot-nginx
```

## certbot

```bash
certbot certonly --nginx
```

## certbot renew

```bash
certbot renew --dry-run

crontab -e
0 2 1 */2 * /usr/bin/certbot renew --quiet --renew-hook "systemctl reload nginx"
```
