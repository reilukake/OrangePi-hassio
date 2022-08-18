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
