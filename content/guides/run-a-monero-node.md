+++
title = "Run a Monero Node"
date = 2020-12-31T11:49:13-05:00
author = "Seth Simmons"
authorTwitter = "sethisimmons" #do not include @
cover = ""
tags = ["Monero", "Network", "p2p"]
keywords = ["Monero", "Node", "p2p", "network"]
description = "In this short post I'll detail how to easily run a Monero node on a Linux server, the most common OS for virtual private servers (VPS)."
summary = "In this short post I'll detail how to easily run a Monero node on a Linux server, the most common OS for virtual private servers (VPS)."
showFullContent = false
draft = false
+++

# Introduction

With the [ongoing network attacks in Monero]({{< ref "/content/posts/moneros-ongoing-network-attack.md" >}}), it's a great time for users to dive into running their [own node](https://www.monerooutreach.org/monero_best_practices/your_own_node.html).

In this short post I'll detail how to easily run a Monero node on a Linux server, the most common OS for virtual private servers (VPS). I would highly recommend running either Debian or Ubuntu for your Linux distribution, and this guide will assume you are running one of those.

I will also assume in this guide that you have purchased and SSH'd into the VPS/host of your choosing, but if you need help with those first steps here are a few good links to follow:

- [Hosting services accepting Monero](https://www.getmonero.org/community/merchants/#hosting)
  - These are some of the options available for hosting a VPS while paying with Monero, and each come with pro's and con's.
- [Simple Linode deployment guide](https://www.pragmaticlinux.com/2020/07/setup-a-minimal-debian-10-buster-server-as-a-linode-vps/)
  - If you use Linode, consider using [my referral link](https://www.linode.com/?r=c956dbb75d14063251557a0e5003efb5ceacc74d) so we both get free credits.

If you're using your own hardware at home, this guide will still generally apply to you assuming you are running Ubuntu/Debian.

# Install required packages

First we need to install a few tools we will need later:

```bash
sudo apt-get install git ufw gpg wget
```

# Initial Hardening via UFW

We will want to make sure that the system is hardened in a simple way by making sure that the firewall is locked down to only allow access to the ports necessary for SSH and `monerod`, using UFW.

A great intro to getting started with UFW is available [on DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server).

Run the following commands to add some basic UFW rules and enable the firewall:

```bash
# Install UFW via apt-get
sudo apt-get install ufw

# Deny all non-explicitly allowed ports
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH access
sudo ufw allow ssh

# Allow monerod p2p port
sudo ufw allow 18080/tcp

# Allow monerod restricted RPC port
sudo ufw allow 18089/tcp

# Enable UFW
sudo ufw enable
```

# Download and install monerod

Create our system user and directories for Monero PID and log files:


```bash
# Create a system user and group to run monerod as
sudo adduser --system monero --home /var/lib/monero
sudo addgroup --system monero

# Create necessary directories for monerod
sudo mkdir /var/run/monero
sudo mkdir /var/log/monero

# Set permissions for new directories
sudo chown monero:monero /var/run/monero
sudo chown monero:monero /var/log/monero
```

Download and verify the latest CLI binaries using my gist:

```bash
wget https://gist.githubusercontent.com/sethsimmons/ad5848767d9319520a6905b7111dc021/raw/2b431ccdbd64401a3a43c6a51611524aa797a526/download_monero_binaries.sh
chmod +x download_monero_binaries.sh
./download_monero_binaries.sh
tar xvf monero-linux-*.tar.bz2
cp -r monero-x86_64-linux-gnu-*/* /var/lib/monero/
chown -R monero:monero /var/lib/monero/
```

Full code from the gist:

{{< code language="bash" title="download_monero_binaries.sh " id="0" expand="Show" collapse="Hide" isCollapsed="true" >}}
#!/bin/bash
# Download fluffypony's GPG key
wget -q -O binaryfate.asc https://raw.githubusercontent.com/monero-project/monero/master/utils/gpg_keys/binaryfate.asc

# Verify fluffypony's GPG key
echo "1. Verify binaryfate's GPG key: "
gpg --keyid-format long --with-fingerprint binaryfate.asc

# Prompt user to confirm the key matches that posted on https://src.getmonero.org/resources/user-guides/verification-allos-advanced.html
echo
read -p "Does the above output match https://src.getmonero.org/resources/user-guides/verification-allos-advanced.html?" -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]
then
        # Import binaryfate's GPG key
        echo
        echo "----------------------------"
        echo "2. Import binaryfate's GPG key"
        gpg --import binaryfate.asc
fi

# Download hashes.txt
wget -q -O hashes.txt https://getmonero.org/downloads/hashes.txt

# Verify hashes.txt
echo
echo "--------------------"
echo "3. Verify hashes.txt"
gpg --verify hashes.txt

# Download binary version requested
echo
echo "-------------------------------------"
echo "4. Download latest Linux binaries"
echo -n "Enter your preferred version (64/32) and press [ENTER]: "
read VERSION
if [[ $VERSION == 64 ]]; then
        echo "Downloading..."
        wget -q --content-disposition https://downloads.getmonero.org/cli/linux64
elif [[ $VERSION == 32 ]]; then
        echo "Downloading..."
        wget -q --content-disposition https://downloads.getmonero.org/cli/linux32
else
        echo "Incorrect version chosen, exiting..."
        exit 1
fi
# Verify shasum of downloaded binaries
echo
echo "---------------------------------------"
echo "5. Verify hashes of downloaded binaries"
if grep "$(sha256sum monero-linux-x64-*.tar.bz2 | cut -d " " -f 1)" hashes.txt
then
        echo
        echo "Success: The downloaded binaries verified properly!"
else
        echo
        echo -e "\e[31mDANGER: The download binaries have been tampered with or corrupted\e[0m"
        rm -rf monero-linux-x64-*.tar.bz2
        exit 1
fi
{{< /code >}}

# Install monerod systemd script

Installing `monerod` via a systemd script allows Monero to start automatically on boot, restart on any crash, and log to a given file.

Choose the proper systemd script depending on if you want to prune, run a public restricted RPC node to allow other users to sync their wallets using your node, or any combination of the two:

{{< code language="systemd" title="monerod systemd script" id="1" expand="Show" collapse="Hide" isCollapsed="true" >}}
[Unit]
Description=Monero Full Node (Mainnet)
After=network.target

[Service]
User=monero
Group=monero
WorkingDirectory=~
RuntimeDirectory=~

# Clearnet config
#
Type=forking
PIDFile=/var/run/monero/monerod.pid
ExecStart=/var/lib/monero/monerod --rpc-restricted-bind-ip 0.0.0.0 --rpc-restricted-bind-port 18089 --confirm-external-bind --log-file /var/log/monero/monerod.log --pidfile /var/run/monero/monerod.pid --detach --enable-dns-blocklist
Restart=always
RestartSec=always

[Install]
WantedBy=multi-user.target
{{< /code >}}

{{< code language="systemd" title="monerod systemd script w/ public restricted RPC" id="2" expand="Show" collapse="Hide" isCollapsed="true" >}}
[Unit]
Description=Monero Full Node (Mainnet) with restricted public RPC
After=network.target

[Service]
User=monero
Group=monero
WorkingDirectory=~
RuntimeDirectory=~

# Clearnet config
#
Type=forking
PIDFile=/var/run/monero/monerod.pid
ExecStart=/var/lib/monero/monerod --public-node --rpc-bind-ip 0.0.0.0 --rpc-restricted-bind-ip 0.0.0.0 --rpc-restricted-bind-port 18089 --confirm-external-bind --log-file /var/log/monero/monerod.log --pidfile /var/run/monero/monerod.pid --detach --enable-dns-blocklist
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
{{< /code >}}

{{< code language="systemd" title="monerod systemd script (pruned)" id="3" expand="Show" collapse="Hide" isCollapsed="true" >}}
[Unit]
Description=Monero Full Node (Mainnet)
After=network.target

[Service]
User=monero
Group=monero
WorkingDirectory=~
RuntimeDirectory=~

# Clearnet config
#
Type=forking
PIDFile=/var/run/monero/monerod.pid
ExecStart=/var/lib/monero/monerod --rpc-restricted-bind-ip 0.0.0.0 --rpc-restricted-bind-port 18089 --confirm-external-bind --log-file /var/log/monero/monerod.log --pidfile /var/run/monero/monerod.pid --detach --prune-blockchain --enable-dns-blocklist
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
{{< /code >}}

{{< code language="systemd" title="monerod systemd script w/ public restricted RPC (pruned)" id="4" expand="Show" collapse="Hide" isCollapsed="true" >}}
[Unit]
Description=Monero Full Node (Mainnet) with restricted public RPC
After=network.target

[Service]
User=monero
Group=monero
WorkingDirectory=~
RuntimeDirectory=~

# Clearnet config
#
Type=forking
PIDFile=/var/run/monero/monerod.pid
ExecStart=/var/lib/monero/monerod --public-node --rpc-bind-ip 0.0.0.0 --rpc-restricted-bind-ip 0.0.0.0 --rpc-restricted-bind-port 18089 --confirm-external-bind --log-file /var/log/monero/monerod.log --pidfile /var/run/monero/monerod.pid --detach --prune-blockchain --enable-dns-blocklist
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
{{< /code >}}

Once you've chosen the script you want, simply copy the contents and save it to `/etc/systemd/system/monerod.service` using vim or nano:

```bash
sudo nano /etc/systemd/system/monerod.service
```

*To escape from the nano shell and save the file, hit `ctrl+x`.*

Then run the following to start `monerod`:
```bash
sudo systemctl daemon-reload
sudo systemctl start monerod
sudo tail -f /var/log/monero/monerod.log
```

You should see `monerod` start up properly there and tell you it is synchronizing with the network!

# Updating your Monero node

Whenever a new version is released, you'll want to update as soon as possible to ensure you have the latest fixes, improvements, and features available.

In order to do that, simply run the below commands:

```bash
cd ~/monero
./download_monero_binaries.sh
tar xvf monero-linux-*.tar.bz2
sudo systemctl stop monerod
cp -r monero-x86_64-linux-gnu-*/* /var/lib/monero/
sudo chown -R monero:monero /var/lib/monero
sudo systemctl start monerod
```

That will download the latest binaries, replace the old ones, and restart `monerod` with the latest version!

# Conclusion

Hopefully this guide simplified the process of setting up a remote node on a VPS, and many more similar guides should be popping up shortly.

Please reach out via [Twitter, Keybase, or email]({{< ref "/content/about.md" >}}) if you have any questions, think a step needs clarification, or need further help getting up and running.

Thanks!