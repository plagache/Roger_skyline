# Roger-Skyline


## PARTIE 1 : Installation

I recommand using my script to install the Vm, if you want to understand whats happening you can read it, but it's very simple.

During the installation

create * 4.501gb partition for /
	   * 1gb for /swap
	   * rest for /home

Do not install firefox or anything useless

```
apt install -y vim sudo net-tools iptables-persistent fail2ban sendmail apache2
```

```
vim /etc/ssh/sshd_config
```

modify port 2222 for ssh.

uncomment ligne :

```
PAsswordAuthentification yes
PermitRootlogin prohibit passwd and replace for no
PubkeyAuthentification yes
```

create a new interfaces : 

```
vim /etc/network/interfaces
```

Add this content :

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp

allow-hotplug enp0s8
iface enp0s8 inet static
address 192.168.56.3
netmask 255.255.255.252
```

create a new user with a home, a specified shell, and a specific group :

```
sudo useradd -g sudo -s /bin/bash -m username
```

reboot

## PARTIE 2 : SSH

In the host terminal generate ssh key:

```
ssh-keygen
```

then copy the key to your machine serveur : 

```
ssh-copy-id -p 2222 -i ~/.ssh/id_rsa.pub username@localhost
```

Now try logging into the machine, with:

```
ssh -p 2222 username@localhost
```
and check to make sure that only the key(s) you wanted were added.

you may need to type your passwd.

Then modify again the sshd_config file :

```
sudo vim /etc/ssh/sshd_config 
```

and pass"PasswordAuthentification yes" to no.

restart ssh service : 

```
sudo service ssh restart
```

You can now exit from the connection and reconnect, this time you won't need to type your passwd.

## PARTIE 3 : Firewall

The firewall parts use iptable rules. you can list the rules with cmd :

```
sudo iptables -L
```

create the files :

```
sudo vim /etc/network/if-pre-up.d/iptables
```

Add the news rules to the files :

```
#!/bin/bash

iptables-restore < /etc/iptables.test.rules

iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

iptables -P INPUT DROP

iptables -P OUTPUT DROP

iptables -P FORWARD DROP

iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

iptables -A INPUT -p tcp -i enp0s8 --dport 2222 -j ACCEPT

iptables -A INPUT -p tcp -i enp0s8 --dport 80 -j ACCEPT

iptables -A INPUT -p tcp -i enp0s8 --dport 443 -j ACCEPT

iptables -A OUTPUT -m conntrack ! --ctstate INVALID -j ACCEPT

iptables -I INPUT -i lo -j ACCEPT

iptables -A INPUT -j LOG

iptables -A FORWARD -j LOG

iptables -I INPUT -p tcp --dport 80 -m connlimit --connlimit-above 10 --connlimit-mask 20 -j DROP

exit 0
```

Make it executable.
```
sudo chmod+x /etc/network/if-pre-up.d/iptables
```

The iptable rules are clean with reboot. the files will allow the system to load them at each reboot. Modify the port in consequences

## PARTIE 4 : DOS

Create a log file for the Apache server.

```
sudo touch /var/log/apache2/server.log
```

The fail2ban package have protection against minor attacks. 
You can activate them by creating a configuration files.

```
sudo vim /etc/fail2ban/jail.local
```

Then add the following rules :

```
[DEFAULT]
destemail = USER@student.le-101.fr
sender = root@roger-skyline.fr

[sshd]
port = 2222
enabled = true
maxretry = 5
findtime = 120
bantime = 60

[sshd-ddos]
port = 2222
enabled = true

[recidive]
enabled = true

[apache]
enabled = true
port = http, https
filter = apache-auth
logpath = /var/log/apache2*/*error.log
maxretry = 6
findtime = 600

[apache-noscript]
enabled = true

[apache-overflows]

enabled  = true
port     = http,https
filter   = apache-overflows
logpath  = /var/log/apache2*/*error.log
maxretry = 2

[apache-badbots]

enabled  = true
port     = http,https
filter   = apache-badbots
logpath  = /var/log/apache2*/*error.log
maxretry = 2

[http-get-dos]
enabled = true
port = http,https
filter = http-get-dos
logpath = /var/log/apache2/server.log
maxretry = 100
findtime = 300
bantime = 300
action = iptables[name=HTTP, port=http, protocol=tcp]
```

then create a file for the DOS attacks :

```
sudo vim /etc/fail2ban/filter.d/http-get-dos.conf
```

And add this content:

```
[Definition]

# Option: failregex
# Note: This regex will match any GET entry in your logs, so basically all valid and not valid entries are a match.
# You should set up in the jail.conf file, the maxretry and findtime carefully in order to avoid false positives.

failregex = ^<HOST> -.*"(GET|POST).*

# Option: ignoreregex
# Notes.: regex to ignore. If this regex matches, the line is ignored.
# Values: TEXT
#
ignoreregex =
```

Then restart the fail2ban service : 

```
sudo systemctl restart fail2ban.service
```

## PARTIE 5 : Port Scan


The command will list the open port of your machine. Make sure only the port your using are open.

```
netstat -paunt
```

Then you will protect the open port from being view in nmap.


## PARTIE 6 : Useless Service

Command that list the services.

```
systemctl list-unit-files
```

The command to disable the useless service.

```
systemctl disable <services inutiles>
```

## PARTIE 7 : Script update

Create the script :

```
vim /home/USER/update.sh
```

```
#! /bin/bash

## update ##

printf "\33[90mupdate running\n"

apt-get update >/var/log/update_script.log.

apt-get -y upgrade >>/var/log/update_script.log.

## add dist-upgrade for distribution upgrade ##

printf "\33[92mupdate finish\n"

## clean ##

printf "\33[90mcleaning start\n"

apt-get autoclean >>/var/log/update_script.log.

apt-get clean >>/var/log/update_script.log.

apt-get -y autoremove >>/var/log/update_script.log.

printf "\33[92mcleaning done.\n\33[93mYou can see the log in /var/log/update_script.log\n"

## tail -f /var/log/update_script.log
```

Make it executable :

```
chmod +x update.sh
```

Then add the following to /etc/crontab

```
0 4	* * 1	root	/home/USER/update.sh  >> /var/log/update_script.log
@reboot		root	/home/USER/update.sh  >> /var/log/update_script.log
```

you can check if cron as work properly with 

```
grep CRON /var/log/syslog
```

## PARTIE 8 : Monitor script

Create a copy of your crontab :

```
cp /etc/crontab /home/USER/tmp
```

Create the template for your mail:

```
vim /home/USER/email.txt
```

Then create the script :

vim /home/USER/watch_script.sh

```
#!/bin/bash
cat /etc/crontab > /home/USER/new
DIFF=$(diff new tmp)
if [ "$DIFF" != "" ]; then
	sudo sendmail ROOT@MAIL.com < /home/USER/email.txt
	rm -rf /home/USER/tmp
	cp /home/USER/new /home/USER/tmp
fi
```

Make it executable :

```
chmod +x watch_script.sh
```

Then add the following rule to crontab :

```
sudo vim /etc/crontab
```

```
0  0	* * *	root	/home/USER/watch_script.sh
```


## PARTIE 9 : WEB

Generate new SSL key :

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/roger-skyline.com.key -out /etc/ssl/certs/roger-skyline.com.crt
```
Add the following request.

Then :

```
sudo vim /etc/apache2/sites-available/default-ssl.conf
```

Then modify only the SSL ligne with the good path for the keys :

```
<IfModule mod_ssl.c>
 <VirtualHost _default_:443>       
                ServerAdmin webmaster@localhost
                DocumentRoot /var/www/html

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                #Include conf-available/serve-cgi-bin.conf

                #   SSL Engine Switch:
                #   Enable/Disable SSL for this virtual host.
                SSLEngine on
                SSLCertificateFile      /etc/ssl/certs/roger-skyline.com.crt
                SSLCertificateKeyFile /etc/ssl/private/roger-skyline.com.key
                #
                #SSLCertificateFile      /etc/ssl/certs/ssl-cert-snakeoil.pem
                #SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

......................
.......................

 </VirtualHost>
</IfModule>
```

Finaly test the cmd :

```
sudo apachectl configtest
sudo a2enmod ssl
sudo a2ensite default-ssl
```

Reboot services :

```
sudo systemctl restart apache2.service
```

Make a copy of your configuration :

```
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/001-default.conf
```

modify the files :

```
sudo vim /etc/apache2/sites-available/001-default.conf
```

Change ServerName with what ever you want and add DocumentRoot to the path of you website.

Activate new configuration :

```
# Deactivate the old one
a2dissite 000-default.conf
# Activate new conf file
a2ensite 001-site.conf
# reload service
systemctl reload apache2
```

The site will be accessible from the IP of your machine. (static IP https://192.168.56.3).

If you didnt change path for the site, you can put the files in /var/www/html.

Maybe sudo chown -R /var/www/html.
