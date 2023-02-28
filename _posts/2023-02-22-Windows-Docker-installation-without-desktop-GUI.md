---
title: Windows Docker installation without desktop GUI
date: 2023-02-22 5:00:00 
categories: [windows, wsl]
tags: [windows, docker, wsl, ubuntu]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

The aim is to install Dockerd with Docker Compose to manage containers on my local machine. This installation instructions are done to WSL Ubuntu on Windows 11. 

This is not complete tutorial how to use docker or docker compose. Just a brief documentation how to install Dockerd to your windows 11 machine without having to use Docker Desktop.
I didn't want to use docker desktop since there's really no need for it in my use. If I would like to use GUI for managing my containers I would use locally something like [Portainer](https://github.com/portainer/portainer) instead.
If you need to install WSL follow this documentation [Install Linux on Windows with WSL](https://learn.microsoft.com/en-us/windows/wsl/install)

Open WSL Ubuntu and let's get going. 

>**Warning**  
>This guide doesn't work with [vscode local dev containers](https://code.visualstudio.com/docs/devcontainers/containers). I found out this because out of the blue I had a need to start figure out how to create developer containers and noticed this configuration doesn't work at all. I didn't have time or energy to find out how to configure this properly. It's on my mile long ToDo-list.
>
>For now I'm back to Docker Desktop vscode extension which installs docker desktop itself. 
{: .prompt-danger }

## Installation
### Install pre-required packages
```bash
sudo apt update
sudo apt install --no-install-recommends apt-transport-https ca-certificates curl gnupg2
```

### Configure docker package repository
```bash
source /etc/os-release
curl -fsSL https://download.docker.com/linux/${ID}/gpg | sudo apt-key add -
sudo apt-key list

# Change legacy trusted.gpg --> ...gpg.d
sudo cp /etc/apt/trusted.gpg /etc/apt/trusted.gpg.d

echo "deb [arch=amd64] https://download.docker.com/linux/${ID} ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

### Install Docker & add current user to docker group
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
```

### Configure Dockerd
```bash
DOCKER_DIR=/mnt/wsl/shared-docker
mkdir -pm o=,ug=rwx "$DOCKER_DIR"
sudo chgrp docker "$DOCKER_DIR"
sudo mkdir /etc/docker
sudo nano /etc/docker/daemon.json
```
Edit daemon.json and add following content
```json
{
   "hosts": ["unix:///mnt/wsl/shared-docker/docker.sock"]
}
```

Run command `sudo dockerd` to see if docker installation is working. Working installation should return in the end `API listen on /mnt/wsl/shared-docker/docker.sock` .

I had to update iptables due to getting Error message instead of `API listen on...`  so deal with the errors accordingly (google ftw)
```bash
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

To always start docker automatically when you open ubuntu add following line to `.bashrc` file (located in /home/username folder). I just scrolled down to end of the file and pasted 
```bash
# Start docker automatically
DOCKER_DISTRO="Ubuntu-20.04"
DOCKER_DIR=/mnt/wsl/shared-docker
DOCKER_SOCK="$DOCKER_DIR/docker.sock"
export DOCKER_HOST="unix://$DOCKER_SOCK"
if [ ! -S "$DOCKER_SOCK" ]; then
   mkdir -pm o=,ug=rwx "$DOCKER_DIR"
   sudo chgrp docker "$DOCKER_DIR"
   /mnt/c/Windows/System32/wsl.exe -d $DOCKER_DISTRO sh -c "nohup sudo -b dockerd < /dev/null > $DOCKER_DIR/dockerd.log 2>&1"
fi
```

And added permissions to start dockerd without password prompt by adding following line to visudo file (`sudo visudo`)
```bash
%docker ALL=(ALL) NOPASSWD: /usr/bin/dockerd
```

Everthing should be done. Now you can test the installation by creating simple html page, docker compose file and starting the container.
I had docker compose installed automatically but if it's missing use following command `sudo apt-get install docker-compose-plugin`.

## Testing the installation

Just a simple web page which is served by nginx web server from the container.

Create a folder and navigate to it. Create following file strucuture inside the folder
```
├── data
│   └── index.html
└── docker-compose.yml
```

Add following content to Index.html
```html
<!doctype html>
<html lang="en">
<head>
    <title>Nginx with Docker Compose</title>
</head>
<body>
    <h2>Install Nginx using Docker Compose.</h2>
    <p>This content is being served by an Nginx Docker container.</p>
</body>
</html>
```

Add following content to docker-compose.yml:
```docker
version: '3'
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./data:/usr/share/nginx/html
```

Now run command `docker compose up` and wait a bit. 
When you see "Start worker process..." navigate to http://localhost:8080/ with your browser. 
You should see your index page opening up. 

![Docker compose results in browser](/assets/img/2023-02-22-Windows-Docker-installation-without-desktop-GUI/browser.png)

The reason why I didn't add --detach to up command is to easily see if there's any errors. `docker compose up` vs. `docker compose up -d` --> `docker compose logs --f` ...

Don't forget to shutdown & remove your test container after you are done by entering command `docker compose down` 
also remove nginx image to keep your image gallery neat and tidy (unless you need this image later on the road). Get images list by running `docker images` command
```shell
userName@host:~/helloworld$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        latest    3f8a00f137a0   13 days ago   142MB
```

Use image name & tag or image id to remove the image
`docker rmi nginx:latest` or `docker rmi 3f8a00f137a0`

Source for installation instructions: [Solita blog post](https://dev.solita.fi/2021/12/21/docker-on-wsl2-without-docker-desktop.html) which is modified to get working on my machine.
Rest are more or less from my previous work.