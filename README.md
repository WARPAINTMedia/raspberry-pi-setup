Raspberry Pi Setup
==================

### Introduction

This tutorial will help you install the Raspberry Pi **without using a keyboard or mouse**. It will also enable SSH auth keys (no passwords required), and also setting up a local hostname (no SSH IPs anymore).

### Install OS

First [download Rasbian](http://www.raspbian.org/RaspbianInstaller) and [then install it](http://www.raspberrypi.org/documentation/installation/). 

This whole tutorial assumes **you are using Rasbian**.

### Find Raspberry Pi On Your Network

Install [nmap](http://nmap.org/book/install.html "nmap install instructions") and then run:

```
nmap -sP 192.168.1.0/24 | awk '/^Nmap/{ip=$NF}/B8:27:EB/{print ip}'
```

If you are running through a few routers or something like that, the base IP (192.168.1.XXX) may be different. If this doesn't make sense, [find you local IP first](http://lifehacker.com/5833108/how-to-find-your-local-and-external-ip-address), then use the first 3 sections of that.

This will return an IP that we can use to connect to the Pi. I am *going to pretend* that an IP of **192.168.1.101** was returned.

### SSH Connect To Pi

The default username for the Pi is `pi`.

```
ssh pi@192.168.1.101
```

You will also need to enter in a password, the default is `raspberry`. This can be changed in the next step.

### Setup

Now that you are logged in, you will be asked to run `sudo raspi-config`. Here you can change your password, enable booting to desktop, change your charset/language (I set mine to *en_US.UTF-8*). You should also make sure that your **keyboard is setup correctly and in the correct language**.

### Update

Since this is the first time you ran the Pi, assuming you didn't run any updates during the `raspi-config`, you should run the following one after another:

```
sudo apt-get update
sudo apt-get dist-upgrade
```

This will update the repository and then upgrade the installed programs including updating the OS and removing unused or old packages.

### Use SSH Authorized Key

Enabling the use of an authorized key, lets you sign into the Pi without entering in the password each time.

First [generate a key](https://help.github.com/articles/generating-ssh-keys/) and then copy it to your clipboard (this file is commonly generated as *id_rsa.pub*).

SSH into the Pi, and run:

```
sudo mkdir ~/.ssh
sudo nano ~/.ssh/authorized_keys
```

Then paste the clipboard and press *^X* (control + X). You want to save, so press `Y` when asked, then hit enter to keep the same filename, this will then close nano and save the file.

If you run `cat ~/.ssh/authorized_keys`, you will see your pasted code inside, which should be the exact same as the text from the *.pub* file you generated.

When you next connect to the Pi, you will be asked to add the Pi to your list of known hosts, don't be alarmed. This means that the Pi will now be trusted, and allow you to connect without a password from now on.

### Assign A Local Hostname

Assigning a nice hostname (instead of having a dynamic IP you need to find each time), is much nicer.

```
sudo apt-get install avahi-daemon
```

No don't need to restart after the installation. Just open a new terminal tab and try to ping the Pi.

```
ping raspberrypi.local
```

You will see the results in the terminal. You can press *^C* (control + C) to close the `ping` tool.

I had some issues with the hostname not resolving. I ended up finding these little commands to reset the hostname. You can append them to a `~/.bash_aliases` file and fix things quickly.

```
function fix-hostname() {
  # Update boot startup for avahi-daemon
  sudo insserv avahi-daemon
  # Apply the new configuration with:
  sudo /etc/init.d/avahi-daemon restart
}
```

You can now SSH without using an IP or a password. Yay!

```
ssh pi@raspberrypi.local
```

If you have many PIs on your network, you may want to [change the hostname](http://www.howtogeek.com/167195/how-to-change-your-raspberry-pi-or-other-linux-devices-hostname/).

## Extras

### git projects

This is the function I have added to my `~/.bash_aliases` file. It creates a git repo setup so that you can push git repos to your Pi from your localhost.

Before we run this for the first time, we need to create the proper folder:

```
sudo mkdir /var/www/git/
```

Then we can put this script in the `~/.bash_aliases` file:

```
# create a new git enabled project
function new-project() {
  # where are we starting from?
  LOCATION="$(pwd)"
  # get the sitename
  SITENAME="$1"
  # setup the directories
  mkdir /var/www/$SITENAME # && cd /var/www/$SITENAME
  mkdir /var/www/git/$SITENAME.git && cd /var/www/git/$SITENAME.git
  git init --bare && cd hooks
  # create post-receive file and own it
  echo "git --work-tree=/var/www/$SITENAME --git-dir=/var/www/git/$SITENAME.git checkout -f" > post-receive
  chmod +x post-receive
  # go back to where we started
  cd $LOCATION
  # tell us how to clone this repo, we assume the username is the default `pi`
  echo "git remote add dev ssh://pi@raspberrypi.local:/var/www/git/$SITENAME.git"
}
```

This is how it would work on your local machine, assuming the project was named `test-site`.

```
git remote add dev ssh://pi@raspberrypi.local:/var/www/git/test-site.git
git push dev master
```

This will actually push the project to your Pi, making it available on your Pi web server.

### iptables rules

If you plan on setting up a web server or maybe pushing things to the Pi using git, then you are going to want to setup your iptables properly.

[These settings](https://gist.github.com/james2doyle/0bd0efa7e578781297a3) have been working great for me. Put them in `/etc/iptables.firewall.rules` and run the following.

```
sudo iptables-restore < /etc/iptables.firewall.rules
sudo nano /etc/network/if-pre-up.d/firewall
```

Put this script into that new nano file we started:

```
#!/usr/bin/env sh

/sbin/iptables-restore < /etc/iptables.firewall.rules
```

Own the file so it can be executed:

```
sudo chmod +x /etc/network/if-pre-up.d/firewall
```

Finally, save the iptables setup:

```
sudo iptables-save
```

### Node.js

Download and install an ARM version or Node.js:

```
wget http://node-arm.herokuapp.com/node_latest_armhf.deb 
sudo dpkg -i node_latest_armhf.deb
```

### Luvit

[Luvit](https://github.com/luvit/luvit/) is a framework that wraps up Lua + LibUV + LuaJit.

```
git clone https://github.com/luvit/luvit --branch luvi-up
cd luvit
export LUVI_APP=`pwd`/app
make
make test
```

### fswebcam

`fswebcam` is a tool that can create snapshots and videos from a webcam. It is based on [GD](http://libgd.github.io/) and can be run with loops and timers.

Since we need `GD`, let's get that:

```
sudo apt-get install libgd2-noxpm
sudo apt-get install libgd2-noxpm-dev
```

Now let's install the tool.

```
git clone https://github.com/fsphil/fswebcam
cd fswebcam
./configure --prefix=/usr
make
make install
```

Run `fswebcam --help` to see all the options and flags.

### Bash and Vim Niceness

We maintain a repo called [Digital Ocean Start](https://github.com/WARPAINTMedia/digital-ocean-start). This is what we use for our Digital Ocean servers whenever we set them up. I think it has some sensible defaults for remote servers.







