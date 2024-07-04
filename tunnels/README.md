# Homeserver - Cloudflare Tunnels

In this section I will help you setup your personal server using ***Cloudflare Tunnels***. Use this if you either can't or prefer not to forward ports on your router.

## Goals & Features
After following this tutorial you will have:
- Secure access to your locally hosted services using [tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- Remote access to your server from anywhere by [VNC](https://en.wikipedia.org/wiki/Virtual_Network_Computing) and from LAN by [SSH](https://en.wikipedia.org/wiki/Secure_Shell) from any device you want
- Shared folders using [Samba](https://en.wikipedia.org/wiki/Samba_(software))
- Couple of web or standalone dockerized services
- Minecraft server with *mc.**your-domain.tld***

## 1. Install Your Preferred Linux Distribution

For the Linux distro, I will use [EndeavourOS](https://endeavouros.com/), but you can use any Arch-based distro (e.g. Manjaro, Garuda, or plain Arch) to essentially copy-paste commands. I chose EndeavourOS, because it comes with some useful stuff (that I will eventually need) installed and already configured and it has ISOs with many DE (KDE Plasma, Gnome, Xfce4 and more). If you opt for a non-Arch-based distro, you will need to find equivalent instructions for your chosen distribution.

- ### 1a. Update your system
	If you are using EndeavourOS just run ``yay`` in your terminal and type ``sudo`` password. For other distros find equivalent instructions.

- ### 1b. Turn off auto-sleep
	This depends of your distribution and your graphical enviroment. Just google how to do that. It shouldn't be complicated.

- ### 1c. Change shell (optional)
	This is just my preferance. You can completely ignore that step.

	Change your default shell to [zsh](https://www.zsh.org/) and enable plugins wiht [oh-my-zsh](https://ohmyz.sh/)

## 2. Remote connection
Setup VNC and SSH to remote access your soon-to-be headless server.
- ### 2a. VNC
> **IMPORTANT!** You need to either download some dummy X11 driver or buy dummy HDMI adapter for about 4 euro

	- 1. Install [RealVNC Viewer](https://www.realvnc.com/en/connect/download/viewer/) on your client (in my case Windows 11 Home).
	- 2. Install RealVNC Server on your server:

	```bash
	yay -S realvnc-vnc-server
	```
	```bash
	sudo systemctl enable vncserver-x11-serviced
	```
	```bash
	sudo systemctl start vncserver-x11-serviced
	```
 
	After you do this, login to your RealVNC account on RealVNC Server. Make sure you check ``SHA-256`` encryption.
	Reboot and boom! You have encrypted VNC connection! With VNC you can connect to your server **from anywhere**.
- ### 2b. SSH
  	Install ``SSH`` and connect to it.

  	```bash
	sudo systemctl enable sshd
  	```

  	```bash
	sudo systemctl enable sshd
  	```

  	then you can connect from any device within your LAN to your server by command:
  
  	```bash
   ssh <username>@<hostname/ip-address>
   	```

	for example:
   	```bash
   	ssh myAwesomeLinuxUsername@192.168.0.18
   	```
    
  	type password for your user nad congrats! You are connected via SSH! With SSH you can connect to your server **from LAN only**.

## 3. Docker & Docker Compose
Setup Docker with Docker Compose and add your user to "docker" group.
- ### 3.1. Install Docker and add user to "docker" group
	```bash
	yay -S docker
	```
	```bash
	sudo usermod -aG docker $USER
	```
	```bash
	newgrp docker
	```
	```bash
	sudo systemctl enable docker
	```
	```bash
	sudo systemctl start docker
	```
- ### 3.2 Install ``compose`` plugin
	> Visit offical [docker](https://docs.docker.com/compose/install/linux) website for instructions for your distribution
	```bash
	DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
	```
	```bash
	mkdir -p $DOCKER_CONFIG/cli-plugins
	```
	```bash
	curl -SL https://github.com/docker/compose/releases/download/v2.19.1/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
	```
	```bash
	chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
	```

### 4. Network & Firewall
Install and enable firewall to prevent common attacks:
```bash
yay -S firewalld
```
```bash
sudo systemctl enable firewalld.service
```
```bash
sudo systemctl start firewalld.service
```

### 5. Shared folders
Install ``Samba`` package:
```bash
yay -S samba
```

As ``Samba`` doesn't come with config file, we need to create one. I will use official config file from Samba [repository](https://git.samba.org/samba.git/?p=samba.git;a=blob_plain;f=examples/smb.conf.default;hb=HEAD).
Paste this config here:
```bash
sudo nano /etc/samba/smb.conf
```
In the section ``[global]`` change ``workgroup`` to following:
```bash
workgroup = WORKGROUP
```
so it will match Windows's default one.

- ### 5.1 Configure firewall
	In order to access your samba share from other computers, you must change your firewall's setting:
	```bash
	firewall-cmd --permanent --zone=public --add-service=samba
	```
	```bash
	firewall-cmd --reload
	```
	```bash
	systemctl enable --now smb.service
	```
	```bash
	systemctl enable --now nmb.service
	```

- ### 5.2 Samba group
	Create ``sambausers`` group and add yourself to it:
	```bash
	sudo groupadd -r sambausers
	```
	```bash
	sudo usermod -aG sambausers YOURUSERNAME
	```
	Create samba password for your shares:
	```bash
	sudo smbpasswd -a YOURUSERNAME
	```

- ### 5.3 Example share
	I will use my ``Jellyfin`` library as example yet practical share.

	Scroll to the bottom and add:
	```bash
	[Jellyfin]
	comment = Jellyfin media
	path = /home/docker/jellyfin/media
	writable = yes
	browsable = yes
	create mask = 0700
	directory mask = 0700
	read only = no
	guest ok = no
	```

	> At this point make sure that directory you specified in share's path actually exists! If not run [Jellyfin](services/jellyfin) service or create it:
	> ``sudo mkdir /home/docker/jellyfin/media``

	Change directory ownership and permissions:
	```bash
	sudo chown -R :sambausers /home/docker/jellyfin/media
	```
	```bash
	sudo chmod 1770 /home/docker/jellyfin/media
	```


## 5. Tunnels & Services
Setup [Portainer](services/portainer) with [Cloudflare Tunnels](services/tunnels) to allow access to your services outside your home network, then add as many services as you want.

- ### 5a. Services
	Here are details for setting some services. You can find all configs in [services](services) folder. Paste all of them in Portainer.
	- **[Jellyfin](services/jellyfin)** - The Free Software Media System
	- **[Jellyseerr](services/jellyseerr)** - Application For Managing Requests For Your Media Library
	- **[NextCloud](services/nextcloud)** - A Safe Home For All Your Data
	- **[Homarr](services/homarr)** - customizable browser's home page for your homeserver
	- **[Mealie](services/mealie)** - Recipe Management For The Modern Household
	- **[Linkding](services/linkding)** - Self-hosted bookmark manager
	- **[Uptime Kuma](services/uptime_kuma)** - A Fancy Self-Hosted Monitoring Tool
	- **[Minecraft](services/minecraft)** - Minecraft server with your own IP
	- **[dashdot](services/dash)** - a modern server dashboard
	- **[Watchtower](services/watchtower)** - update your Docker containers automatically
	- **[qBittorrent](services/qbittorrent)** - qBittorrent BitTorrent client
	- **[Starr Apps](services/starr_apps)** - collection managers apps with similar functionalities for anime, tv shows, movies, music and ebooks
	- **[Home Assistant](services/homeassistant)** - open source home automation that puts local control and privacy first
	- **[Custom service](services/custom)**