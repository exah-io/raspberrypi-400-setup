This 'guide' assumes that the Desktop version of Raspberry Pi OS is being used and the onboarding setup has been done.
Tested on a Raspberry Pi 400.

To login to custom SSH port use: ssh pi@192.168.1.xx -p XXXXX

# Setup:
- Pi's home folder has more restricted permissions
- Sudo requires password
- UFW with incoming traffic blocked, except the custom SSH port
- SSH with custom port and password auth disabled
- fail2ban
- Apparmor
- 4K 60Hz and GPU memory reserved

```
# Update Raspberry Pi OS
sudo apt update
sudo apt full-upgrade
sudo reboot

# Restrict default user's home directory so that other users/groups cannot read pi's home folder
sudo chmod -R 750 /home/pi

# Make sudo require a password
sudo sed -i 's/NOPASSWD/PASSWD/g' /etc/sudoers.d/010_pi-nopasswd

# Install and setup firewall (UFW)
sudo systemctl enable ufw
sudo systemctl start ufw
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit XXXXX/tcp comment 'SSH port rate limit'

# Disable SSH login with password, change SSH port and enable the service
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
sudo sed -i 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config
sudo sed -i 's/#Port 22/Port XXXXX/g' /etc/ssh/sshd_config
sudo systemctl enable ssh
sudo systemctl start ssh

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

# Copy SSH public key to RPi (run ip -a to confirm IP)
ssh-copy-id pi@192.168.1.xx

# Install apparmor and enable it
sudo apt -y install apparmor
sudo sed -i 's/plymouth.ignore-serial-consoles/plymouth.ignore-serial-consoles lsm=apparmor/g' /boot/cmdline.txt

# Enable 4K 60Hz and reserve memory for GPU
# 4K 60Hz ref: https://www.raspberrypi.org/documentation/configuration/hdmi-config.md
# Memory reserve ref: https://www.raspberrypi.org/documentation/configuration/config-txt/memory.md
sudo tee -a /boot/config.txt << EOF

hdmi_enable_4kp60=1
gpu_mem=256
EOF
```
