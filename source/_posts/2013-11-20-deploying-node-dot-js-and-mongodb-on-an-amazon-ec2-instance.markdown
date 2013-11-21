---
layout: post
title: "Deploying Node.js and MongoDB to a Production Amazon EC2 Instance on Ubuntu"
date: 2013-11-20 11:02
comments: true
categories: [nodejs mongodb aws ubuntu servers]
published: false
---

## Provision your EC2 instance
I chose Ubuntu Server 13.10 for my AMI. You'll want to use 64-bit for optimal MongoDB support (see [this post](http://blog.mongodb.org/post/137788967/32-bit-limitations)). Make sure to read through all the options in the wizard to configure your instance for your needs. [MongoDB recommends](http://docs.mongodb.org/ecosystem/platforms/amazon-ec2/) an [instance type](https://aws.amazon.com/ec2/instance-types/) that is EBS-optimized. More on that shortly.

If you use an automatically assigned IP address, you'll lose it when your instance is stopped or terminated. So to prevent any DNS-related interruption of service, you'll want to reserve a dedicated IP address, which Amazon refers to as an [Elastic IP](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html) (EIP). You can't assign an EIP until you've already launched your EC2, so we'll revisit that later.

## Storage
The [AWS storage documentation](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Storage.html) recommends using EBS for persistent data that you care about--like your database. From the documentation:

> An Amazon EBS volume behaves like a raw, unformatted, external block device that you can attach to a single instance. They persist independently from the running life of an Amazon EC2 instance. After an EBS volume is attached to an instance, you can use it like any other physical hard drive. As illustrated in the previous figure, you can attach multiple volumes to an instance. You can also detach an EBS volume from one instance and attach it to another instance.

Again, [MongoDB recommends](http://docs.mongodb.org/ecosystem/platforms/amazon-ec2/) an EBS-optimized EC2 instance with a [Provisioned IOPS EBS volume](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html#EBSVolumeTypes_piops). So set up an EBS volume, and we'll be mounting it for MongoDB to use. It would make good sense to utilize Amazon S3 for static assets, but that's outside the scope of this article.

## Configure Security Group
You'll want to have SSH (port 22) open to any IP you might access it from. You'll also want HTTP and HTTPS ports accessible from any IP.

## A Note About Keypairs
It's really funky that Amazon generates a private key for you to download, rather than letting you paste in your public key(s). It is possible to upload your public key beforehand, via the command line: [ec2-import-keypair](http://docs.aws.amazon.com/AWSEC2/latest/CommandLineReference/ApiReference-cmd-ImportKeyPair.html). I installed [aws-cli](https://github.com/aws/aws-cli) and issued the following command:

```bash
aws ec2 import-key-pair --key-name user@email.com --public-key-material file://~/.ssh/id_rsa.pub
```

If you do this beforehand, your key will show up in the web interface for you to select when launching. I didn't figure this out until after launch, so I generated a key, and copied my normal SSH key into the `authorized_keys` file afterwards. Assuming you've placed the key in your `~/.ssh/` dir:

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

# Onwards: Install and Configure MongoDB
I appreciated the explanations offered by [Murvin Lai](http://www.murvinlai.com/nodejs--mongodb-on-aws.html) and [MongoDB's official documentation](http://docs.mongodb.org/ecosystem/platforms/amazon-ec2/#deploy-mongodb-ec2). Here's the jist of it.

The AWS console will say your volume is something like /dev/sdb, but the actual device name in Ubuntu will be something like /dev/xvdb.

```bash
# Create the file system on your EBS volume.
sudo mkfs -t ext4 <device name> # <device name> is your EBS path, e.g. /dev/xvdb
# Create a folder wherever you want to mount it. For example,
mkdir ~/database
# Mount it
sudo mount <device-name> <folder>
# Create directories for the actual data, logs, and journal
cd ~/database
mkdir data log journal
ln -s ~/database/journal ~/database/data/journal
```

Now that your EBS volume is mounted and ready, let's install MongoDB. These commands are directly from [their tutorial](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/):

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

Now, configure MongoDB to use the EBS volume we setup earlier:

```
sudo nano /etc/mongodb.conf
```
```
dbpath = /home/ubuntu/database/data
logpath = /home/ubuntu/database/log/mongodb.log
```

# Setup Node
```bash
sudo apt-get install nodejs
# Due to a naming conflict, node was renamed to nodejs in apt
# So you'll need to create a symlink to use the command 'node'
ln -s /usr/bin/nodejs /usr/bin/node
```
See [the official wiki](https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager#ubuntu-mint-elementary-os) for more information about installing Node.js on Ubuntu.

Edit your ~/.bashrc to include this:

```bash
NODE_ENV=production
export NODE_ENV
```



