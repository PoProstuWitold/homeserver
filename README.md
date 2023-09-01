# Homeserver

Welcome to my home server guide! Here, you'll find easy-to-follow steps to set up your own server at home using ***Linux***, ***Docker***, and all without the need for ***port-forwarding***, as many ISPs don't allow it. Please note, some parts of this guide are based on my personal preferences (e.g., the Linux distro), and your setup for certain things may slightly differ. Let's begin, shall we?

## Goals & Features
After following this tutorial you will have:
- Secure access to your locally hosted services using [tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/).
- Remote access to your server by [VNC](https://en.wikipedia.org/wiki/Virtual_Network_Computing) and [SSH](https://en.wikipedia.org/wiki/Secure_Shell) from any device you want.
- Shared folders using [Samba](https://en.wikipedia.org/wiki/Samba_(software)).
- Preconfigured, isolated & selfhosted cloud, media server, dashboard and "service managment center" using Docker and Portainer and as many more as you want.
- Minecraft server with *mc.**your-domain.com***

## 0. Things to consider & Requirements
Remember that your server is likely going to run 24/7, so keep in mind the energy consumption of your workstation and its noise. You can use your old PC, [Raspberry Pi 4](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/), or some mini PC (I recommend some older, used HP, Dell, Lenovo, or Intel NUC models). In this guide, I will be using an **[Intel NUC11TNHI5](https://www.intel.com/content/www/us/en/products/sku/205594/intel-nuc-11-pro-kit-nuc11tnhi5/specifications.html)** with 32GB RAM and a 2TB SSD, as it only consumes 28W of energy. It's not necessary to buy exactly the same hardware as mine to follow this tutorial.

In terms of hardware, here are my recommendations:
- **CPU**: at least 9th generation Intel Core i3 or i5 or AMD equivalent; 4+ cores.
- **GPU**: don't run the server with a gpu (unless you want your own ***[gaming cloud](https://en.wikipedia.org/wiki/Cloud_gaming)***) as you won't need it and it will greatly increase power consumption.
- **RAM**: I recommend a minimum of 8GB. If you're going to run lots of services, then 16GB or even 32GB may be necessary, especially if you want to run game servers. In 90% of cases, 64GB is overkill, but if you can afford it and want it, then go ahead.
- **STORAGE**: I recommend either going full SSD (at least 512GB) or using an SSD for the OS (128GB or 256GB) and an HDD (min. 512GB) for data. SSDs are more energy-efficient but also more expensive.

### All Requirements
To fully follow this tutorial you need:
- Your own domain.
- A Cloudflare account.
- A workstation where the server will be running.


## 1. Update BIOS and Install Your Preferred Linux Distribution
This step will be different depending on your hardware. Just google "bios download" and your motherboard name or name of your machine (PC, laptop).

For the Linux distro, I will use [EndeavourOS](https://endeavouros.com/), but you can use any Arch-based distro (e.g., Manjaro, Garuda, or plain Arch) to essentially copy-paste commands. I chose EndeavourOS, because it comes with some useful stuff (that I will eventually need) installed and already configured and it has ISOs with many DE (KDE Plasma, Gnome, Xfce4 and more). If you opt for a non-Arch-based distro, you will need to find equivalent instructions for your chosen distribution.

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
	Reboot and boom! You have encrypted VNC connection!
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
    
  	type password for your user nad congrats! You are connected via SSH!

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
	comment = Jellyfin's media
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
	- **[NextCloud](services/nextcloud)** - A Safe Home For All Your Data
	- **[Dashy](services/dashy)** - A Self-Hostable Personal Dashboard
	- **[Mealie](services/mealie)** - Recipe Management For The Modern Household
	- **[Briefkasten](services/briefkasten)** - Self-Hosted Bookmarking App
	- **[Uptime Kuma](services/uptime_kuma)** - A Fancy Self-Hosted Monitoring Tool
	- **[Minecraft](services/minecraft)** - Minecraft server with your own IP
	- **[Custom service](services/custom)**