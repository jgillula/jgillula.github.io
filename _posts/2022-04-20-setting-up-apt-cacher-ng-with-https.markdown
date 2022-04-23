---
layout: post
title:  Setting up apt-cacher-ng with HTTPS
date:   2022-04-20
tags: apt-cacher-ng https apache2
description: "How to get ancient software to support modern security"
---

# Goals

[apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/) is a proxy server that sits between your apt client (i.e. the `apt` command on your computer) and the apt archives from which you usually download packages. As the name implies, it caches those apt packages. I want to use it to keep a local cache of the packages I install, so that if I ever need to roll back (I'm looking at you, changes that dramatically alter the UI without making things actually better) I have a complete archive of the versions I've installed.

To do that, I'm going to run [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/) on my local server. That means my laptop will need to connect to it (instead of the usual apt archives) whenever it wants to check for updates. Sometimes my laptop is not on my home network, and I don't want it to connect to any old arbitrary server claiming to be my local server. Thus, I need to have my local [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/) server only accept connections over HTTPS.

# 0. The Plan

Of course, [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/) doesn't support HTTPS out of the box. To fix this, I'm going to have to set up apache2 on my local server, have it use HTTPS, and have it act as a [reverse proxy](https://oxylabs.io/blog/reverse-proxy-vs-forward-proxy) in front of [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/).

*Throughout these instructions I'll use `servername` as the hostname of my server. You should replace it with your own local server name.*

*All of the work in the following steps is done on my server unless otherwise specified.*

# 1. Getting the HTTPS certificate

Normally this is the step where I'd spin up [Certbot](https://certbot.eff.org/) or some other [Let's Encrypt](https://letsencrypt.org) client. But since my local server is not going to be accessible from the general Internet I can't get a publicly valid cert. Instead I'm going to have to create a local certificate authority, generate a CA root cert, put it on my laptop, and then generate a CA-root-signed cert for my server and make nginx use it.

Creating and signing certs is arcane black magic, so instead of trying to do everything by hand I'm going to use the [mkcert](https://github.com/FiloSottile/mkcert) tool. Since `mkcert` doesn't have a PPA or a standard apt package, you'll have to download a pre-built binary from [https://github.com/FiloSottile/mkcert/releases](https://github.com/FiloSottile/mkcert/releases)
```bash
wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64 -O mkcert # By default wget will validate certificates, so this should be safe
chmod +x mkcert
```
I want to store my CA key and certificate on a removable drive for security, so to do that we set the `CAROOT` environment variable, and then make sure it gets passed to sudo:
```bash
export CAROOT=/some/directory
sudo --preserve-env=CAROOT ./mkcert
```
That created two files in `$CAROOT`, the CA's private key `rootCA-key.pem`, and the CA's root certificate `rootCA.pem`. I'll need to copy the CA root certificate to any device that I want to have recognize my certs, so I'll copy that back to my working directory:
```bash
cp $CAROOT/rootCA.pem .
```

Finally to create the certificate for my server, I just do
```
sudo --preserve-env=CAROOT ./mkcert -cert-file /etc/ssl/certs/apt-cacher-ng.pem -key-file /etc/ssl/private/apt-cacher-ng.key apt-cacher-ng.servername
```
This installs the certificate and private key in locations where apache2 will be able to find them later.

***Note: By default the certs `mkcert` creates are only valid for 27 months, so you'll have to manually repeat this process to create new certs before then. See the bottom of the page for how to do it.***

# 2. Install and configure apt-cacher-ng

```bash
sudo apt install apt-cacher-ng
```
By default [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/) listens on port `3142`, so you can verify it's working by visiting [http://servername:3142/acng-report.html](http://servername:3142/acng-report.html)

# 3. Install and configure apache2

```bash
sudo apt install apache2
```
I'm going to want apache2 to proxy requests from [https://apt-cacher-ng.servername](https://apt-cacher-ng.servername), so I create a new site config as follows and save it in `/etc/apache2/sites-available/apt-cacher-ng-proxy.conf`:
```
<VirtualHost *:443>
  ServerName apt-cacher-ng.servername

  ProxyPass "/"  "http://localhost:3142/"
  ProxyPassReverse "/"  "http://localhost:3142/"

  SSLEngine on
  SSLCertificateFile    /etc/ssl/certs/apt-cacher-ng.pem
  SSLCertificateKeyFile /etc/ssl/private/apt-cacher-ng.key
</VirtualHost>
```

Then to get apache2 to actually enable the proxy, I do:
```bash
sudo a2ensite apt-cacher-ng-proxy.conf
sudo a2enmod ssl
sudo systemctl restart apache2
```

Last but not least, I need machines on my local network to know at what IP address to find [https://apt-cacher-ng.servername](https://apt-cacher-ng.servername). I've already set up avahi to do this for other aliases, so I just add `apt-cacher-ng.servername` to the `/etc/avahi/aliases` file, and restart avahi:
```bash
sudo systemctl restart avahi-alias
```

# 4. Install the root CA certificate on your clients

To install the root CA certificate on your Ubuntu machine, you'll need to first transfer the `rootCA.pem` file we created in step 1 to the client machine. Then do the following **on your client machine**:
```bash
sudo cp rootCA.pem /usr/local/share/ca-certificates/servername.crt # The .crt extension is required. Also note that you can't use a period character anywhere in the filename except in the ".crt" part
sudo update-ca-certificates
```

If you want to test and make sure everything is working, you can also install the root cert in Chrome by visiting [chrome://settings/certificates](chrome://settings/certificates), choosing "Authorities", and then clicking "Import" and following the prompts. Then you can visit [https://apt-cacher-ng.servername/acng-report.html](https://apt-cacher-ng.servername/acng-report.html) and verify that apt-cacher-ng is running, apache2 is proxying the https request to it, and that your browser accepts the certificate as valid (in reverse order, really).

# 5. Next time...

In the next blog post, I'll document how to configure apt on an Ubuntu system to use the new [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/) proxy we created.

# 6. How to renew your certs

As mentioned above, the certs created by `mkcert` are only valid for 27 months (which is probably a good idea). To create a fresh cert for the apt-cacher-ng proxy, just make sure the external drive where you saved the private key for the root CA is plugged in and available at `/some/directory`, and then do:
```bash
wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.3/mkcert-v1.4.3-linux-amd64 -O mkcert # By default wget will validate certificates, so this should be safe
chmod +x mkcert
export CAROOT=/some/directory
sudo --preserve-env=CAROOT ./mkcert -cert-file /etc/ssl/certs/apt-cacher-ng.pem -key-file /etc/ssl/private/apt-cacher-ng.key apt-cacher-ng.servername
sudo systemctl reload apache2
```
