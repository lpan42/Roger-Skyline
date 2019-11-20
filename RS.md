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
## You have to change the default port of the SSH service by the one of your choice. 
## SSH access HAS TO be done with publickeys. 
## SSH root access SHOULD NOT be allowed directly, but with a user who can be root.
server side:
sudo systemctl status ssh 
(check ssh, if not install) sudo apt install openssh-server
sudo vi /etc/ssh/sshd_config
uncomment Port 22 and replay by 2222
uncomment PasswordAuthentication yes
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

# Installing iptables-persistent to make the rule change permanent
sudo apt install iptables-persistent
# Start the service
sudo service netfilter-persistent start