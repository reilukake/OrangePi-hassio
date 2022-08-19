## Move files from SD to MMC
1. Boot from SD card
2. Login with default credentials root:1234
3. Change username/password
4. Create account
5. type sudo nand-sata-install
7. Choose Boot from eMMC - System on eMMC
8. Accept erasing data
9. Remove SD card
10. Reboot

## Initial config & Installing docker

- Update the OS
```
$ sudo apt update && sudo apt upgrade -y
```
 $ sudo armbian-config
	-  Install Docker via Softy
	-  or apt install docker
	-  Update other settings per your liking
-  Test docker installation by typing: 
```
$ sudo docker run hello-world
```
- Add yourself to docker usergroup and ensure docker starts on boot
```
$ sudo usermod -aG docker $USER
$ sudo systemctl start docker
$ sudo systemctl enable docker
```
- Install Docker-compose and check that it's working
```
$ sudo apt install docker-compose-plugin docker-compose
$ docker compose version
```
## Installing portainer
Portainer is a simple GUI that allows you to view and manage Docker containers.
```
$ cd opt
$ sudo nano docker-compose.yaml
```
Insert the following text into the yaml file
```
version: '3.0'

services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce
    restart: always
    ports:
      - "9000:9000/tcp"
    environment:
      - TZ=Europe/London
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/portainer:/data
```
Use docker-compose to install the container
```
docker-compose up -d
```
Navigate to http://hostname:9000/#!/init/admin and set up your username and password

## Installing Home Assistant
Add the following lines to your /opt/docker-compose.yml -file created in the previous part
```
version: '3'
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /opt/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
```
- Next run the docker-compose up -d command again in the /opt/ folder
- Now you can login into your home assistant install from http://deviceip:8123/ and start your initial Hassio config.

## Installing iFrame panels to Home Assistant
iFrame panels will let you embed other docker installation web interfaces to your Home Assistant GUI. For example, we can add the portainer GUI to the Home Assistant interface.

We start by editing the configuration.yaml -file in home assistant's directory. 
```
$ sudo nano /opt/homeassistant/config/configuration.yaml
```
Add the following lines to the TOP of the file:
```
panel_iframe:
  portainer:
    title: "Portainer"
    url: "http://192.168.0.214:9000/#/containers"
    icon: mdi:docker
    require_admin: true
```
Restart Home assistant from web gui

## Install MQTT and Tasmota
Containerized hassio install does not support Add-on store, so addons have to be installed manually via docker.

First we need to install Mosquitto MQTT -broker. We need to add the following to the docker-compose.yaml -file in /opt/
```
## Mosquitto MQTT
version: '2'
services:
  mosquitto:
    container_name: mosquitto
    image: eclipse-mosquitto
    restart: always
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - /opt/mosquitto/config:/mosquitto/config
      - /opt/mosquitto/data:/mosquitto/data
      - /opt/mosquitto/log:/mosquitto/log
```
Install mosquitto by running
```
$ docker-compose up -d
```
Check the installation from Portainer Containers

The mosquitto from docker-compose does not include the config file for Mosquitto. We will create a default config file by pulling the mosquitto repository from git.
```
$ cd $home
$ git clone https://github.com/eclipse/mosquitto.git eclipse-mosquitto
```

Now we need to change to the newly created mosquitto directory that we pulled with git and copy it to /opt/mosquitto -folder
```
$ cd $home/eclipse-mosquitto/
$ sudo cp mosquitto.conf /opt/mosquitto/config/
```
After that we need to reboot the container from Portainer or via command line.
![image](https://user-images.githubusercontent.com/49016081/185434680-10de6a09-1260-4421-bff8-521751780f06.png)
Next we can check the Mosquitto startup logs from Portainer Logs -icon
![image](https://user-images.githubusercontent.com/49016081/185434892-13d88069-1179-4eda-bbab-dcf410b5325b.png)

Now we need to create username & password for Mosquitto.
Open a terminal to the Mosquitto docker container
```
$ sudo docker exec -it mosquitto sh
```
Now we have a terminal open inside the Mosquitto container. (hassio is the username in the following command)
```
$ mosquitto_passwd -c /mosquitto/config/pwfile hassio
```
- Enter your desired password twice and press enter and exit the container with the "exit" command.
We need to change our mosquitto.conf to read the password from the created pwfile -file.
```
$ sudo nano /opt/mosquitto/config/mosquitto.conf
```
- Uncomment allow_anonymous false

![image](https://user-images.githubusercontent.com/49016081/185442465-67c4137a-f9a1-43e8-bb7d-fc9ee50b0536.png)
- Uncomment persistence and add true. Add your data folder to the persistence location.

![image](https://user-images.githubusercontent.com/49016081/185442545-78fb8e16-8882-4e46-bff3-8b82ad9cff3f.png)
- Uncomment listener field and add your port 1883 to the listener field.

![image](https://user-images.githubusercontent.com/49016081/185442661-13198d2d-fbf6-4489-a199-4e532747bac1.png)



Uncomment the "password_file" line and add the path /mosquitto/config/pwfile.
![image](https://user-images.githubusercontent.com/49016081/185436608-7b6dc11d-d65b-4a96-996d-dd76f923a31f.png)
- Restart Mosquitto from Portainer once again.
Now we are done with the Mosquitto configuration, lets integrate it with MQTT inside Home Assistant.
- Navigate to Settings->Devices&Services->Integrations
- Click "Add Integration"
- Add your Mosquitto -related information when prompted to do so.

![image](https://user-images.githubusercontent.com/49016081/185437963-f9da6daf-751f-4b4a-9a54-f5e718224116.png)

### Mosquitto and MQTT broker are now set up. 

## Setting up Wireguard VPN
Add the following to the bottom of your docker-compose.yaml -file in /opt/ and run "docker-compose up -d"
```
## Wireguard VPN
version: "2.1"
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Helsinki
      - SERVERURL=wireguard.domain.com #optional
      - SERVERPORT=51820 #optional
      - PEERS=1 #optional
      - PEERDNS=auto #optional
      - INTERNAL_SUBNET=10.13.13.0 #optional
      - ALLOWEDIPS=0.0.0.0/0 #optional
      - LOG_CONFS=true #optional
    volumes:
      - /path/to/appdata/config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```
### Remember to enable port forwarding from your Modem/Router to the Hassio's IP (Port 51820)
Next to check if the WireGuard installation is working correctly, type the following command and scan the QR code with your smart device's Wireguard Application
```
$ docker logs wireguard
```

# The following will be custom information/setup for different devices.

## Sonoff S26 Switches (Tasmota)
- Go to configuration->Configure MQTT
- Add your MQTT information to the fields below.
 
![image](https://user-images.githubusercontent.com/49016081/185445042-5b019272-2517-4f8a-9745-0aa0edd30c20.png)
- Save configuration and the device should be found in Hassio devices -tab.

## PiHole configuration
I use PiHole as a DHCP server so the config is pretty straightforward.
- Go to Settings->Devices&Services->Add Integration
- Find Pi-Hole integration
- Add Pi-Hole's IP address and click OK.
