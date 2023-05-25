# vaultwardenGuide
A guide on how to install and host vaultwarden. 

This guide will show you one of many ways on how to run a personal vaultwarden instance. This method allows you to host behind your firewall, and as long as you have a VPN to your home network, you can access it from anywhere. All software and tools here will be completely free. Please keep in mind that I will be using Ubuntu, there may be differences if you are using other operating systems. All work will be done within the machine that is intended to host the vaultwarden instance

It will be step by step, so hopefully anyone can follow along. However, to accomplish this, we will need to successfully utilize these technologies in order to deploy the application:

- Docker
- Caddy Reverse Proxy utilizing DNS Cerificate Challenge (via let's Encrypt)
- GO programming language (prerequisite for xcaddy)
- xcaddy - Caddy package building tool
- DuckDNS
- Linux
- tailscale

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


















