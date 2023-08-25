+++
title = "Own Arch Linux Repository"
date = "2020-07-14"
description = "self-hosted repository for your own Arch Linux packages"

[taxonomies]
tags = ["linux", "nginx", "selfhosting"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

## Prerequisites
* Ubuntu Server with Nginx and Docker
---

## Creating repository

Repository database is managed via `repo-add` script bundled with Arch Linux `pacman` package manager. Since pacman is not available in Ubuntu repository I use docker `archlinux` image for managing repository. This guide assumes that repository located in `/srv/arch-repo`. First of all move all your packages into /srv/arch-repo. Following command will create or update repository database.

```sh
REPO_URL=repo.sun.hexor.ru
REPO_PATH=/srv/arch-repo
docker run -v ${REPO_PATH}:/repo --rm archlinux \
bash -c "repo-add /repo/${REPO_URL}.db.tar.gz /repo/*pkg.tar.zst"
```

### **Important aspect**
* Name of the database should be REPO_URL.db.tar.gz, in this case REPO_URL is repo.sun.hexor.ru.
---

## Periodically database repo update

I use systemd:
```ini
# Service unit
# /etc/systemd/system/update-arch-repo.service
[Unit]
Description=Updating arch linux repository database for %I
Requires=docker.service

[Service]
ExecStart=/usr/bin/docker run -v /srv/arch-repo:/repo --rm archlinux bash -c "repo-add /repo/%i.db.tar.gz /repo/*pkg.tar.zst"

[Install]
WantedBy=multi-user.target
```

```ini
# Timer unit
# /etc/systemd/system/update-arch-repo.timer
[Unit]
Description=Schedule arch repo database update for %I

[Timer]
# every 15 minutes
OnCalendar=*:0/15

[Install]
WantedBy=timers.target
```

Activate timer:
```sh
REPO_URL=repo.sun.hexor.ru
systemctl enable update-arch-repo@${REPO_URL}.timer
```

## Reverse proxy for HTTPS access

I use NGINX
```js
server {
    server_name repo.sun.hexor.ru;
    listen [::]:443 ssl;
    listen 443 ssl;
    include security.conf; # my security options
    include letsencrypt.conf; # my ssl config. 
    root /srv/arch-repo;
    location / {
        autoindex on;
        try_files $uri $uri/ =404;
    }
    access_log /var/log/nginx/logs/repo.sun.hexor.ru.access.log custom;
    error_log /var/log/nginx/logs/repo.sun.hexor.ru.error.log;
}
```

## Configure repo on your machines

Add your repo to `/etc/pacman.conf`:

```ini
[repo.sun.hexor.ru]
Server = https://repo.sun.hexor.ru
```

