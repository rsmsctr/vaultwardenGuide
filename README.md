# vaultwardenGuide
A guide on how to install and host vaultwarden. 

This guide will show you one of many ways on how to run a personal vaultwarden instance. This method allows you to host privately WITH HTTPS, and as long as you have a VPN to your home network, you can access it from anywhere. All software and tools here will be completely free. Please keep in mind that I will be using Ubuntu, there may be differences if you are using other operating systems. All work will be done within the machine that is intended to host the vaultwarden instance

It will be step by step, so hopefully anyone can follow along. However, to accomplish this, we will need to successfully utilize these technologies in order to deploy the application:

- Docker
- Caddy Reverse Proxy utilizing DNS Cerificate Challenge (via let's Encrypt)
- GO programming language (prerequisite for xcaddy)
- xcaddy - Caddy package building tool
- DuckDNS
- Linux
- tailscale

## Initial Step

1. Make a Directory called `vaultwarden` in your home profile. Shelve it for now, but later it will be needed when we make our `Caddyfile`, `Caddy` build and `docker-compose.yml`.


## Installing Docker

The first order of business is to install the docker engine and docker compose application on our linux host. I am personally using ubuntu LTS.
To install the docker engine, it is recommended that we add the repository made by Docker as it is the most up to date and then install the applications from there. You can find the instructions [here](https://docs.docker.com/engine/install/ubuntu/) or follow along.

1. Update apt packages and allow apt to use a repository over https:
```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```
2. Add Dockers GPG Key:
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
4. Set up the repository:
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
5. Once the repository is added, update apt's index again:
```
sudo apt-get update
```
6. Install the latest version of Docker:
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Installing xcaddy and building our custom caddy package

Now that we have docker installed, we can move on to installing xcaddy, which is a program that allows us to make a custom caddy builds from source that include additional modules that are not typically included. We will be using xcaddy to make a Caddy build that allows use to a module that utilizes a [DNS certificate challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge) against DuckDNS. Caddy includes Let's Encrypt by default which will be performing this challenge. 

### GO

Right off the bat to utilize xcaddy, we need to make sure that we install the GO programming language which is a dependency for the xcaddy application. To do this we need to download the package from [GO's website](https://go.dev/dl/).

1. Download the package for Linux:

`wget https://go.dev/dl/go1.20.4.linux-amd64.tar.gz`

2. Extract the package:

`tar -xzf go1.20.4.linux-amd64.tar.gz`

3. You should see a GO application in your working directory. Move the installation to /usr/local

`mv go /usr/local`

4. Add GO to the PATH environment variable

`export PATH=$PATH:/usr/local/go/bin`

5. Also append that exact same line to the end of your .bashrc profile

`cd ~`

`nano .bashrc`

Press `alt + /` to quickly navigate to the end of the file

paste `export PATH=$PATH:/usr/local/go/bin` at the bottom of the file

`ctrl + O` to save

`ctrl + X` to exit

6. Verify that GO is installed

`go --version`

### xcaddy 

Now we can finally install xcaddy. 

1. install debian keyrings and allow apt over https

`sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https`

2. add cloudsmith repo that contains xcaddy

`curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-xcaddy-archive-keyring.gpg`

`curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-xcaddy.list`

3. Update apt repos and install xcaddy

`sudo apt update`
`sudo apt install xcaddy`

### Creating the Caddy Build

Now that you have installed bot GO and xcaddy we can begin to compile our own caddy build. 

Simply type `xcaddy build --with github.com/caddy-dns/duckdns`

It will take a few minutes to compile, but once done it will spit out a `caddy` binary.

And voila - you just compiled your own custom caddy build that includes a dns certificate challenge against duckDNS using let's encrypt. 

Now move this `caddy` build into `~/vaultwarden` directory that you made on the first step.


## DUCKDNS

The next step will be for us create an account and to grab a free DNS name over at https://www.duckdns.org/. From here we can also grab the token that is required for the docker-compose.yml and caddy file.

1. Once signed in, create a sub domain provided by Duck DNS.

2. Find out the current internal IP of your vaultwarden host ( usually 192.168.x.x or 10.x.x.x) and assign it to the domain name.

## Preparing Docker Compose and running the containers

Now onto the fun part! 

1. Navigate to the directory you made within the very first step 

`cd ~/vaultwarden`

2. make a docker compose file

`touch docker-composeyml`

3. Use nano to edit the file:

`nano docker-compose.yml`

4. Copy this into your docker-compose.yml and make sure that you change the variables for `EMAIL`, `DOMAIN` (your duckDNS domain), and `DUCKDNS_TOKEN` (your duckDNS token)

- Important note: When entering the `DOMAIN` variable, please ensure that you continue to use the `https://` even though DUCKDNS's website specifies `http://`. This is because Caddy will be utilizing the `https://` service with port `:443`. If the prefix and the port mismatch (ie `http://` and `:443`) the docker container for Caddy will fail and restart over and over again. Resulting in :(. 

```
version: '3'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      WEBSOCKET_ENABLED: "true"  # Enable WebSocket notifications.
    volumes:
      - ./vw-data:/data

  caddy:
    image: caddy:2
    container_name: caddy
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./caddy:/usr/bin/caddy  # Your custom build of Caddy.
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy-config:/config
      - ./caddy-data:/data
    environment:
      DOMAIN: "https://vaultwarden.example.com"  # Your domain.
      EMAIL: "admin@example.com"                 # The email address to use for ACME registration.
      DUCKDNS_TOKEN: "<token>"                   # Your Duck DNS token.
      LOG_FILE: "/data/access.log"
```
















