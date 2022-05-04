# Illumio Core Letsencrypt Certificated

## Simplified step by step LetsEncrypt certificate usage with Illumio Core

```
Author: John Westerman, Illumio, Inc.
Serial number for this document is 20220503232126;
Version 2022.5
Tuesday May 03, 2022 23:21

Things I changed:
1. Nothing. This is my first version of this document
```

## Background and prerequisites

Illumio Core requires the use of TLS for secure communication between the PCE and each VEN enabled workload as well
as any other tool that interfaces Core. Any type of certificate that is valid can be used including self-signed
certificates. However, anything that is not trusted by a CA in a trusted domain (public CA, for example) will throw
all kinds of errors at the user of the system stating that, for any number of reasons, the certificate is invalid. Using
a valid and trusted certificate is important.

This document will demystify at least one method of setting up and using the "free" public CA by LetsEncrypt (https://letsencrypt.org).

To use LetsEncrypt certificates there is one very important requirement: Your PCE must be internet facing. This can be either
on the actual internet with a public IP address (AWS) or inside your lab where you are forwarding port 80 (http) calls back to your
PCE in a DMZ of some sort. This entire process will be automated. There is a callback from their servers to validate the existance
of the IP address and the URL being used. If that process can not be completed you will not be issued a certificate by their CA.

You will need an A record in DNS that can be resolved to your host. That URL will be used in this process and must be defined before you
do anything else.

So, if you can not provide a Internet facing PCE or create an A record in public DNS you do not need to read further. If you can, read on!

## Scenario Used

In the following text I will show you how I use LetsEncrypt to generate a certificate for a SNC PCE in AWS.

If you have access to Elastic
IPs that makes this easier but not a requirement. If you don't have a static IP address just remember you may have to change your A
record if you IP address changes.

If your PCE is like mine, it is rarely rebooted. When it is rebooted it typically gets the same IP
address. Certificates don't care about a relationship with IP address so if it does change all you ahve to do is update your A-record
and the rest will catch up over a short period of time once your TTL expires for your DNS record.

## Assumptions

I will not cover the installation of the PCE software. I will assume you have done a full installation of the PCE using the --generate-cert
option during the installation process. We will replace that certificate in this process with LetsEncrypt certificates and then set you
up to automate that process every 90 days.

For information on how to install the PCE you can follow this link:

https://github.com/johnwesterman/illumio_core

As of this writing I personally use CentOS 8 basic (minimal) images in AWS. It is worth noting that I will use one of the following
image types:

1. M5.xlarge 6 vCPU / 8GB RAM (optimal)
1. t3.X-Large (minimal)
1. I typically use the cheapest 4cpu/8gb instance
1. Note: Minimum specs require 6 CPUs / 8GB RAM. You can go lower and silence the errors reported by not having minimal specs for a small SNC setup.
To silence the errors in a sub-optimal installation you can use "node_under_specs_notification_enabled: false" optin in the runtime yaml file.

Once I have the AWS instance set up I will download all the software required
to set up a full PCE with VEN repo. That software includes:

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

OK, I know this is going to be brief and Apache is an animal all of its own. But to keep this short I am going to over-simplify the installation of Apache on CentOS.

This is what I do:

1. Install Apache ("yum install httpd")
1. Create your website in /etc/httpd.
      1. I create two directories
      1. /etc/httpd/sites-available - where I put my web site config file.
      1. /etc/httpd/sites-enabled - where I link my active web sites I want to expose to the Internet.

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

This creates the virtual host config needed to activate my web site using Apache. The file name can be anything you want
as long as it ends in .conf which is called by the Apache configuration to include any CONF files in "sites-enabled". This
is "standard" Apache setup. Anything in "sites-enabled" is simply a soft link to anything in "sites-available" where the
actual config files are. The idea is you can link a file (ln -s) and if you change the contents of the file in "sites-available"
those changes will be in the linked sites in "sites-enabled". You only need to change the contents in sites-available. There
is a longer lesson here that I am not going to get into. You probably get the gist of this setup by now.

On my PCE I only run Apache during the certificate generation process and then turn it off when I am done.

You can choose to leave port 80 always open or you can choose to create firewall policy that opens port 80 when needed. That is up to you.

For this example I leave port 80 open to the world and only enable Apache to get the certificate and then disable Apache when I am done.

Once you have APACHE installed and you can get to a valid test page you are ready to complete this process.

## LetsEncrypt

LetsEncrypt is a public CA widely used to provide free certificates to anyone who may need one. The entire process it automated so you must follow their
process to get the certificate. If you want to read more about their process you can find it here: https://letsencrypt.org/getting-started/

I'm working to define a very specific use case:

1. CentOS 8 minimal server
1. PCE Core installed and running
1. Apache web server installed and running
1. Certbot installed and ready to be used
1. "Hand jammed" or automated process for renewal

To get to the meat of what I am going to describe to you based on the steps above you can go directly to their web site with these details selected here:

https://certbot.eff.org/instructions?ws=apache&os=centosrhel8

## Once again the reminder to use LetsEncrypt certificates

To use certbot, you'll need...
1. A computer connected to the Internet
1. A HTTP website that is already online that can be addressed with a valid URL (Fully configured Apache)
1. With an open TCP port 80 bound to Apache
1. That is hosted on a server which you can access via ssh with the ability to "sudo" (super user access)

## The basic flow of the process

The basic flow of the certificate creation and renewal process is this:

* SSH into the server
* Install snapd and ensure that your version of snapd is up to date

```
sudo snap install core; sudo snap refresh core
```

* Remove certbot-auto and any Certbot OS packages. If you have any Certbot packages installed using an OS package manager like apt, dnf, or yum, you should remove them before installing the Certbot snap to ensure that when you run the command certbot the snap is used rather than the installation from your OS package manager. The exact command to do this depends on your OS, but common examples are sudo apt-get remove certbot, sudo dnf remove certbot, or sudo yum remove certbot. If you previously used Certbot through the certbot-auto script, you should also remove its installation by following the instructions here.

* Install Certbot

```
sudo snap install --classic certbot
```

* Prepare the Certbot command for use

```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

* Get and install your certificates...

```
sudo certbot --apache
```

* Test automatic renewal. The Certbot packages on your system come with a cron job or systemd timer that will renew your certificates automatically before they expire. You will not need to run Certbot again, unless you change your configuration. You can test automatic renewal for your certificates by running this command:

```
sudo certbot renew --dry-run
```

* And finally we will need to automate the process using a cronjob so you don't have to think about renewing the certiricate every 90 days.

## Give me the BEEF!

OK, you made it this far. That was the hardest part. But let's just get down to the basics now.

If you did all of the above successfully you likely generated a certificate in the process. LetsEncrypt puts all of the files in a directory
/etc/letsencrypt/live/[your-site-name-here]. In that directory there will be 4 key files:

1. cert.pem - only the server certficate (I do not use this)
1. chain.pem - the server and first trust (I do not use this)
1. fullchain.pem - the full certificate chain
1. privkey.pem - the private key

And a README file.

The two files of concern are the fullchain.pem and privkey.pem.

Inside the fullchain.pem file we are interested in the first two certificates in that file. You can either take them out by hand or use a tool I created to extract the certificates individually and rebuild the certificate file on your own by hand. If you want to play with my tool to do this you can find it here:

https://github.com/johnwesterman/certificates

You can "git clone" that repo to your server and just run the program to extract the individual certificates from the certificate file (fullchain.pem). cert1 and cert2 will need to be combined to create server.crt in /var/lib/illumio-pce/cert/ (/var/lib/illumio-pce/cert/server.crt). And the privkey.pem file will be copied over /var/lib/illumio-pce/cert/server.key.

Once you have created these files you can check your environment using:

```
sudo -u ilo-pce illumio-pce-env check
```

You should get an "OK" back from this command using your new certificate. If you have messed something up, don't have the proper file permissions, etc. fix them and rerun the check. Once you get a clean environment check you can restart the PCE which will use this new certificate.

If you do this by hand every 30-60 days you will always have a good certificate on your PCE

## Automation

This section is TBD. Once I have fully automated my own system I'll put my process and my code in here. For now, I am doing mine semi-automated. But the flow is basically this:

1. Start Apache (systemctl start httpd)
1. Check for a new certificate (sudo certbot --apache)
1. Tear apart the fullchain.pem file to individual certs
1. Combine cert1 and cert2 into server.crt and copy this combined certificate into /var/lib/illumio-pce/cert/
1. Copy privkey.pem over server.key in /var/lib/illumio-pce/cert/
1. Stop Apache (systemctl stop httpd)

That is it.

I will work to simplify this document further.