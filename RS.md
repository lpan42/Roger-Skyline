
# Logged as root, install sudo, git, make, vim, net_tools(for ifconfig)
apt-get install sudo git make vim net-tools
# add user to sudoers
usermod -aG sudo roger
# go back to your non-root user and use sudo when you need root privileges



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
sudo vi /etc/ssh/sshd_config
uncomment Port 22 and replay by 2222
uncomment PasswordAuthentication change to no
Uncomment PermitRootLogin and replace prohibit-password by no
sudo service sshd restart

Client side:
# Generate a publickey to access VM via SSH
ssh-keygen
# Copy the publickey into the VM publickeys file
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
<!-- 
# Reset rules
iptables		-F
iptables		-X
iptables -t nat		-F
iptables -t nat		-X
iptables -t mangle	-F
iptables -t mangle	-X -->

# Drop everything as default behavior
iptables -P INPUT	DROP
iptables -P OUTPUT	DROP
iptables -P FORWARD 	DROP

# Loopback
(The loopback device is a special, virtual network interface 
that your computer uses to communicate with itself. 
It is used mainly for diagnostics and troubleshooting, 
and to connect to servers running on the local machine.
任何送到该接口的网络数据报文都会被认为是送往设备自身的。
大多数平台都支持使用这种接口来模拟真正的接口。)

iptables -A INPUT	-i lo						-j ACCEPT
iptables -A OUTPUT	-o lo						-j ACCEPT

# DNS
iptables -A OUTPUT	-p tcp		--dport	53			-j ACCEPT
iptables -A OUTPUT	-p udp		--dport	53			-j ACCEPT

# SSH
iptables -A INPUT	-p tcp		--dport	2222			-j ACCEPT

# HTTP
iptables -A OUTPUT 	-p tcp		--dport	80			-j ACCEPT

# HTTPS
iptables -A OUTPUT	-p tcp		--dport	443			-j ACCEPT

# Ping(Ping uses the Internet Control Message Protocol (ICMP) Echo function)
iptables -A OUTPUT	-p icmp		--icmp-type echo-request	-j ACCEPT
iptables -A INPUT	-p icmp		--icmp-type echo-reply		-j ACCEPT

# Already established connections
iptables -A INPUT	-m conntrack	--ctstate ESTABLISHED,RELATED	-j ACCEPT
iptables -A OUTPUT	-m conntrack	--ctstate ESTABLISHED,RELATED	-j ACCEPT

# apply
sudo chmod +x /etc/network/if-pre-up.d/iptables
sudo /etc/network/if-pre-up.d/iptables
sudo iptables -L

:::::::::
:: DOS ::
:::::::::

## You have to set a DOS (Denial Of Service Attack) protection on your open ports of your VM.
(ref: https://www.scaleway.com/en/docs/protect-server-fail2ban/
https://www.supinfo.com/articles/single/2660-proteger-votre-vps-apache-avec-fail2ban)
# install
sudo apt install fail2ban

# By default Fail2ban keeps all the configuration files in /etc/fail2ban/ directory. 
# The main configuration file is jail.conf, it contains a set of pre-defined filters. 
# Override it by creating a new configuration file jail.local inside /etc/fail2ban/ directory.
cd /etc/fail2ban
sudo cp jail.conf jail.local
sudo vim jail.local

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

# to check
