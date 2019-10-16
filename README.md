# Roger-Skyline


## PARTIE 1 : Installation

I recommand using the script "CreationVm" to install the Vm, if you want to understand whats happening you just need to read the script, it's very simple.

During the installation :

* 4.501gb partition for /
* 1gb for /swap
* rest for /home

First we need to install our tools :

```
apt install -y neovim sudo ufw fail2ban sendmail apache2 git portsentry openssl
```

Then we need to create a new user with a specific group, a specified shell, and a home :

```
sudo useradd -g sudo -s /bin/bash -m username
```

or you can use the Script CreateUser.

Then we need to change our network interface to static in the files  ```nvim /etc/network/interfaces``` :

to know your gateway : ```traceroute google.com```

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
iface enp0s3 inet static
	address 10.11.19.93/30
	gateway 10.11.254.254
```

You may now reboot and loggued as the new user.(we restart the networking-services in the same time)

## PARTIE 2 : SSH

For the ssh part we have certain rules to respect ```sudo nvim /etc/ssh/sshd_config``` :

* First we need to change the default port, ``` port 2222 ```
* Then prevent root login : ``` PermitRootlogin no ```

In the host terminal generate ssh key:

```
ssh-keygen
```

then copy the key to your machine serveur : 

```
ssh-copy-id -p 2222 -i ~/.ssh/id_rsa.pub username@localhost
```

Finally we can disable the access with password,    ``` PasswordAuthentification no ```

restart ssh service : 

```
sudo service ssh restart
```

Now try logging into the machine, with:

```
ssh -p 2222 username@localhost
```

## PARTIE 3 : Firewall

For the firewall part, We will use UFW.

```
sudo ufw status
sudo ufw enable
```

Setup firewall rules : 
* SSH ```sudo ufw limit 2223```
* http ```sudo ufw allow 80/tcp```
* https ``` sudo ufw allow 443```


## PARTIE 4 : DOS


The fail2ban package have protection against minor attacks. 
You can activate them by creating a configuration files ```sudo vim /etc/fail2ban/jail.local```.

Then add the following rules :

```
[sshd]
port = 2222
enabled = true
maxretry = 5
findtime = 60
bantime = 60

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

then create a file for the DOS attacks ``` sudo vim /etc/fail2ban/filter.d/http-get-dos.conf``` :

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

edit the file ``` /etc/default/portsentry ``` :

```
TCP_MODE="atcp"
UDP_MODE="audp"
```

also edit the file ``` /etc/portsentry/portsentry.conf ``` :

```
BLOCK_UDP="1"
BLOCK_TCP="1"

comment the current KILL_ROUTE and uncomment the following :
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"

#KILL_HOSTS_DENY="ALL: $TARGET$ : DENY
```

Finaly restart the services with :

``` sudo service portsentry restart```

## PARTIE 6 : Useless Service

Command that list the services.

```
systemctl list-unit-files
```

The command to disable the useless service.

```
systemctl disable <services inutiles>
```
```
sudo systemctl disable console-setup.service
sudo systemctl disable keyboard-setup.service
sudo systemctl disable apt-daily.timer
sudo systemctl disable apt-daily-upgrade.timer
```

## PARTIE 7 : Script update

Create the script ```sudo nvim /home/$username/update.sh ``` :

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

printf "\33[92mcleaning done.\n\33[93mYou can see the log in /var/log/update_script.log\n\33[0m"

## tail -f /var/log/update_script.log
```

Make it executable :

```
chmod +x update.sh
```

Then add the following to /etc/crontab

```
0 4	* * 1	root	/home/$username/update.sh
@reboot		root	/home/$username/update.sh
```

you can check if cron as work properly with 

```
grep CRON /var/log/syslog
```

## PARTIE 8 : Monitor script

Create a copy of your crontab :

```
cp /etc/crontab /home/$username/tmp
```

Create the template for your mail:

```
vim /home/$username/email.txt
```

Then create the script ``` sudo nvim /home/USER/watch_script.sh ``` :

```
#!/bin/bash
cat /etc/crontab > /home/$username/new
DIFF=$(diff new tmp)
if [ "$DIFF" != "" ]; then
	sudo sendmail ROOT@MAIL.com < /home/USER/email.txt
	rm -rf /home/$username/tmp
	cp /home/$username/new /home/$username/tmp
fi
```

Make it executable :

```
chmod +x watch_script.sh
```

Then add the following rule to crontab ``` sudo nvim /etc/crontab ``` :

```
0  0	* * *	root	/home/USER/watch_script.sh
```


## PARTIE 9 : SSL

Generate new SSL key :

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/roger-skyline.com.key -out /etc/ssl/certs/roger-skyline.com.crt
```
Add the following request. most important server name : Server_IP_address

Then we need to configure apache to use ssl

Create a new snippet in the ```/etc/apache2/conf-available``` directory.
We will name the file ```ssl-params.conf``` to make its purpose clear ``` sudo nvim /etc/apache2/conf-available/ssl-params.conf ``` :

```
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLHonorCipherOrder On
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
# Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
# Requires Apache >= 2.4.11
SSLSessionTickets Off
```

let's back up the original SSL Virtual Host file :
```
sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default-ssl.conf.bak
```

Next, let's modify ``` sudo nvim /etc/apache2/sites-available/default-ssl.conf ```

After making these changes, your server block should look similar to this:

```
<IfModule mod_ssl.c>
        <VirtualHost _default_:443>
                ServerAdmin your_email@example.com
                ServerName 10.12.19.93

                DocumentRoot /home/$username/web-app

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

                SSLEngine on

                SSLCertificateFile      /etc/ssl/certs/roger-skyline.com.crt
                SSLCertificateKeyFile /etc/ssl/private/roger-skyline.com.key

                <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                SSLOptions +StdEnvVars
                </FilesMatch>
                <Directory /usr/lib/cgi-bin>
                                SSLOptions +StdEnvVars
                </Directory>

        </VirtualHost>
</IfModule>
```

To adjust the unencrypted Virtual Host file to redirect all traffic to be SSL encrypted, open the ``` sudo nvim /etc/apache2/sites-available/000-default.conf ``` files.

Inside, within the VirtualHost configuration blocks, add a Redirect directive, pointing all traffic to the SSL version of the site:
```
<VirtualHost *:80>
        . . .

        Redirect "/" "https://10.12.19.93/"

        . . .
</VirtualHost>

```


Finaly :

```
sudo a2enmod ssl
sudo a2enmod headers
sudo a2ensite default-ssl
sudo a2enconf ssl-params
sudo apache2ctl configtest
```

Reboot services :

```
sudo systemctl restart apache2
```

The site will be accessible from the IP of your machine. (static IP https://10.12.19.93).


## PARTIE 10 : WEB

For this part you only need a directory with all the file of your web application.

On the server machine create a directory for the Webhooks.

Then ```git init --bare```

it will create files in the directory.

then ```cd hooks```

create a script named ``` post-receive ```

and fill it with :

```
#!/bin/bash
while read oldrev newrev ref
do
    if [[ $ref =~ .*/master$ ]];
    then
        echo "Master ref received.  Deploying master branch to production..."
        git --work-tree=/home/username/web-app --git-dir=/home/username/repo checkout -f
    else
        echo "Ref $ref successfully received.  Doing nothing: only the master branch may be deployed on this server."
    fi
done
```

and don't forget to make it executable ``` chmod +x post-receive ```

We need to modify ``` sudo nvim /etc/apache2/apache2.conf ```

and replace the directory : ``` <Directory /home/username/web-app> ```

also modify ``` sudo nvim /etc/apache2/site-available/000-conf```

then we go on the real repository with the web application.

we initialize it with ``` git init ```

and add the remote name linked to the webhooks ``` git remote add $remotename ssh://username@10.12.19.93:2222/home/username/repo ```

it's done.

now when you just need to push on the remote name you linked before.
```git add .```
``` git commit -m " commit text" ```
``` git push $remotename master ```

it will automaticaly update the server and the files.
