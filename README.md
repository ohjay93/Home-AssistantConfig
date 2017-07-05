# Home-Assistant
Current Version: 0.47.1 
Running on a Raspberry PI 2 Model B  
Raspbian Jessie 8.0
Razberry Z-wave controller

# Setup
### Usual Raspbian setup 
* Install
* Boot into Pixel
* Join Wifi
* Fix keyboard language
* Change hostname to HomeAssistant
* Set Timezone
* Expand file system
* Shutdown

### Update and install dependencies
```
sudo apt-get update
sudo apt-get dist-upgrade
sudo apt-get install python3 python3-venv python3-pip
sudo apt-get clean
```

### Install Home Assistant old school-style
NOTE: Perhaps not use sudo - might prevent dependencies from updating later
```
sudo pip3 install homeassistant
sudo pip install netdisco
sudo pip install astral
sudo pip3 install plexapi
sudo pip3 install xmltodict


```

#### Install Z-Way Razberry support
```
wget -q -O - razberry.z-wave.me/install | sudo bash
```
open the panel: http://192.168.100.20:8083

#### Disable Z-way services
delete/move z-way-server script from src/init.d folder

#### Install open-zwave
```
sudo apt-get install cython3 libudev-dev python3-sphinx python3-setuptools git
sudo pip3 install --upgrade cython==0.24.1
git clone https://github.com/OpenZWave/python-openzwave.git
cd python-openzwave
git checkout python3
PYTHON_EXEC=$(which python3) make build
sudo PYTHON_EXEC=$(which python3) make install
```

#### Install OZWCP
```
sudo apt-get -y install libgnutls28-dev libgnutlsxx28
cd
wget ftp://ftp.gnu.org/gnu/libmicrohttpd/libmicrohttpd-0.9.19.tar.gz
tar xf libmicrohttpd-0.9.19.tar.gz
mv libmicrohttpd-0.9.19 libmicrohttpd
cd libmicrohttpd
./configure
make
sudo make install
cd
git clone https://github.com/OpenZWave/open-zwave-control-panel.git
cd open-zwave-control-panel
nano Makefile
```
edit lines 24 & 25 to be:
```
OPENZWAVE := ../python-openzwave/openzwave  
LIBMICROHTTPD := /usr/local/lib/libmicrohttpd.a
```
Then uncomment lines 37 & 38, and comment out lines 44 & 45. Save & close the file

### Build ozwcp
```
make
ln -sd ../python-openzwave/openzwave/config
sudo systemctl stop home-assistant.service
```
Run OZWCP
```./ozwcp -p 8888```

### Install SMB
```
sudo apt-get install samba
sudo nano /etc/samba/smb.conf
```
Add/Edit the rules below under the "Share Definitions" section
```
[home assistent]
path = /home/pi/.homeassistant
comment = No comment
browsable = yes
read only = no
valid users =
writable = yes
guest ok = yes
public = yes
create mask = 0777
directory mask = 0777
force user = root
force create mode = 0777
force directory mode = 0777
hosts allow =
```
```
sudo smbpasswd -a pi
sudo service smbd restart
```

### Autostart Home Assistant
Find out where hass is located 
```
whereis hass
```	
Set up systemd
```
su -c 'cat <<EOF >> /etc/systemd/system/home-assistant@[your user].service
[Unit]
Description=Home Assistant
After=network.target

[Service]
Type=simple
# Enable the following line if you get network-related HA errors during boot
#ExecStartPre=/usr/bin/sleep 60
# Use hass: /usr/local/bin/hass to determine the path of hass
ExecStart=/usr/local/bin/hass -c "/home/pi/.homeassistant"

[Install]
WantedBy=multi-user.target
EOF'
```
```
sudo systemctl --system daemon-reload
sudo systemctl enable home-assistant@pi
sudo systemctl start home-assistant@pi
```
reboot!

### Fix z-wave config for HASS:
Add this to configuration.yaml:

```
zwave:
  usb_path: /dev/ttyAMA0
  config_path: /usr/local/lib/python3.4/dist-packages/libopenzwave-0.3.2-py3.4-linux-armv7l.egg/config/
  polling_interval: 10000
  debug: 0
```

### Git Setup
* https://home-assistant.io/cookbook/githubbackup/


### Install MySQL DB
```
sudo apt-get install mysql-server && sudo apt-get install mysql-client
sudo apt-get install libmysqlclient-dev
```
* you will be asked for a password for the db
```
sudo apt-get install python-dev python3-dev
mysql -uroot -p
CREATE DATABASE dbname;
CREATE USER 'dbuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON dbname.* TO 'dbuser'@'localhost';
FLUSH PRIVILEGES;
```
**Test if user works:**  
```
mysql -udbuser dbname -p
```
```
sudo apt-get install libmysqlclient-dev
sudo pip3 install mysqlclient

```  
**Add to configuration.yaml**  
```
recorder:
  db_url: mysql://dbuser:password@localhost/dbname
  purge_days: 3  
  
```

## DuckDNS
Instructions from:
https://www.duckdns.org/install.jsp?tab=pi
```
mkdir duckdns
cd duckdns
nano duck.sh
```
Copy and modify this:
```
echo url="https://www.duckdns.org/update?domains=exampledomain&token=a7c4d0ad-114e-40ef-ba1d-d217904a50f2&ip=" | curl -k -o ~/duckdns/duck.log -K -
```
```
chmod 700 duck.sh
crontab -e
```
Copy this:
```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```
```
sudo service cron start
```


## LetEncrypt
* Guide here: https://www.youtube.com/watch?v=BIvQ8x_iTNE
```
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
```
Run the following, but update with your email and duckDNS address!
```
./letsencrypt-auto certonly --email youremail@address.com -d example.duckdns.org
```
On the "how would you like to authenticate with the Let's Encrypt CA?" page select "2  Automatically use a temporary webserver (standalone)"

Remove the port forwards setup eariler, and set port 443 to forward to port 8123 on in your router settings.
```
sudo chmod -R 777 /etc/letsencrypt
```
Add and modify this to configuration.yaml:
```
http:
  api_password: yourpassword
  ssl_certificate: '/etc/letsencrypt/live/yourdomain.duckdns.org/fullchain.pem'
  ssl_key: '/etc/letsencrypt/live/yourdomain.duckdns.org/privkey.pem'
```

#### Renew Certificate
On the Router:
Map External port number 443 to Internal port number 443 (instead of 8123)

Find any process PID using port  443 and kill it:
```
sudo netstat -tulpn
sudo kill [PID]
cd letsencrypt
./letsencrypt-auto certonly --email [my email]  -d [my domain]
```
Choose 1: Spin up a temporary webserver (standalone)
Map External port number 443 to Internal port number 8123 (instead of 443)

* Creation date 2017 July 5
* Renewal date  2017 October 3

# Useful commands
### Start / stop / restart Home Assistant
```
sudo systemctl start home-assistant@pi.service
sudo systemctl stop home-assistant@pi.service
sudo systemctl restart home-assistant@pi.service
```
### UPGRADING
```
pip3 install --upgrade homeassistant
```
### Check config
```
hass --script check_config
```
### Start Open-Zwave-Control-Panel
```
sudo ./ozwcp -p 8888
```
### Link openzwave
```
sudo ln -s /srv/homeassistant/homeassistant_venv/lib/python3.4/site-packages/libopenzwave-0.3.2-py3.4-linux-armv7l.egg/config /srv/homeassistant/src/open-zwave-control-panel/config
```
### Commit to GitHub
```
./gitupdate.sh
```

# Notes
### Important things Z-way does that might make installing it superfluous:
* Preparing AMA0 interface:
 removing 'console=ttyAMA0,115200' and 'kgdboc=ttyAMA0,115200 and 'console=serial0,115200' from kernel command line (/boot/cmdline.txt)
 removing '*:*:respawn:/sbin/getty ttyAMA0' from /etc/inittab
mv: cannot stat ‘/tmp/zway_install_inittab’: No such file or directory
AMA0 interface reconfigured, please restart Raspberry