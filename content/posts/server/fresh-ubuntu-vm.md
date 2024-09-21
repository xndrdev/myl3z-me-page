+++
title = "Fresh Ubuntu and Docker installation "
date = "2024-09-21"
author = "xander"
tags = ["server"]
+++

# Ubuntu
Create a fresh Ubuntu virtual machine:

- Make sure that ISO Image is already somewhere on the Proxmox Installation
- Create new VM
- Use new Installer if asked while installing ubuntu
- Install OpenSSH
- After the Installation the system will reboot -> don't forget to remove the ISO from the drive

## Docker
I use Docker on the systems and I like Docker Compose.

- sudo apt update
- sudo apt upgrade
- sudo apt install docker.io

- sudo adduser mylez
- sudo usermod -aG sudo mylez

- su - mylez
- sudo usermod -aG docker mylez

Logout and login again and test if docker containers can be run:
- docker run hello-world

## Docker Compose
- sudo apt install ca-certificates curl gnupg
- sudo install -m 0755 -d /etc/apt/keyrings
- curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
- sudo chmod a+r /etc/apt/keyrings/docker.gpg

- echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
- sudo apt update
- sudo apt install docker-compose-plugin