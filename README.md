# Vaultwarden Guide
A guide on how to install and host Vaultwarden. 

This guide will show you how to run a personal Vaultwarden instance at home. This particular configuration allows you run the instance behind your firewall without having to expose port 80 for the HTTP-01 certificate challenge. Combined with a VPN, this proves to be a robust and private password managing solution.

All software and tools here will be completely free. It will be step by step, so hopefully anyone can follow along. However, to accomplish this, we will need to successfully utilize these technologies in order to deploy the application:

- Docker
- Caddy reverse proxy - utilizes DNS-01 certificate challenge (via ACME client)
- GO programming language - prerequisite for xCaddy
- xCaddy - Caddy package building tool
- DuckDNS
- Linux



## Initial step

Make a Directory called `vaultwarden` in your home profile. Shelve it for now, but later it will be needed when we make our `Caddyfile`, `Caddy` build and `docker-compose.yml`.



## Installing Docker

The first order of business is to install the Docker Engine and Docker Compose application on our Linux host. This guide will be using the latest version of Ubuntu Server. 

To install the Docker Engine, it is recommended that we add the repository made by Docker as it is the most up to date. The application from Docker's repository comes with Docker Compose. You can find the instructions [here](https://docs.docker.com/engine/install/ubuntu/) or follow along. You can omit the sudo commands if you are root.

1. Update apt packages and allow apt to use a repository over https:
```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```
2. Add Dockers GPG key:
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
5. Once the repository is added, update apt index again:
```
sudo apt-get update
```
6. Install the latest version of Docker:
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Installing xCaddy and building our custom Caddy package

Now that we have Docker installed, we can move on to installing xCaddy, which is a program that allows us to make a custom Caddy build. This build will be compiled from source because it needs to include an additional module that is not in the base package. This module will allow us to utilize the [DNS-01 certificate challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge)  by the ACME client on the reverse proxy. 
### GO

To utilize xCaddy, we need to make sure that we install the GO programming language which is a dependency for the xCaddy application. To do this we need to download the package from [GO's website](https://go.dev/dl/).

1. Download and extract the package:
```
wget https://go.dev/dl/go1.20.4.linux-amd64.tar.gz
tar -xzf go1.20.4.linux-amd64.tar.gz
```
2. You should see a GO application in your working directory. Move the installation to `/usr/local`:
```
mv go /usr/local
```
3. Add GO to the PATH environment variable
```
export PATH=$PATH:/usr/local/go/bin
```
4. Append that exact same line to the end of your `~/.bashrc` profile:
```
cd ~
nano .bashrc
```

Press `alt + /` to quickly navigate to the end of the file.

paste `export PATH=$PATH:/usr/local/go/bin` at the bottom of the file.

`ctrl + O` to save.

`ctrl + X` to exit.

6. Verify that GO is installed:
```
go --version
```


### xCaddy 

Now we can install xCaddy. 

1. Install Debian keyrings and allow apt over https:
```
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
```
2. Add Cloudsmith repository that contains xCaddy:
```
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-xcaddy-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-xcaddy.list
```
3. Update apt repositories and install xCaddy:
```
sudo apt update
sudo apt install xcaddy
```


### Creating the Caddy Build

Now that you have installed bot GO and xCaddy we can begin to compile our own caddy build. 

```
xcaddy build --with github.com/caddy-dns/duckdns
```

It will take a few minutes to compile, but once done it will spit out a `Caddy` binary.

And voila. You just compiled your own custom Caddy build that includes a DNS-01 certificate challenge against DuckDNS using ACME. 

Now move this `Caddy` build into `~/vaultwarden` directory that you made on the first step.

## DuckDNS

The next step will be for us create an account and to grab a free DNS name over at https://www.duckdns.org/. From here we can also grab the token that is required for the `docker-compose.yml` and `Caddy` file.

1. Once signed in, create a subdomain provided by DuckDNS and take note of the token.
	- This token is meant to be private, make sure not to share it with anyone.

3. Find out the current internal IP of your Vaultwarden host (usually 192.168.x.x or 10.x.x.x) and assign it to the subdomain that you just created.

## Preparing Docker Compose directory

Now onto the fun part! 

1. Navigate to the `vaultwarden` directory you made within the very first step:
```
cd ~/vaultwarden
```
2. Make a Docker Compose file:
```
touch docker-compose.yml
```
3. Use Nano to edit the file:
```
nano docker-compose.yml
```
4. Copy the contents below into your `docker-compose.yml` and make sure that you change the variables for `EMAIL`, `DOMAIN` (your DuckDNS domain), and `DUCKDNS_TOKEN` (your DuckDNS token).

	- Important note: When entering the `DOMAIN` variable, please ensure that you continue to use the `https://` even though DuckDNS's website specifies `http://`. This is because Caddy will be utilizing the `https://` service with port `:443`. If the prefix and the port mismatch (i.e. `http://` and `:443`) the Docker container for Caddy will fail and restart over and over again.

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
5. Make a `Caddyfile` in your `~/vaultwarden` directory:
```
sudo touch Caddyfile
```
6. Copy the contents below into the `Caddyfile`. There is no need to modify the `$DOMAIN` and `$DUCKDNS_TOKEN` variables, as they are passed as environmental variables when Docker Composition starts.
```
{$DOMAIN}:443 {
  log {
    level INFO
    output file {$LOG_FILE} {
      roll_size 10MB
      roll_keep 10
    }
  }

  # Use the ACME DNS-01 challenge to get a cert for the configured domain.
  tls {
    dns duckdns {$DUCKDNS_TOKEN}
  }

  # This setting may have compatibility issues with some browsers
  # (e.g., attachment downloading on Firefox). Try disabling this
  # if you encounter issues.
  encode gzip

  # Notifications redirected to the WebSocket server
  reverse_proxy /notifications/hub vaultwarden:3012

  # Proxy everything else to Rocket
  reverse_proxy vaultwarden:80
}
```

## Running the containers

Once all said and done, you should be within your `~/vaultwarden` directory. The directory should contain a `docker-compose.yml` file, a `Caddy` binary, and a `Caddyfile`.

If you have confirmed that all of the components are there and are properly configured, you are ready to kick off the Docker Composition with your  `docker-compose.yml` file.

While within the directory run:
```
sudo docker compose up -d
```

## Verify

Navigate to the domain that you made in DuckDNS. It should take you to the Vaultwarden login page. 

If you are having trouble with the setup after running the Docker Compose command, you can run the command below to stop the containers and troubleshoot:
``` 
sudo docker compose down
```
You can then run: 
``` 
sudo docker compose up
```

This will run the containers and display information that is occurring within them. This can help find errors and troubleshoot. 





