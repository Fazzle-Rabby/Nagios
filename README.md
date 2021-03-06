# Nagios


*** Prerequisites

- 2 Ubuntu 20.04 servers
- Nagios server - hostname: nagios with an IP: 192.168.1.6
- Ubuntu client - hostname: client with an IP: 10.0.2.15
- Root privileges

*** Agenda

- Install Packages Dependencies
- Install Nagios Core 4.4.6
- Install Nagios Plugin and NRPE Plugin
- Add Host to Monitor to Nagios Server
- Testing

*** Step 1 - Install Packages Dependencies

- update the Ubuntu repository and install some packages dependencies for the Nagios installation.
$ sudo apt update

- install packages dependencies for Nagios installation
$ sudo apt install -y autoconf bc gawk dc build-essential gcc libc6 make wget unzip apache2 php libapache2-mod-php libgd-dev libmcrypt-dev make libssl-dev snmp libnet-snmp-perl gettext

*** Step 2 - Install Nagios Core 4.4.6

- Go to your home directory and download the Nagios Core source code.
$ cd ~/
$ wget https://github.com/NagiosEnterprises/nagioscore/archive/nagios-4.4.6.tar.gz

- Extract the Nagios package and go to the extracted Nagios directory.
$ tar -xf nagios-4.4.6.tar.gz
$ cd nagioscore-*/

- compile Nagios source code and define the Apache virtual host configuration for Nagios
$ sudo ./configure --with-httpd-conf=/etc/apache2/sites-enabled
$ sudo make all

- Create the Nagios user and group, and add the 'www-data' Apache user to the 'nagios' group.
$ sudo make install-groups-users
$ sudo usermod -a -G nagios www-data

- Install Nagios binaries, service daemon script, and the command mode.
$ sudo make install
$ sudo make install-daemoninit
$ sudo make install-commandmode

- install the sample script configuration
$ sudo make install-config

- install the Apache configuration for Nagios and activate the mod_rewrite and mode_cgi modules
$ sudo make install-webconf
$ sudo a2enmod rewrite cgi

- restart the Apache service
$ systemctl restart apache2

- Install the Exfoliation theme for the Nagios web interface.
$ sudo make install-exfoliation

- If you want to use classic Nagios theme, run:
$ sudo make install-classicui

- Create a new apache basic authentication for the user the "nagiosadmin"
$ sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin
New password: 
Re-type new password: 
Adding password for user: nagiosadmin

- Add the SSH and Apache HTTP port using the ufw command below
$ for svc in Apache ssh
$ do
$ ufw allow $svc
$ done

- OR,
- if you have ufw firewall, allow http and https ports for inbound traffic.
$ for i in http https ssh; do sudo ufw allow $i; done

- start the UFW firewall service and add it to the system boot
$ ufw enable

- check all available rules using the command below
$ ufw status numbered

*** Step 3 - Install Nagios Plugins and NRPE Plugin

- Both Nagios and NRPE plugins are available by default on the Ubuntu repository. You can install those packages using the apt command below
$ sudo apt install monitoring-plugins nagios-nrpe-plugin

- Once the installation is complete, go to the nagios installation directory "/usr/local/nagios/etc" and create a new directory for storing all server hosts configuration
$ cd /usr/local/nagios/etc
$ mkdir -p /usr/local/nagios/etc/servers

- edit the Nagios configuration 'nagios.cfg' using nano editor
$ sudo nano nagios.cfg
// Uncomment the 'cfg_dir' option that will be used for sotring all server hots configurations
cfg_dir=/usr/local/nagios/etc/servers
Save and close.

- edit the configuration file "resource.cfg" and define the path binary files of Nagios Monitoring Plugins
$ sudo nano resource.cfg
// Define the Nagios Monitoring Plugins path by changing the default configuration as below
$USER1$=/usr/lib/nagios/plugins
Save and close.

- add the nagios admin email contacts by editing the configuration file "objects/contacts.cfg"
$ sudo nano objects/contacts.cfg
// Change the email address with your own
define contact{
        ......
        email             email@host.com
}
Save and close.

- define the nrpe check command by editing the configuration file "objects/commands.cfg"
$ sudo nano objects/commands.cfg
// did the following configuration to the end of the line
define command{
        command_name check_nrpe
        command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}
Save and close.

- start the Nagios service and add it to the system boot
$ systemctl start nagios
$ systemctl enable nagios

- The Nagios service is up and running, check using the following command
$ systemctl status nagios

- Nagios service is up and running. Now we need to restart the Apache service to apply a new Nagios configuration
$ systemctl restart apache2

- Open your web browser and type the server IP address following the "nagios" URL path.
http://localhost/nagios/

- Log in with the user "nagiosadmin" and type your password

*** Step 4 - Add Linux Host to Monitor

- In this step, we will add the Ubuntu server with hostname "client" and the IP address "10.0.2.15" to the Nagios server
- Log in to the "client" server using your ssh (host and client connected on SSH link)
$ ssh root@10.0.2.15

- Once you've logged in, update the Ubuntu repository and install Nagios Plugins and NRPE Server
$ sudo apt update
$ sudo apt install nagios-nrpe-server monitoring-plugins

- go to the NRPE installation directory "/etc/nagios" and edit the configuration file "nrpe.cfg"
$ cd /etc/nagios/
$ sudo nano nrpe.cfg
// Uncomment the "server_address" line and change the value with the "client" IP address
server_address=10.0.2.15
// One the "allowed_hosts" line, add the Nagios Server IP address "192.168.1.6"
allowed_hosts=127.0.0.1,::1,192.168.1.6
// Save and close.

- Edit the "nrpe_local.cfg" configuration
$ sudo nano nrpe_local.cfg
// Change the IP address with the "client" IP address, and paste the configuration into it
command[check_root]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /
command[check_ping]=/usr/lib/nagios/plugins/check_ping -H 10.0.2.15 -w 100.0,20% -c 500.0,60% -p 5
command[check_ssh]=/usr/lib/nagios/plugins/check_ssh -4 10.0.2.15
command[check_http]=/usr/lib/nagios/plugins/check_http -I 10.0.2.15
command[check_apt]=/usr/lib/nagios/plugins/check_apt
- Save and close.

- Restart the NRPE service and add it to the system boot
$ systemctl restart nagios-nrpe-server
$ systemctl enable nagios-nrpe-server

- Check the NRPE service using the following command
$ systemctl status nagios-nrpe-server

- Back to the Nagios Server and check the "client" NRPE server
$ /usr/lib/nagios/plugins/check_nrpe -H 10.0.2.15
$ /usr/lib/nagios/plugins/check_nrpe -H 10.0.2.15 -c check_ping
// successfully installed the Nagios NRPE Server and Nagios Plugins on the "client" host.

- Add Hosts Configuration to the Nagios Server
- Back to the Nagios server terminal, go to the "/usr/local/nagios/etc" directory and create a new configuration "server/client.cfg"
$ cd /usr/local/nagios/etc
$ sudo nano servers/client.cfg
// Change the IP address and the hostname with your own and paste the configuration into it.
Ubuntu Host configuration file1

define host {
        use                          linux-server
        host_name                    client
        alias                        Ubuntu Host
        address                      10.0.2.15
        register                     1
}

define service {
      host_name                       client
      service_description             PING
      check_command                   check_nrpe!check_ping
      max_check_attempts              2
      check_interval                  2
      retry_interval                  2
      check_period                    24x7
      check_freshness                 1
      contact_groups                  admins
      notification_interval           2
      notification_period             24x7
      notifications_enabled           1
      register                        1
}

define service {
      host_name                       client
      service_description             Check Users
      check_command                   check_nrpe!check_users
      max_check_attempts              2
      check_interval                  2
      retry_interval                  2
      check_period                    24x7
      check_freshness                 1
      contact_groups                  admins
      notification_interval           2
      notification_period             24x7
      notifications_enabled           1
      register                        1
}

define service {
      host_name                       client
      service_description             Check SSH
      check_command                   check_nrpe!check_ssh
      max_check_attempts              2
      check_interval                  2
      retry_interval                  2
      check_period                    24x7
      check_freshness                 1
      contact_groups                  admins
      notification_interval           2
      notification_period             24x7
      notifications_enabled           1
      register                        1
}

define service {
      host_name                       client
      service_description             Check Root / Disk
      check_command                   check_nrpe!check_root
      max_check_attempts              2
      check_interval                  2
      retry_interval                  2
      check_period                    24x7
      check_freshness                 1
      contact_groups                  admins
      notification_interval           2
      notification_period             24x7
      notifications_enabled           1
      register                        1
}

define service {
      host_name                       client
      service_description             Check APT Update
      check_command                   check_nrpe!check_apt
      max_check_attempts              2
      check_interval                  2
      retry_interval                  2
      check_period                    24x7
      check_freshness                 1
      contact_groups                  admins
      notification_interval           2
      notification_period             24x7
      notifications_enabled           1
      register                        1
}

define service {
      host_name                       client
      service_description             Check HTTP
      check_command                   check_nrpe!check_http
      max_check_attempts              2
      check_interval                  2
      retry_interval                  2
      check_period                    24x7
      check_freshness                 1
      contact_groups                  admins
      notification_interval           2
      notification_period             24x7
      notifications_enabled           1
      register                        1
}

- Save and close.

- Now restart the Nagios Server
$ systemctl restart nagios

*** Step 5 - Testing

- Back to your browser and wait for some minutes

- Click on the "Hosts" menu and you will get the "client" has been added
- added Host to monitor to the Nagios Server

*** the installation of Nagios 4.4.6 on Ubuntu 20.04 Server has been completed successfully.
