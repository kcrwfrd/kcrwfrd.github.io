---
layout: post
title: "Deploying a Production App on Node.js and MongoDB to an Amazon EC2 Instance on Ubuntu"
date: 2013-11-20 11:02
comments: true
categories: [nodejs mongodb aws ubuntu servers]
published: false
---

This tutorial will be a two-part series. First, we'll cover launching an EC2 instance, setting up the Node.js/MongoDB stack, and keeping your app running. The next part will focus on build and deployment methodologies, specifically for a JS-heavy thick client built on a framework like Angular/Backbone/Ember/\<insert framework here\>.

# Part 1: Setup Your Stack

## First, a Note About Keypairs
When you launch a new instance, it's pretty funky that Amazon doesn't let you paste or upload your public key to use for authentication. You can either generate a new private key to download, or select an existing key from your account. I recommend that you upload your public key beforehand, by going to the __AWS Console -> EC2 -> Key Pairs -> Import Key Pair__.

You can also upload a public key via the command line, with [ec2-import-keypair](http://docs.aws.amazon.com/AWSEC2/latest/CommandLineReference/ApiReference-cmd-ImportKeyPair.html). I installed [aws-cli](https://github.com/aws/aws-cli) (note: different from `ec2-import-keypair`) and issued the following command:

```bash
aws ec2 import-key-pair --key-name user@email.com --public-key-material file://~/.ssh/id_rsa.pub
```

If you do this beforehand, your key will show up in the web interface for you to select when launching.

## Provision your EC2 instance
I chose Ubuntu Server 13.10 for my AMI. You'll want to use 64-bit for optimal MongoDB support (see [this post](http://blog.mongodb.org/post/137788967/32-bit-limitations)). Make sure to read through all the options in the wizard to configure your instance for your needs. [MongoDB recommends](http://docs.mongodb.org/ecosystem/platforms/amazon-ec2/) an [instance type](https://aws.amazon.com/ec2/instance-types/) that is EBS-optimized. More on that shortly.

If you use an automatically assigned IP address, you'll lose it when your instance is stopped or terminated. So, to prevent any DNS-related interruption of service, you'll want to reserve a dedicated IP address, which Amazon refers to as an [Elastic IP](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html) (EIP). You can't assign an EIP until you've already launched your EC2, so we'll revisit that later. For right now, automatically assign a public IP.

## Storage
The [AWS storage documentation](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Storage.html) recommends using EBS for persistent data that you care about--like your database. From the documentation:

> An Amazon EBS volume behaves like a raw, unformatted, external block device that you can attach to a single instance. They persist independently from the running life of an Amazon EC2 instance. After an EBS volume is attached to an instance, you can use it like any other physical hard drive. As illustrated in the previous figure, you can attach multiple volumes to an instance. You can also detach an EBS volume from one instance and attach it to another instance.

This comes in handy if you need to upgrade your EC2 instance, or provision a new one for any reason. Again, [MongoDB recommends](http://docs.mongodb.org/ecosystem/platforms/amazon-ec2/) an EBS-optimized EC2 instance with a [Provisioned IOPS EBS volume](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html#EBSVolumeTypes_piops). So set up an EBS volume, and we'll be mounting it for MongoDB to use. It would make good sense to utilize Amazon S3 for static assets, but that's outside the scope of this article.

## Configure Security Group
You'll want to have SSH (port 22) open to any IP you might access it from. You'll also want HTTP and HTTPS ports accessible from any IP.

## Launch!

You didn't upload your SSH key beforehand, did you? That's okay, I didn't figure that out until afterwards, either. So I generated a key, and copied my normal SSH key into the `authorized_keys` file afterwards. Assuming you've placed the key in your `~/.ssh/` dir:

```bash
# Make sure you restrict permissions for your private key file
chmod 400 ~/.ssh/aws.pem
# Add the downloaded AWS key to your key agent
ssh-add ~/.ssh/aws.pem
# If you don't have ssh-copy-id, you can install it via homebrew:
brew install ssh-copy-id
# Then, copy your normal SSH key to the EC2's ~/.ssh/authorized_keys file
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@hostname
```

Now you can setup [ssh agent forwarding](https://help.github.com/articles/using-ssh-agent-forwarding) for handy stuff like connecting to Github!

## Onwards: Install and Configure MongoDB
Again, this is all based on [MongoDB's official documentation](http://docs.mongodb.org/ecosystem/platforms/amazon-ec2/#deploy-mongodb-ec2). They recommend using separate EBS stores for your data, journal, and log, but I'm just going to cover putting it all on a single EBS to keep things a little simpler.

First, let's install MongoDB. These commands are directly from [their tutorial](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/):

```bash
# Import MongoDB's GPG key used to ensure package authenticity
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
# Create a /etc/apt/sources.list.d/mongodb.list file
echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | sudo tee /etc/apt/sources.list.d/mongodb.list
# Update your repository
sudo apt-get update
# Install MongoDB
sudo apt-get install mongodb-10gen
```

Now that that's done, let's configure it to use the EBS volume we setup earlier. The AWS console will say your volume is something like /dev/sdb, but the actual device name in Ubuntu will be something like /dev/xvdb.

```bash
# Create the file system on your EBS volume. <device name> is your EBS path
# sudo mkfs -t ext4 <device name>
sudo mkfs -t ext4 /dev/xvdb
# Create a folder wherever you want to mount it
# mkdir <mount point>
mkdir /database
# Add an fstab entry so it gets mounted on system boot
echo '/dev/xvdb /database ext4 defaults,auto,noatime,noexec 0 0' | sudo tee -a /etc/fstab
# Mount it
sudo mount /dev/xvdb /database
# Create directories for the actual data, logs, and journal
cd /database
mkdir data journal log
# Set MongoDB to the owner of these directories
sudo chown -R mongodb:mongodb /database
# And link your journal
ln -s /database/journal /database/data/journal
```

Now configure MongoDB to use these paths: `sudo nano /etc/mongodb.conf`


```
dbpath = /database/data
logpath = /database/log/mongodb.log
```

## Install Node
See [the official wiki](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager#ubuntu-mint-elementary-os) for more information about installing Node.js on Ubuntu. Ubuntu's default Node package lags behind the latest stable release. If that's okay, then go ahead:

```bash
sudo apt-get install nodejs
# Due to a naming conflict, node was renamed to nodejs in apt
# So you'll need to create a symlink to use the command 'node'
ln -s /usr/bin/nodejs /usr/bin/node
```

But if you want to install the latest version of Node, you're gonna have to do this:

```bash
sudo apt-get update
sudo apt-get install python-software-properties python g++ make
sudo apt-add-repository ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get install nodejs
ln -s /usr/bin/nodejs /usr/bin/node
```

Listening on port 80 requires root privileges. Instead, Node will listen on port 3000, and we'll redirect requests from port 80 to there:

```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3000
```

Add the command (minus `sudo`) to your rc.local file to make sure this applies on boot as well: `sudo nano /etc/rc.local`

## Keep your app running forever
For this, we're going to use [Forever](https://github.com/nodejitsu/forever) and [Upstart](http://upstart.ubuntu.com/). Forever is used in production at [Nodejitsu](https://www.nodejitsu.com/), and restarts your app if it crashes. Upstart registers your app as a service, starting it on boot and cleanly stopping it on shutdown. The configuration that follows is directly from [this awesome guide on exratione.com](https://www.exratione.com/2013/02/nodejs-and-forever-as-a-service-simple-upstart-and-init-scripts-for-ubuntu/).

First, install Forever:

```bash
sudo npm install -g forever
```

Then, create the following Upstart configuration: `sudo nano /etc/init/myapp.conf`

```bash
# My App upstart config /etc/init/myapp.conf
description "Startup script for My App using Forever"

start on startup
stop on shutdown

# So that Upstart reports the pid of the Node.js process started by Forever
# rather than Forever's own pid
expect fork

# Full path to the node binaries
env NODE_BIN_DIR="/usr/bin/node"

# Path for finding global NPM node_modules
env NODE_PATH="/usr/lib/nodejs:/usr/lib/node_modules:/usr/share/javascript"

# Directory containing My App
env APPLICATION_DIRECTORY="/home/ubuntu/myapp"

# Application javascript filename
env APPLICATION_START="server.js"

# Environment to run app as
env NODE_ENV="production"

# Log file
env LOG="/var/log/chirp.log"

script
  PATH=$NODE_BIN_DIR:$PATH

  exec forever --sourceDir $APPLICATION_DIRECTORY --append -l $LOG \
    --minUptime 5000 --spinSleepTime 2000 start $APPLICATION_START
end script

pre-stop script
  PATH=$NODE_BIN_DIR:$PATH
  exec forever stop $APPLICATION_START >> $LOG
end script

```

Now you can start, restart, and stop your app like this:

```bash
sudo service myapp start
sudo service myapp restart
sudo service myapp stop
```
