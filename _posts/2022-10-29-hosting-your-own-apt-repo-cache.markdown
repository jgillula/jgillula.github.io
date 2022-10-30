---
layout: post
title:  Hosting Your Own Apt Repo Cache
date:   2022-10-29
tags: backups apt syncthing
description: "Never go looking for an old version of an apt package again."
---

# Motivation

I'm picky about the software running on my machine, but I also want to be able to try new updates. That means sometimes I want to be able to roll back installation or upgrades of `apt` packages. There are good tools that help you do this like [`apt-clone`](https://github.com/mvo5/apt-clone) and [`apt-rollback`](https://gitlab.com/fabio.dellaria/apt-rollback#apt-rollback), but they're only useful if you actually have the older versions of the packages you want to downgrade to. While older versions of most packages can probably be found somewhere on the Internet (and `apt-rollback` even tries to find some of them), finding them can be a chore. Additionally, many third-party repos don't host older versions once newer ones are posted. Thus since storage space is cheap I'd rather just host them myself.

Note that one solution here is to use [`apt-cacher-ng`](https://www.unix-ag.uni-kl.de/~bloch/acng/). I tried [using it at first](/2022/04/setting-up-apt-cacher-ng-with-https/), but it was really complicated to make it intercept HTTPS traffic and it often failed when I would try to do an `apt update`. Instead, I've decided to hack together my own version of it using the absolutely fantastic [Syncthing](https://syncthing.net/) and a little bash scripting glue.

## The plan

The idea is actually pretty simple:
* Every time one of my machines upgrades or installs a new package, it will of course cache the `.deb` file temporarily in `/var/cache/apt/archives/`.
* I'll then use Syncthing to sync that cache to my home server.
* I'll then use a little bash scripting + apache to turn that cache into an honest-to-god apt package repo.
* I then just add that repo to all of my machines, and any time they need an old package, they can get it from the repo.

Obviously there's some subtlety here, so let's dive in.

# Syncing with Syncthing

## On the client

The first thing I did was add `/var/cache/apt/archives/` as a folder to sync in Syncthing on one of my client machines. Since that folder is owned by root I had to temporarily `chmod` it so that the user I run Syncthing as could write to it, just to create the `.stfolder` folder and `.stignore` file. I named the folder to match the release of the client (e.g. bionic, jammy, etc.), since at the moment the client and the server are on different releases. I also made the folder "Send Only", and made sure to turn off versioning. With all that done, I could share it to the server.

## On the server

On the server I created the directory from which I wanted to host my apt repo, e.g. `/srv/backups/apt-packages/`. Then since [apt expects specific releases/distributions to be in a `dists` subdirectory](https://wiki.debian.org/DebianRepository/Format#Debian_Repository_Format:~:text=The%20distribution%20part%20(stable%20in%20this%20case)%20specifies%20a%20subdirectory%20in%20%24ARCHIVE_ROOT/dists.), I made that too. Finally, I created a directory for that particular distribution, ending up with a directory like `/srv/backups/apt-packages/dists/bionic`. I made sure the Syncthing user on the server had write permissions, and then accepted the shared folder, pointing Syncthing at this new directory.

I also made this folder "Receive Only", turned off versioning, and added the following as ignore patterns (since these will be files that will be created to host the repo, and really don't need to be synced):
```
Release
main
InRelease
```
Last but not least I [turned off deletes](https://docs.syncthing.net/v1.22.0/users/config#config-option-folder.ignoredelete) via the Actions > Advanced > Folders > bionic packages menu. This way, even when my clients delete their local copy of a package, the version on the server will stay.

# Turning a synced folder into an apt repository

The next step is to turn this synced folder into an apt repository. There are [several](https://askubuntu.com/questions/170348/how-to-create-a-local-apt-repository) [guides](https://help.ubuntu.com/community/CreateAuthenticatedRepository) and [resources](https://wiki.debian.org/DebianRepository/Format) on how to do this. After perusing them all, I put together a [little script that does it all for me, automatically](https://github.com/jgillula/apt-repo-manager/tree/v1.0.0#readme). Basically just follow the instructions there.

Don't forget to use the cron file to refresh the repo on a daily/hourly/whatever basis.

# Hosting it

Now we need to make the repo accessible from the network. I want it to be accessible over HTTPS, so the first step is to get an HTTPS certificate. I already wrote about how to do this [in my guide to setting up apt-cacher-ng](/2022/04/setting-up-apt-cacher-ng-with-https/#1-getting-the-https-certificate), so just follow step #1 there.

Next, we need to add the apache config that will host the repo, i.e. `/etc/apache2/sites-available/apt-repo.conf`
```
<VirtualHost *:443>
    ServerName apt-repo.servername
    DocumentRoot /srv/backups/apt-packages

    <Directory />
      Require all granted
      Options Indexes
  </Directory>

  SSLEngine on
  SSLCertificateFile    /etc/ssl/certs/apt-repo.pem
  SSLCertificateKeyFile /etc/ssl/private/apt-repo.key
</VirtualHost>
```
Make sure to adjust `apt-repo.servername` to whatever the hostname of your server actually is, and also the `DocumentRoot` to wherever you're actually keeping the repo (i.e. the parent of the `dists` directory).

Then just do
```bash
sudo a2ensite apt-repo.conf
sudo a2enmod ssl
sudo systemctl restart apache2
```

Last but not least, I need machines on my local network to know at what IP address to find https://apt-repo.servername. I've already set up avahi to do this for other aliases, so I just add `apt-repo.servername` to the `/etc/avahi/aliases` file, and restart `avahi-alias`:
```bash
sudo systemctl restart avahi-alias
```

# Connecting from the client

First, setup the root CA certificate (if you haven't already) on the client, using [the instructions from that old `apt-cacher-ng` post](/2022/04/setting-up-apt-cacher-ng-with-https/#4-install-the-root-ca-certificate-on-your-clients).

Then follow the [instructions from `apt-repo-manager`](https://github.com/jgillula/apt-repo-manager/tree/v1.0.0#access-your-repo-remotely) to access your repo remotely.

# What's next

Since nothing is deleting old packages from the repo on the server it's just going to keep growing and growing. At some point I should probably adjust the `update-local-apt-repos.sh` script to delete really old packages, but hey: space is cheap and I've got plenty of it on my server.
