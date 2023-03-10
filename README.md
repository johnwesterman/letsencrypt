# Illumio Core with LetsEncrypt Certificates

## Simplified step by step LetsEncrypt certificate usage with Illumio Core

```
Author: John Westerman, Illumio, Inc.
Serial number for this document is 20230310140225;
Version 2023.3
Friday March 10, 2023 14:02

Things I changed:
1. Cleaning up the text.
```

## Background and prerequisites

Illumio Core requires the use of TLS for secure communication between the PCE and each VEN enabled workload as well as any other tool that interfaces Core. Any type of certificate that is valid can be used including self-signed certificates. However, anything that is not trusted by a CA in a trusted domain (public CA, for example) will throw all kinds of errors at the user of the system stating that, for any number of reasons, the certificate is invalid. Using a valid and trusted certificate is important if you do not want to deal with trust issues and SSL errors.

This document will demystify at least one method of setting up and using the "free" public CA by LetsEncrypt (https://letsencrypt.org).

To use LetsEncrypt certificates there is one very important requirement: Your PCE must be internet facing. This can be either on the actual internet with a public IP address (AWS) or inside your lab where you are forwarding port 80 (http) calls back to your PCE in a DMZ of some sort. This entire process will be automated. There is a callback from their servers to validate the existance of the IP address and the URL being used. If that process can not be completed you will not be issued a certificate by their CA.

You will need an Internet facing DNS A-record that can be resolved to your host. That URL will be used in this process and must be defined before you do anything else.

If you can not provide a Internet facing PCE or create an Interfacing DNS A-record in you do not need to read further. If you can do this and want to use this certificate system then read on!

## Scenario Used

In the following text I will show you how I use LetsEncrypt to generate a certificate for an SNC PCE in AWS.

If you have access to Elastic IPs that makes this easier but not a requirement. If you don't have a static IP address just remember you may have to change your A-record if your IP address changes.

If your PCE is like mine, it is rarely rebooted. When it is rebooted it typically gets the same IP address. Certificates don't care about a relationship with IP address so if it does change all you ahve to do is update your A-record and the rest will catch up over a short period of time once your TTL expires for your DNS record.

## Assumptions

I will not cover the installation of the PCE software. I will assume you have done a full installation of the PCE using the --generate-cert option during the installation process. We will replace that certificate in this process with LetsEncrypt certificates and then set you up to automate that process every 90 days.

For information on how to install the PCE you can follow this link:

https://github.com/johnwesterman/illumio_core

As of this writing I personally use CentOS 8 basic (minimal) images in AWS. It is worth noting that I will use one of the following image types:

1. M5.xlarge 6 vCPU / 8GB RAM (optimal)
1. t3.X-Large (minimal)
1. I typically use the cheapest 4cpu/8gb instance
1. Note: Minimum specs require 6 CPUs / 8GB RAM. You can go lower and silence the errors reported by not having minimal specs for a small SNC setup.To silence the errors in a sub-optimal installation you can use node_under_specs_notification_enabled: false" option in the runtime yaml file.

Once I have the AWS instance set up I will download all the software required to set up a full PCE with VEN repo. That software includes:

1. PCE Core RPM
1. PCE UI RPM
1. VEN Compatibility matrix bz2
1. VEN bundle bz2

So you will:

1. Launch your AWS instance
1. Get any software updates you may need
1. Copy the Illumio software to the /tmp directory on your new instance
1. Install the Illumio software using the --generate-cert option during that process. Note: You can follow my instructions on how to get it working fully with the temporary certificate (https://github.com/johnwesterman/illumio_core)
1. Once these steps are done you will be ready to get a valid public (and free) certificate from LetsEncrypt

Make sure all of the above is working well and you have a fully working SNC and you'll be ready to continue to the next steps in this process.

## Setting up Apache

This is going to be brief and Apache is an animal all of its own. To keep this short and to the point I am going to over-simplify the installation of Apache on CentOS.

This is what I do:

1. Install Apache ("yum install httpd")
1. Create your website in /etc/httpd.
   1. I create two directories
   1. /etc/httpd/sites-available - where I put my web site config file.
   1. /etc/httpd/sites-enabled - where I link my active web sites I want to expose to the Internet.
1. Create the log directory.
   1. mkdir /var/www/log
   1. touch /var/www/log/error.log
   1. touch /var/www/log/requests.log 

To make that last step easy, here is the text you can drop into your system:
```
   mkdir /var/www/log
   touch /var/www/log/error.log
   touch /var/www/log/requests.log 
```

An example of one of my web site config files is:

```
<VirtualHost *:80>
    ServerName wildwest.packetwhisper.com
    ServerAlias wildwest.packetwhisper.com
    DocumentRoot /var/www/html/wildwest.packetwhisper.com/
    ErrorLog /var/www/log/error.log
    CustomLog /var/www/log/requests.log combined
</VirtualHost>
```

The configuration above creates the virtual host config needed to activate my web site using Apache. The file name can be anything you want as long as it ends in .conf which is called by the Apache configuration to include any CONF files in "sites-enabled". This is "standard" Apache setup. Anything in "sites-enabled" is simply a soft link to anything in "sites-available" where the actual config files are. The idea is you can link a file (ln -s) and if you change the contents of the file in "sites-available" those changes will be in the linked sites in "sites-enabled". You only need to change the contents in sites-available. There is a longer lesson here that I am not going to get into. You probably get the gist of this setup by now.

Be sure and name this system configuration "something.conf" (with the .conf extension) and put it in the "sites-available" directory above. And when you are ready to enable it create a symbolic link to that in the "sites-enabled" directory:

```
ln -s /etc/httpd/sites-available/wildwest.conf /etc/httpd/sites-enabled/wildwest.conf
```

Then include this at the end of the /etc/httpd/conf/httpd.conf file:

```
IncludeOptional sites-enabled/*.conf
```

... and restart httpd:

```
systemctl restart httpd
```

There should be no complaints at this point. If there are, sort through those configuration errors. I generally put a small text file in /var/www/html/wildwest.packetwhisper.com/ and name it index.html. Inside that file you can put any text. When you go to your web site you should see that text file show up on your screen. If not, there is a configuration error somewhere. If so, you know you have set everything up properly.

On my PCE I only run Apache during the certificate generation process and then turn it off when I am done. You can choose to leave port 80 always open or you can choose to create firewall policy that opens port 80 when needed. That is up to you.

For this example I leave port 80 open to the world and only enable Apache to get the certificate and then disable Apache when I am done.

Once you have APACHE installed and you can reach your server on port 80 on the Internet you are ready to proceed. If not, figure that out and then proceed.

## LetsEncrypt

LetsEncrypt is a public CA widely used to provide free certificates to anyone who may need one. The entire process it automated so you must follow their process to get the certificate. If you want to read more about their process you can find it here: https://letsencrypt.org/getting-started/

I'm working to define a very specific use case:

1. CentOS Stream 8 minimal server
1. PCE Core installed and running
1. Apache web server installed, running and listening on tcp/80
1. Certbot installed and ready to be used
1. "Hand jammed" or automated process for renewal

To get to the meat of what I am going to describe to you based on the steps above you can go directly to Let's Encrypt web site with these details selected here:

https://certbot.eff.org/instructions?ws=apache&os=centosrhel8

## Reminder on the use of LetsEncrypt certificates

To use certbot, you'll need...
1. A computer connected to the Internet
1. A HTTP website that is already online that can be addressed with a valid URL (Fully configured Apache)
1. With an open TCP port 80 bound to Apache
1. That is hosted on a server which you can access via ssh with the ability to "sudo" (super user access)

## The basic flow of the process

The basic flow of the certificate creation and renewal process is this:

1. SSH into the server
1. Install snapd and ensure that your version of snapd is up to date

```
dnf install snapd; systemctl start snapd
```
```
sudo snap install core; sudo snap refresh core
```
Link the snapd daemon correctly for snap to work properly:
```
sudo ln -s /var/lib/snapd/snap /snap
```

### Remove certbot-auto and any Certbot OS packages.

If you have any legacy Certbot packages installed using an OS package manager like apt, dnf, or yum, you should remove them before installing the Certbot snap image that will follow to ensure that when you run the command certbot the snap is used rather than the installation from your OS package manager. The exact command to do this depends on your OS, but common examples are sudo apt-get remove certbot, sudo dnf remove certbot, or sudo yum remove certbot. If you previously used Certbot through the certbot-auto script, you should also remove its installation by following the instructions here.

### Install Certbot

```
sudo snap install --classic certbot
```

### Prepare the Certbot command for use

```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### Get and install your certificates **with a private key genenerated with type *RSA*.** As of this writing the PCE does not support ECC private key derived certificates.

```
sudo certbot certonly --key-type rsa
```

At this point you should have new certificates loaded on your system that you can use. These keys will be put into a directory structure similar to this: /etc/letsencrypt/live/wildwest.packetwhisper.com/. There is still some configuration of the certificate/key on the PCE which we will do below.

Before going further, another thing I typically do is link the long letsencrypt directory to my home directory (example):

```
ln -s /etc/letsencrypt/live/wildwest.packetwhisper.com/ letsencrypt
```
Doing this I do not have to worry about remembering where all these certs are located.

### Test automatic renewal.

The Certbot packages on your system come with a cron job or systemd timer that will renew your certificates automatically before they expire. You will not need to run Certbot again, unless you change your configuration. You can test automatic renewal for your certificates by running this command:

```
sudo certbot renew --dry-run
```

And finally we will need to automate the process using a cronjob so you don't have to think about renewing the certiricate every 90 days.

## Give me the BEEF!

OK, you made it this far. That was the hardest part. But let's just get down to the basics now.

If you did all of the above successfully you likely generated a certificate in the process. LetsEncrypt puts all of the files in a directory:

/etc/letsencrypt/live/[your-site-name-here]

In that directory there will be 4 key files:

1. cert.pem - only the server certficate (I do not use this)
1. chain.pem - the server and first trust (I do not use this)
1. fullchain.pem - the full certificate chain
1. privkey.pem - the private key
1. a README file

The two files of concern and what will be used on the PCE are the **fullchain.pem** and **privkey.pem**.

Inside the **fullchain.pem** file we are interested in the **only the first two certificates** in that file. You can either take them out by hand or use a tool I created to extract the certificates individually and rebuild the certificate file on your own by hand.

Once you have the certificates from Letsencrypt:

1. Pull the first two certificates from fullchain.pem and put them a file named "server.crt".
1. Copy the privkey.pem to a file named "server.key"

You can go through the process of making sure they are related if you like. But they will be if you did this properly.

On my systems I use the default directory for the certificate and key files. So I copy "server.key" and "server.crt" to /var/lib/illumio-pce/cert/. If necessary make sure the ownership and permissions are set correctly:

```
chmod 400 /var/lib/illumio-pce/cert/server.*
chown ilo-pce: /var/lib/illumio-pce/cert/server.*
```

Once you have created these files correctly created and in their proper places you can check your environment using:

```
sudo -u ilo-pce illumio-pce-env check
```

You should get an "OK" back from this command using your new certificate. If you have messed something up, don't have the proper file permissions, etc. fix them and rerun the check. Once you get a clean environment check you can restart the PCE which will use this new certificate.

If you do this by hand every 30-60 days you will always have a good certificate on your PCE.

## Automation

The flow is basically this:

1. Start Apache (systemctl start httpd)
1. Check for a new certificate (sudo certbot --apache)
1. Tear apart the fullchain.pem file to individual certs
1. Combine cert1 and cert2 into server.crt and copy this combined certificate into /var/lib/illumio-pce/cert/
1. Copy privkey.pem over server.key in /var/lib/illumio-pce/cert/
1. Stop Apache (systemctl stop httpd)

If you just want to check for new certs put this in your crontab:

0 4 * * 1 : Check for new certificate ; /bin/certbot certonly renew

This will check for new certificates every Monday at 4am. You really don't need to do any more checking than once a week. The certficates will not renew until they are about 30 days from expiring. When they do renew make sure you extract the proper certificates and put them in place. Then you will need to restart the PCE to load these new certificates. If I ever automate this I'll put my code in here.

## The end