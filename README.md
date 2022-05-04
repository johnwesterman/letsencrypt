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

1 M5.xlarge 6 vCPU / 8GB RAM (optimal)
1 t3.X-Large (minimal)
1 I typically use the cheapest 4cpu/8gb instance
1 Note: Minimum specs require 6 CPUs / 8GB RAM. You can go lower and silence the errors reported by not having minimal specs for a small SNC setup.
To silence the errors in a sub-optimal installation you can use "node_under_specs_notification_enabled: false" optin in the runtime yaml file.

Once I have the AWS instance set up I will download all the software required
to set up a full PCE with VEN repo. That software includes:

1 PCE Core RPM
1 PCE UI RPM
1 VEN Compatibility matrix bz2
1 VEN bundle bz2

So you will:

1 Launch your AWS instance
1 Get any software updates you may need
1 Copy the Illumio software to the /tmp directory on your new instance
1 Install the Illumio software using the --generate-cert option during that process. Note: You can follow my instructions on how to get it working fully with the temporary certificate (https://github.com/johnwesterman/illumio_core)
1 Once these steps are done you will be ready to get a valid public (and free) certificate from LetsEncrypt

Make sure all of the above is working well and you have a fully working SNC and you'll be ready to continue to the next steps in this process.

## LetsEncrypt

LetsEncrypt is a public CA widely used to provide free certificates to anyone who may need one. 