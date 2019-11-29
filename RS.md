
# Logged as root, install sudo, git, make, vim, net_tools(for ifconfig)
apt-get install sudo git make vim net-tools
# add user to sudoers
/usr/sbin/usermod -aG sudo roger
# go back to your non-root user and use sudo when you need root privileges
# Check if user can do sudo operations
sudo fdisk -l

# == Network and security part == #

:::::::::::::::
:: Static IP ::
:::::::::::::::
## We don’t want you to use the DHCP service of your machine. 
## You’ve got to configure it to have a static IP and a Netmask in \30.
(DHTP: is a network server that automatically provides and assigns IP addresses, 
    default gateways and other network parameters to client devices.
    known as Dynamic Host Configuration Protocol.)

sudo vim /etc/network/interfaces
replace dhcp by static
    address 		10.13.42.42
    gateway 		10.13.254.254 # Same as host
    netmask         255.255.255.252
    broadcast 		10.13.255.255

sudo reboot

:::::::::	
:: SSH ::
:::::::::
(Secure Shell (SSH) is a cryptographic network protocol for operating network services 
securely over an unsecured network.
Typical applications include remote command-line, login, and remote command execution, 
but any network service can be secured with SSH.
SSH provides a secure channel over an unsecured network in a client–server architecture, 
connecting an SSH client application with an SSH server.)
## You have to change the default port of the SSH service by the one of your choice. 
## SSH access HAS TO be done with publickeys. 
## SSH root access SHOULD NOT be allowed directly, but with a user who can be root.
server side:
sudo systemctl status ssh 
(check ssh, if not install) sudo apt install openssh-server
sudo vim /etc/ssh/sshd_config
uncomment Port 22 and replay by 2222
uncomment PasswordAuthentication change to no
Uncomment PermitRootLogin and replace prohibit-password by no
sudo service sshd restart

Client side:
# Generate a publickey to access VM via SSH
ssh-keygen
# Copy the publickey into the VM publickeys file
cd .ssh/
ssh-copy-id -i id_rsa.pub roger@10.13.42.42 -p 2222
# connect
ssh roger@10.13.42.42 -p 2222

::::::::::::::
:: Firewall ::
::::::::::::::
## You have to set the rules of your firewall on your server 
## only with the services used outside the VM.

# show firewall config
sudo iptables -L
# Installing iptables-persistent to make the rule change permanent
sudo apt install iptables-persistent
# Start the service
sudo service netfilter-persistent start
# get current setting 
sudo iptables -L
# create new a default config file
sudo vim /etc/network/if-pre-up.d/iptables

(TCP (Transmission Control Protocol), UDP(User Datagram Protocol)
DUP throws out all the error-checking stuff.
Losing all this overhead means the devices can communicate more quickly.
UDP is used when speed is desirable and error correction isn’t necessary. 
For example, UDP is frequently used for live broadcasts and online games.)

#!/bin/bash
# Drop everything as default behavior
iptables -F

# Loopback
(The loopback device is a special, virtual network interface 
that your computer uses to communicate with itself. 
It is used mainly for diagnostics and troubleshooting, 
and to connect to servers running on the local machine.
任何送到该接口的网络数据报文都会被认为是送往设备自身的。
大多数平台都支持使用这种接口来模拟真正的接口。)

iptables -A INPUT	-i lo -p all				-j ACCEPT
iptables -A OUTPUT	-o lo               		-j ACCEPT

# DNS
iptables -A INPUT	-p tcp	-m tcp	--dport	53			-j ACCEPT
iptables -A OUTPUT	-p udp	-m udp	--dport	53			-j ACCEPT

# SSH
iptables -A INPUT	-p tcp	-m tcp	--dport	2222			-j ACCEPT

# HTTP
iptables -A INPUT 	-p tcp	-m tcp	--dport	80			-j ACCEPT

# HTTPS
iptables -A INPUT	-p tcp	-m tcp	--dport	443			-j ACCEPT

# Allow pings
iptables -A OUTPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

# Already established connections
iptables -A INPUT	-m state	--state ESTABLISHED,RELATED	-j ACCEPT
iptables -A OUTPUT	-m state	--state ESTABLISHED,RELATED	-j ACCEPT

# apply
sudo chmod +x /etc/network/if-pre-up.d/iptables
sudo bash /etc/network/if-pre-up.d/iptables
sudo service netfilter-persistent reload
sudo iptables -L

:::::::::
:: DOS ::
:::::::::

## You have to set a DOS (Denial Of Service Attack) protection on your open ports of your VM.
(ref: https://www.scaleway.com/en/docs/protect-server-fail2ban/
https://www.supinfo.com/articles/single/2660-proteger-votre-vps-apache-avec-fail2ban)
# install
sudo apt-get install fail2ban apache2

# By default Fail2ban keeps all the configuration files in /etc/fail2ban/ directory. 
# The main configuration file is jail.conf, it contains a set of pre-defined filters. 
# Override it by creating a new configuration file jail.local inside /etc/fail2ban/ directory.
cd /etc/fail2ban
sudo cp jail.conf jail.local
sudo vim jail.local

# default
ignoreip =  127.0.0.1/8 10.13.42.42
# in SSH SERVERS SECTION, 
replace all port = ssh by port = 2222
# jails HTTP servers
[apache]
enabled  = true
port     = http,https
filter   = apache-auth
logpath  = /var/log/apache2/*error.log
maxretry = 3
bantime  = 600
findtime = 600
ignoreip = 10.13.42.42

# DOS protection
[apache-dos]

enabled  = true
port     = http,https
filter   = apache-dos
logpath  = /var/log/apache2/access.log
bantime  = 600
maxretry = 300
findtime = 300
action   = iptables[name=HTTP, port=http, protocol=tcp]
ignoreip = 10.13.42.42

# ADD THE FOLLOWING LINES TO THE SECTIONS [apache-badbots] [apache-noscript] [apache-overflows]

enabled  = true
filter   = section-name
logpath  = /var/log/apache2/*error.log
bantime = 600
maxretry = 3
findtime = 600
ignoreip = 10.13.42.42
# save and exit
# Create apache-dos.conf file in filters.d folder:
cd /etc/fail2ban/filters.d/
sudo vim apache-dos.conf

# Add the following :
[Definition] 
failregex = ^ <HOST> -. * "(GET | POST). *
ignoreregex =
# save and exit
# restart the service
sudo service fail2ban restart
# check status
sudo fail2ban-client status sshd

# To check if firewall rule applied
# Try to connect via ssh to the machine with wrong login/password until blocked
# To unblock IP address, go back to the VM
sudo fail2ban-client status sshd
# to check that your ipaddress is in the banned section
sudo fail2ban-client set sshd unbanip your_ip_address
# then restart fail2ban service
sudo service fail2ban restart

:::::::::::::::::::::
:: Scan protection ::
:::::::::::::::::::::

## You have to set a protection against scans on your VM’s open ports.
https://wiki.debian-fr.xyz/Portsentry
https://en-wiki.ikoula.com/en/To_protect_against_the_scan_of_ports_with_portsentry
# Installing protection against port scanning
sudo apt install portsentry
sudo /etc/init.d/portsentry stop
cd /etc/default
sudo vim portsentry
    TCP_MODE="atcp"
    UDP_MODE="audp"
sudo vim /etc/portsentry/portsentry.conf
    BLOCK_UDP="1"
    BLOCK_TCP="1"
sudo service portsentry restart

https://sharadchhetri.com/2013/06/15/how-to-protect-from-port-scanning-and-smurf-attack-in-linux-server-by-iptables/
sudo vim /etc/network/if-pre-up.d/iptables
# Protecting portscans
# Attacking IP will be locked for 24 hours (3600 x 24 = 86400 Seconds)
iptables -A INPUT -m recent --name portscan --rcheck --seconds 86400 -j DROP
iptables -A FORWARD -m recent --name portscan --rcheck --seconds 86400 -j DROP

# Remove attacking IP after 24 hours
iptables -A INPUT -m recent --name portscan --remove
iptables -A FORWARD -m recent --name portscan --remove

# These rules add scanners to the portscan list, and log the attempt.
iptables -A INPUT -p tcp -m tcp --dport 139 -m recent --name portscan --set -j LOG --log-prefix "portscan:"
iptables -A INPUT -p tcp -m tcp --dport 139 -m recent --name portscan --set -j DROP
iptables -A FORWARD -p tcp -m tcp --dport 139 -m recent --name portscan --set -j LOG --log-prefix "portscan:"
iptables -A FORWARD -p tcp -m tcp --dport 139 -m recent --name portscan --set -j DROP
# to check
sudo bash /etc/network/if-pre-up.d/iptables
sudo service netfilter-persistent reload
sudo iptables -L
# To deleting your IP address from the denied hosts file
sudo vim /etc/hosts.deny
# we need to delete our ban from the iptables
sudo iptables -D INPUT 1
# then reboot

## Stop the services you don’t need for this project.
# Check running services
sudo systemctl list-unit-files --state=enabled
apache2.service
autovt@.service #Necessary for using virtual terminals
console-setup.service #Configuration for the console
cron.service #Scheduled tasks
fail2ban.service #Protection against DOS
getty@.service #Necessary for login
keyboard-setup.service #Configuration for the keyboard
netfilter-persistent.service
networking.service #Network
rsyslog.service 
ssh.service #Needed for SSH connection
sshd.service #Needed for SSH connection
syslog.service  
systemd-timesyncd.service    
remote-fs.target             
apt-daily-upgrade.timer      
apt-daily.timer              

# Disable the rest :
sudo systemctl disable service_name

::::::::::::
:: script ::
::::::::::::

## Create a script that updates all the sources of package, 
## then your packages and which logs the whole in a file named /var/log/update_script.log. 
## Create a scheduled task for this script once a week at 4AM and every time the machine reboots.
su
touch autoupdate.sh
sudo vim autoupdate.sh

# write the updating script AND to log the whole updating process in the logfile

#!/bin/bash

date >> /var/log/update_script.log                    # date of update
apt-get -y -q update >> /var/log/update_script.log
apt-get -y -q upgrade >> /var/log/update_script.log
echo "\n" >> /var/log/update_script.log               # \n between each update

crontab -e
# At the end of the file, to set the update at 4 am every week :

# minute hour dayofmonth month dayofweek command
# Update every week at 4 am
0 4 * * 1 /bin/sh /root/autoupdate.sh
# Update at every reboot
@reboot /bin/sh /root/autoupdate.sh

## Make a script to monitor changes of the /etc/crontab file and sends an email to root if it has been modified. 
## Create a scheduled script task every day at midnight.

apt-get install mailutils
touch cronchanges.sh
vim /etc/aliases
# Make sure all the mails go to root and don't get redirected :
mailer-daemon: postmaster
postmaster: root
nobody: root
hostmaster: root
usenet: root
news: root
webmaster: root
www: root
ftp: root
abuse: root
noc: root
security: root
root: root

cd
sudo vim cronchanges.sh
#!/bin/sh

diff /etc/crontab /etc/current > /dev/null 2> /dev/null
if [ $? -ne 0  ]
then
	echo "The crontab file has been modified\n" | mail -s "Crontab" root@localhost
	cp /etc/crontab /etc/current
fi
# Then reboot and relog as root
# To add the script to the crontab :

crontab -e

0 0 * * * /bin/sh /root/cronchanges.sh



::::::::::::
:: web    ::
::::::::::::
## You have to set a web server who should BE available on the VM’s IP or an host (init.login.com for exemple). 
## About the packages of your web server, you can choose between Nginx and Apache. 
## You have to set a self-signed SSL on all of your services.
https://www.digitalocean.com/community/tutorials/how-to-create-a-ssl-certificate-on-apache-for-debian-8
https://www.linode.com/docs/security/ssl/ssl-apache2-debian-ubuntu/
# First, enable the SSL module of apache2 then reload the service
sudo a2enmod ssl
sudo service apache2 start
# Create a Self-Signed SSL Certificate
# need to create a directory where we'll put our certificate and private key.
sudo mkdir /etc/apache2/ssl
# request a new certificate and sign it
# first generate the certificate and private key
# -days number of days the certificate will be valid 
# -keyout path to our generated key, here /etc/apache2/ssl/apache.key 
# -out path to our generated certificate, here /etc/apache2/ssl/apache.crt
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt
# Set the file permissions to protect your private key and certificate.
sudo chmod 600 /etc/apache2/ssl/*
# create own website configuration file
cd /etc/apache2/sites-available/
sudo touch roger.conf
# create the folder for html pages
cd /var/www/html/    # Go to folder where all available websites html folders are
sudo mkdir roger      # Create a folder for our website
cd roger
sudo mkdir html
cd html
sudo touch index.html # Create homepage file

# disable the default websites and enable ours to make sure it is the only enabled website 
# (there should be 3 available websites, 000-default, default-ssl and for_rs.
sudo a2ensite roger.conf  # Enable our website configuration file
sudo a2dissite            # Disable first default website
  000-default
sudo a2dissite            # Disable second default website
  default-ssl
sudo systemctl reload apache2 # Reload apache service

cd /var/www/html/roger
sudo mkdir log
cd log
sudo touch error.log
sudo touch access.log
sudo vim /etc/apache2/sites-available/roger.conf
    <VirtualHost *:80>
        ServerName 10.13.42.42
        DocumentRoot /var/www/html/roger/html
        Redirect permanent /secure https://10.13.42.42
    </VirtualHost>
    <VirtualHost *:443>
            SSLEngine On
            SSLCertificateFile /etc/apache2/ssl/apache.crt
            SSLCertificateKeyFile /etc/apache2/ssl/apache.key
            ServerAdmin lpan@student.42.fr
            ServerName 10.13.42.42
            DocumentRoot /var/www/html/roger/html
            ErrorLog /var/www/html/roger/log/error.log
            CustomLog /var/www/html/roger/log/access.log combined
    </VirtualHost>
sudo systemctl reload apache2


::::::::::::
:: deploy ::
::::::::::::
https://medium.com/@francoisromain/vps-deploy-with-git-fea605f1303b
sudo mkdir -p /srv/tmp/
sudo chgrp -R roger /srv/tmp/
sudo chmod g+w /srv/tmp/

# Create an empty Git repo
sudo mkdir -p /srv/git/roger.git
# Init the repo as an empty git repository
cd /srv/git/roger.git
sudo git init --bare

# Set the permissions on the Git repo so that we can modify its content without sudo
# Define group recursively to "users"(roger), on the directories
sudo chgrp -R roger .
# Define permissions recursively, on the sub-directories 
# g = group, + add rights, r = read, w = write, X = directories only
# . = curent directory as a reference
sudo chmod -R g+rwX .
# Sets the setgid bit on all the directories
# https://www.gnu.org/software/coreutils/manual/html_node/Directory-Setuid-and-Setgid.html
sudo find . -type d -exec chmod g+s '{}' +
# Make the directory a Git shared repo
sudo git config core.sharedRepository group
# need to change the permissions for all the related directory 

# Write a Git hook to deploy the code
# Create the Git hook file
cd /srv/git/roger.git/hooks
# create a post-receive file
sudo touch post-receive
# make it executable 
sudo chmod +x post-receive
# Edit the /srv/git/roger.git/hooks/post-receive file content:
    #!/bin/sh
    # The production directory
    TARGET="/var/www/html/roger/html"
    # A temporary directory for deployment
    TEMP="/srv/tmp/roger"
    # The Git repo
    REPO="/srv/git/roger.git"
    # Deploy the content to the temporary directory
    mkdir -p $TEMP
    git --work-tree=$TEMP --git-dir=$REPO checkout -f
    cd $TEMP
    # Do stuffs, like npm install…
    # Replace the production directory
    # with the temporary directory
    cd /
    rm -rf $TARGET
    mv $TEMP $TARGET


# to the local computer
# Initialize git repo
git init
# Add the server repo as a remote called "deploy"
git remote add deploy ssh://roger@10.13.42.42:2222/srv/git/roger.git/
# to View current remotes
git remote -v
# Remove remote
git remote rm <destination>

# Push to the server (and deploy)
git add . 
git commit -m "<message>"
git push deploy master


# FINAL:
turn VM off and dont turn it on again. 
go to the path of the VM in sgoinfre
shasum < roger.vdi > RS_result
make a copy of roger.vdi and keep it save.
remove the VM machine.

for each correction:
copy the roger.vdi to goinfre and check shasum
in VM, create a new machine 
choose the existing virtual hard disk file
and add the copied roger.vdi in goinfre to start the correction 
dont forget to set the network as Bridged adaptor

