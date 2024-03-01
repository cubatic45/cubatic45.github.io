+++
title = 'Docker'
date = 2024-02-22T12:00:05+08:00
draft = false
tags = ['docker', 'go', 'linux', 'debian']
+++

## Docker install

##### 1. Install from script

```sh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```
##### 2. Install using the apt repository

```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update

# Install Docker
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Run Docker commands without sudo

##### 1. Add the `docker` group if it doesn't already exist

```sh
sudo groupadd docker
```

##### 2. Add the connected user `$USER` to the docker group


```sh
sudo gpasswd -a $USER docker
```

##### 3. Restart the `docker` daemon

```sh
sudo service docker restart
```
