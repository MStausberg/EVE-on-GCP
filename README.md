# EVE-on-GCP

A short guide how to install EVE-NG on the Google Cloud Platform.
Based on the description @ [eve-ng.net](https://www.eve-ng.net/index.php/documentation/installation/google-cloud-install/)

---

## Setting up your VM on Google Cloud Platform

Go to https://console.cloud.google.com/getting-started and sign in or create a new account.

At the time of writing, you get $300 to spend in 12 months for signing up (which I used to set up my EVE-NG instance).

Google will have created a default Project for you (aptly named "My-first-project") - you can use this project. or create a new one.

In the top bar, select the project you want to use.

Open the Google Cloud Shell and enter the following comand to create an Ubuntu 16.04 Image with nested virtualization activated:

```bash
gcloud compute images create nested-ubuntu-xenial --source-image-family=ubuntu-1604-lts --source-image-project=ubuntu-os-cloud --licenses https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx
```

Now go on to actually create the VM instance: Navigate to Menu/Compute Engine/VM Instances and click "create".

Edit your settings, you might want to use a region and zone close to your geographical location.

I chose a N2 highmem machine with 16 vCPUs and 128GB RAM.

**IMPORTANT:** "Deploy a Container Image" must be UNCHECKED

Change the boot disk to your previously created image ("Change" -> "Custom Images") and set an appropriate size (I chose 100GB)

Make sure you allow acces through HTTPS (and, if you want to use LetsEncrypt, HTTP for the initial setup).

## Installing EVE CE on the VM

Open a shell to the newly created VM (through the google cloud console or however you like).

Become ```root```:

``` bash
sudo -i
```

Download and run the install script:

```bash
wget -O - https://www.eve-ng.net/repo/install-eve.sh | bash -i
```

Update & Upgrade all Packages:

```bash
apt update && apt upgrade -y
```

Afterwards, ```reboot``` the VM. You will obviously lose connection to the shell, just reconnect after some time (when you think the VM has rebooted).

When you reconnect to the shell, you will be greeted by the IP wizard which lets you set up network connectivity.

**IMPORTANT:** Set the IP to DHCP!

Become ```root``` again:

``` bash
sudo -i
```

Install EVE DOCKERS:

```bash
apt update
apt install eve-ng-dockers
```

When installation is complete, drop root access with ```exit```.

You can now access your instance through the public IP, but it has no certs yet for HTTPS.

We want to enable LetsEncrypt to fix that.

### Setting up LetsEncrypt

Set your locale to en_US:

```bash
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
sudo dpkg-reconfigure locales
```

Hit OK twice and wait for the process to finish.

Install Certbot:

```bash
cd /usr/local/sbin
sudo wget https://dl.eff.org/certbot-auto
sudo chmod a+x /usr/local/sbin/certbot-auto
```

Enable the SSL module and create a new request:

```bash
sudo a2enmod ssl

sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```

Create the default config file:

```bash
cat << EOF > /etc/apache2/sites-enabled/default-ssl.conf
<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
        ServerAdmin webmaster@localhost
        DocumentRoot /opt/unetlab/html/
        ErrorLog /opt/unetlab/data/Logs/ssl-error.log
        CustomLog /opt/unetlab/data/Logs/ssl-access.log combined
        Alias /Exports /opt/unetlab/data/Exports
        Alias /Logs /opt/unetlab/data/Logs
        SSLEngine on
        SSLCertificateFile    /etc/ssl/certs/apache-selfsigned.crt
        SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
                SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
                SSLOptions +StdEnvVars
        </Directory>
        <Location /html5/>
                Order allow,deny
                Allow from all
                ProxyPass http://127.0.0.1:8080/guacamole/ flushpackets=on
                ProxyPassReverse http://127.0.0.1:8080/guacamole/
        </Location>

        <Location /html5/websocket-tunnel>
                Order allow,deny
                Allow from all
                ProxyPass ws://127.0.0.1:8080/guacamole/websocket-tunnel
                ProxyPassReverse ws://127.0.0.1:8080/guacamole/websocket-tunnel
        </Location>
    </VirtualHost>
</IfModule>
EOF

```

Create the LetsEncrypt certificate:

```bash
certbot-auto --apache -d eve.example.com
```

Restart the Apache server:

``` bash
/etc/init.d/apache2 restart
```

Create a cronjob for auto-renewal of the certificate (This can run once a week since the renewal process is only started if the expiration date is in the next 30 or fewer days).

Open the crontab in edit mode:

```bash
crontab -e
```

add the following line (or customise to your preferences):

```bash
37 7 * * 1 /usr/local/sbin/certbot-auto renew >> /var/log/le-renew.log
```

## Set up Apache2 for access via FQDN

Add the Domain:

```bash
sudo mkdir /var/www/eve.example.com
sudo chown -R www-data:www-data /var/www/eve.example.com
```

Create Apache Virtual Host:

```bash
sudo nano /etc/apache2/sites-available/eve.example.com.conf


<VirtualHost *:443>
    ServerAdmin admin@example.com
    ServerName eve.example.com
    ServerAlias www.eve.example.com
    DocumentRoot /opt/unetlab/html/
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/eve.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/eve.example.com/privkey.pem
</VirtualHost>
<Directory /var/www/eve.example.com/>
    AllowOverride All
</Directory>


```

Now we need to activate the new site and reload the config for Apache2:

```bash
sudo a2ensite eve.example.com.conf
sudo service apache2 reload
```

You should now be able to access your EVE-Instance through the public IP or via its FQDN (if you have set the DNS records, of course).

The default credentials are admin/eve, you have to change them manually after first login.
