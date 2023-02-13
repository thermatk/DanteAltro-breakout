# Project Dante Altro
## Preparing and installing OS
Get Ubuntu 22.04 Server, flash on a usb drive, install.
## Initial OS config
Update:

``` sh
sudo apt update && sudo apt upgrade
```

Set timezone:
``` sh
sudo timedatectl set-timezone Europe/Berlin
```

Kill cloud-init:
``` sh
sudo apt purge cloud-init cloud-guest-utils -y
sudo apt autoremove
sudo rm -rf /etc/cloud
```
Burn snap with fire:
``` sh
sudo snap remove lxd
sudo snap remove core20
sudo snap remove snapd
sudo systemctl stop snapd
sudo apt remove --purge snapd
sudo apt-mark manual squashfs-tools
sudo nano /etc/apt/preferences.d/nosnap.pref
# paste:
Package: snapd
Pin: release a=*
Pin-Priority: -10
# save and exit (Ctrl+O, Enter, Ctrl+X)
```
Remove rests:
``` sh
sudo apt -y autoremove
```
Enable automatic updates:
``` sh
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
# type "yes"
```
Add daily autoclean:
``` sh
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
# add
APT::Periodic::AutocleanInterval "1";
```

Update:

``` sh
sudo apt update && sudo apt upgrade
```
## Install Wireguard
Install wireguard:
``` sh
sudo apt install -y wireguard
```
Allow forwarding:

``` sh
# change sysctl
sudo nano /etc/sysctl.conf
# uncomment/edit
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
# apply sysctl
sudo sysctl -p
```

Using keys added to the passthrough server, add to the config:
``` sh
sudo nano /etc/wireguard/wg0.conf
```
Contents:
``` ini
[Interface]
Address = 10.10.10.5/24
MTU = 1280
PrivateKey = <dantealtro_privatekey>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eno1 -j MASQUERADE

[Peer]
PublicKey = <serv_publickey>
AllowedIPs = 10.10.10.0/24
Endpoint = svrgn.<DOMAIN>:51810
PersistentKeepalive = 25
```
Enable WireGuard server:
``` sh
sudo systemctl enable wg-quick@wg0.service
sudo systemctl start wg-quick@wg0.service
```
## Mount big HDD
Create the mountpoint:
``` sh
sudo mkdir -p /mnt/expandcloud
sudo chown -R $USER:$USER /mnt/expandcloud
```

Format, attach and add to fstab replacing with relevant `UUID`:
``` sh
# find device
lsblk
# format
sudo mkfs -t ext4 /dev/sdb1
# mount
sudo mount -t auto /dev/sdb1 /mnt/expandcloud
sudo chown -R $USER:$USER /mnt/expandcloud
# find UUID of new partition
lsblk -f
```
Add a mount point:
``` sh
sudo nano /etc/fstab
# add:
UUID=<YOURUUID> /mnt/expandcloud ext4 defaults,auto,users,rw,nofail,noatime 0 0
```
Reboot:
``` sh
sudo reboot
```
## Install Docker
Add repo:
``` sh
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install:

``` sh
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Add user to docker group:

``` sh
sudo groupadd docker
sudo usermod -aG docker $USER
sudo reboot
```

Set docker subnet:

``` sh
sudo nano /etc/docker/daemon.json
# paste
{
  "default-address-pools" : [
    {
      "base" : "172.26.0.0/16",
      "size" : 24
    }
  ]
}

```

## External

+ Point domain dantealtro.<> to passthrough server
+ Point passthrough to Wireguard 10.10.10.5

## Adjust DNS for future PiHole
Following [official guide](https://github.com/pi-hole/docker-pi-hole#installing-on-ubuntu-or-fedora).

Disable stub:
``` sh
sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf
```

Change symlink:
``` sh
sudo sh -c 'rm /etc/resolv.conf && ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf'
```

Restart resolved:
``` sh
sudo systemctl restart systemd-resolved
```

Docker:
``` sh
sudo nano /etc/docker/daemon.json
# paste
{
	 ...,
    "dns": ["1.1.1.1", "8.8.8.8"]
}

```

If still failing, kill resolved:
``` sh
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
sudo nano /etc/resolv.conf
# contents
nameserver 1.1.1.1
nameserver 8.8.8.8
```

## Setup docker compose
Create a folder for docker:

``` sh 
mkdir -p ~/dockerworld
```

Add docker-compose.yml:

``` sh
nano ~/dockerworld/docker-compose.yml
```

_Contents in [docker-compose.yml](DanteAltro/docker-compose.yml)._

And add .env:
``` sh
nano ~/dockerworld/.env
```

_Example contents in [.env](DanteAltro/.env)._

Start docker compose:
``` sh
cd ~/dockerworld
sudo docker compose up -d
```

## Configure Nextcloud
Nextcloud: `https://dantealtro.<DOMAIN>`

Configure via interface, afterwards adjust or add this to config:
``` sh
sudo nano ~/dockerworld/nextcloud/config/config.php
```

``` php
  'trusted_domains' => 
  array (
    0 => 'dantealtro.thermatk.com',
    1 => 'nextcloud',
  ),
  'preview_max_x' => 1024,
  'preview_max_y' => 1024,
  'default_phone_region' => 'DE',
  'enabledPreviewProviders' =>
  array (
    'OC\\Preview\\Image',
    'OC\\Preview\\HEIC',
    'OC\\Preview\\TIFF',
    'OC\\Preview\\Movie',
    'OC\\Preview\\MKV',
    'OC\\Preview\\MP4',
    'OC\\Preview\\AVI',
  ),
  'preview_max_memory' => 4096,
  'preview_max_filesize_image' => 256,
```

Limit Nextcloud preview quality for performance:
``` sh
# where 33 is www-data id
docker exec -u 33 nextcloud php occ config:app:set preview jpeg_quality --value="60"
```

Install the "Client Push" (notify_push) app from the appstore.

Run:

``` sh
docker exec -it -u www-data nextcloud php occ notify_push:setup https://dantealtro.<DOMAIN>/push
```

Install and configure Preview Generator:
``` sh
docker exec -u www-data nextcloud php occ preview:generate-all
```

For the Preview Generator, add a cron job:
``` sh
crontab -e
# add
0 0 * * * docker exec -t -u www-data nextcloud php occ preview:pre-generate
```
