+++
title = 'pve postgresql'
date = 2024-11-13T10:30:48+08:00
draft = false
tags = ['postgresql' , 'pve', 'lxc']
+++

## Install

login to pve
```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/postgresql.sh)"
```

assume lxc container's id is 201, then login to container
```bash
lxc-attach 201 or ssh root@your_container_ip
```

reset password
```bash
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD 'your_password';
```

create new user
```bash
CREATE USER your_username WITH PASSWORD 'your_password';
```

create new database
```bash
CREATE DATABASE your_database_name;
```

grant permission to user
```bash
GRANT ALL PRIVILEGES ON DATABASE your_database_name TO your_username;
```

## Reference

- [ProxmoxVE](https://github.com/community-scripts/ProxmoxVE)
