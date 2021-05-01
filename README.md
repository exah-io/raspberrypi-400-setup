This 'guide' assumes that the desktop of Raspberry Pi OS is being used.
Tested on a Raspberry Pi 400.

# Setup
- Pi's home folder has more restricted permissions
- Sudo requires password
- UFW with incoming traffic blocked, except the custom SSH port
- SSH with custom port and password auth disabled
- fail2ban
- Apparmor (to be used with Docker profiles)
- 256MB of memory reserved for GPU

# Base setup
To login to custom SSH port use: ssh pi@192.168.1.xx -p XXXXX

```
# Copy SSH public key to RPi (run ip -a to confirm IP)
sudo systemctl start ssh
[run from client] ssh-copy-id pi@192.168.1.xx

# Change pi password
sudo passwd pi

# Update Raspberry Pi OS
sudo apt update
sudo apt full-upgrade
sudo reboot

# Restrict default user's home directory so that other users/groups cannot read pi's home folder
sudo chmod -R 750 /home/pi

# Make sudo require a password
sudo sed -i 's/NOPASSWD/PASSWD/g' /etc/sudoers.d/010_pi-nopasswd

# Disable SSH login with password, change SSH port and enable the service
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
sudo sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config
sudo sed -i 's/#Port 22/Port XXXXX/g' /etc/ssh/sshd_config
sudo systemctl enable ssh
sudo systemctl restart ssh

# Install and setup firewall (UFW)
sudo apt -y install ufw
sudo systemctl enable ufw
sudo systemctl start ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit XXXXX/tcp comment 'SSH port rate limit'
sudo ufw enable

# Setup fail2ban
sudo apt -y install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo tee -a /etc/fail2ban/jail.local << EOF
[ssh]
enabled  = true
port     = XXXXX
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime = -1
EOF

# Install apparmor and enable it
sudo apt -y install apparmor
sudo sed -i 's/rootwait/rootwait lsm=apparmor /g' /boot/cmdline.txt

# Setup automatic updates
sudo apt-get install -y unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades

# Reserve 256MB of memory for GPU
# 4K 60Hz ref: https://www.raspberrypi.org/documentation/configuration/hdmi-config.md
# Memory reserve ref: https://www.raspberrypi.org/documentation/configuration/config-txt/memory.md
# HDMI configurations: https://www.raspberrypi.org/documentation/configuration/config-txt/video.md
sudo tee -a /boot/config.txt << EOF

gpu_mem=256
EOF

# Disabling SAP plugin (bluetooth)
sudo sed -i 's/bluetooth\/bluetoothd/bluetooth\/bluetoothd --noplugin=sap/g' /etc/systemd/system/bluetooth.target.wants/bluetooth.service
sudo systemctl daemon-reload
sudo service bluetooth restart
```

# Syncthing
```
sudo apt -y install syncthing
sudo systemctl enable syncthing@pi.service
sudo systemctl start syncthing@pi.service
sudo ufw allow from 192.168.1.0/24 to any port 22000 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 21027 proto udp
```

# Steam Link and Kodi
```
sudo apt -y install steamlink joystick

sudo apt-get install -y kodi kodi-peripheral-joystick
sudo useradd -m -U -G "audio,bluetooth,input,plugdev,video" -s /bin/bash -u 1040 kodi
cat <<EOF | sudo tee /etc/systemd/system/kodi.service
[Unit]
Description = Kodi Media Center
After = systemd-user-sessions.service network.target sound.target

[Service]
User = kodi
Group = kodi
Type = simple
ExecStart = /usr/bin/kodi-standalone
Restart = always
RestartSec = 15

[Install]
WantedBy = multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable kodi
sudo systemctl start kodi
```

# Misc
## Issues with PS4 controller pairing via bluetooth:
There seems to be an issue with the currently installed version. Changing versions seems to work (TBC):
```
sudo apt install bluez=5.50-1.2~deb10u1
sudo apt-mark hold bluez
```
