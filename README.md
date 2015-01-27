Raspberry Pi Setup
==================

### Introduction

This tutorial will help you install the Raspberry Pi **without using a keyboard or mouse**. It will also show you how to enable SSH auth keys (no passwords required), and setting up a local hostname (like *raspberrypi.local*, so no IPs anymore).

### Install OS

First [download Rasbian](http://www.raspbian.org/RaspbianInstaller) and [then install it](http://www.raspberrypi.org/documentation/installation/).

This whole tutorial assumes **you are using Rasbian**.

### Find Raspberry Pi On Your Network

If you know the IP of your Pi already, skip this step.

Otherwise, install [nmap](http://nmap.org/book/install.html "nmap install instructions") and then run:

```sh
nmap -sP 192.168.1.0/24 | awk '/^Nmap/{ip=$NF}/B8:27:EB/{print ip}'
```

If you are running through a few routers or something like that, the base IP (192.168.1.XXX) may be different. If this doesn't make sense, [find you local IP first](http://lifehacker.com/5833108/how-to-find-your-local-and-external-ip-address), then use the first 3 sections of that.

This will return an IP that we can use to connect to the Pi.

##### Nothing Is Returned

If nothing is returned, you can always use the `arp` command. This lets you list all the devices that your computer can see on the network. You can use it like so:

```sh
arp -a
```

This will list all the devices that are found in your local network. You will see a list of IPs as well as MAC addresses and device IDs.

You can try **one-by-one** to connect to each IP using `ssh pi@IP_HERE`.

### SSH Connect To Pi

I am *going to pretend* that an IP of **192.168.1.105** was returned from the previous command. The default username for the Pi is `pi`.

```sh
ssh pi@192.168.1.105
```

You will also need to enter in a password, the default is `raspberry`. This can be changed in the next step.

### Setup

Now that you are logged in, you will be asked to run `sudo raspi-config`.

Here you can change your password, enable booting to desktop, change your charset/language (I set mine to *en_US.UTF-8*). You should also make sure that your **keyboard is setup correctly and in the correct language**.

This command can always be run later, so if things aren't quite right, just run it again and try different settings.

### Update

Since this is the first time you ran the Pi, assuming you didn't run any updates during the `raspi-config`, you should run the following one after another:

```sh
sudo apt-get update
sudo apt-get dist-upgrade
```

This will update the repository and then upgrade the installed programs including updating the OS and removing unused or old packages.

### Use SSH Authorized Key

Enabling the use of an authorized key lets you sign into the Pi without entering in the password each time.

First, [generate a key](https://help.github.com/articles/generating-ssh-keys/) and then **copy it to your clipboard** (the default name for this file is *id_rsa.pub*).

SSH into the Pi, and run:

```sh
sudo mkdir ~/.ssh
sudo nano ~/.ssh/authorized_keys
```

This creates the folder we need as well as starts a new empty file called `authorized_keys`.

Paste the contents of your clipboard and press `^X` (control + X). You want to save, so press `Y` when asked, then *hit enter* to keep the same filename, this will then close nano and save the file.

If you run `cat ~/.ssh/authorized_keys`, you will see your pasted code inside, which should be the exact same as the text from the *.pub* file you generated.

When you next connect to the Pi, you will be asked to add the Pi to your list of known hosts, don't be alarmed. This means that the Pi will now be trusted. This will allow you to connect without a password from now on.

### Assign A Local Hostname

Assigning a nice hostname (instead of having a dynamic IP which you need to find each time), is much nicer.

```sh
sudo apt-get install avahi-daemon
```

You don't need to restart after the installation. Just open a new terminal tab and try to *ping the Pi*.

```sh
ping -c 5 raspberrypi.local
```

You will see the results in the terminal. This will send 5 pings to the Pi, then return a message that has `5 packets transmitted, 5 packets received, 0.0% packet loss` at the bottom.

If your packet loss is *anything other than 0.0%*, then there is a problem.

You can press `^C` (control + C) to close the `ping` tool early.

You can now SSH without using an IP or a password. Yay!

```sh
ssh pi@raspberrypi.local
```

If you have many PIs on your network, you may want to [change the hostname](http://www.howtogeek.com/167195/how-to-change-your-raspberry-pi-or-other-linux-devices-hostname/).

I had some issues with the hostname not resolving. I ended up finding these little commands to reset the hostname. You can append them to a `~/.bash_aliases` file and fix things quickly.

```sh
function fix-hostname() {
  # Update boot startup for avahi-daemon
  sudo insserv avahi-daemon
  # Apply the new configuration with:
  sudo /etc/init.d/avahi-daemon restart
}
```

## Extras

### git projects

This is a function I have added to my `~/.bash_aliases` file that creates a git repo so that you can *push git repos to your Pi from your local machine*.

Before we run this for the first time, we need to create the proper folder:

```sh
sudo mkdir /var/www/git/
```

Then we can put this function in the `~/.bash_aliases` file:

```sh
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

This is *how it would work on your Pi*, assuming the project was named `test-site`.

```sh
new-project test-site
git remote add dev ssh://pi@raspberrypi.local:/var/www/git/test-site.git
```

Then on the computer that is *pushing to the Pi*:

```sh
git remote add dev ssh://pi@raspberrypi.local:/var/www/git/test-site.git
git push dev master
```

This will actually push the project to your Pi, making it available on your Pi web server.

### iptables rules

If you plan on setting up a web server or maybe pushing things to the Pi using git, then you are going to want to setup your iptables properly.

[These settings](https://gist.github.com/james2doyle/0bd0efa7e578781297a3) have been working great for me. Put them in `/etc/iptables.firewall.rules` and run the following.

```sh
sudo iptables-restore < /etc/iptables.firewall.rules
sudo nano /etc/network/if-pre-up.d/firewall
```

Put this script into that new nano file we started in the last step:

```sh
#!/usr/bin/env sh

/sbin/iptables-restore < /etc/iptables.firewall.rules
```

Own the file so it can be executed:

```sh
sudo chmod +x /etc/network/if-pre-up.d/firewall
```

Finally, save the iptables setup:

```sh
sudo iptables-save
```

### Node.js

Download and install an ARM version of Node.js:

```sh
wget http://node-arm.herokuapp.com/node_latest_armhf.deb
sudo dpkg -i node_latest_armhf.deb
```

This will install the latest version of Node.js for ARM (Raspberry Pi CPU). When it is done, try using `node` to enter the node REPL.

### Luvit

[Luvit](https://github.com/luvit/luvit/) is a framework that wraps up *Lua + LibUV + LuaJit*.

```sh
git clone https://github.com/luvit/luvit --branch luvi-up
cd luvit
export LUVI_APP=`pwd`/app
make
make test
```

Luvit is a lot like Node.js. One of the key differences being that it is usually used as an embedded language, or it's used on low memory systems. This makes it perfect for the Pi, since Node uses a lot of memory and CPU when not configured precisely.

### fswebcam

`fswebcam` is a tool that can create snapshots and videos from a webcam. It is based on [GD](http://libgd.github.io/) and can be run with loops and timers.

Since we need `GD`, let's get that:

```sh
sudo apt-get install libgd2-noxpm
sudo apt-get install libgd2-noxpm-dev
```

Now let's install the tool.

```sh
git clone https://github.com/fsphil/fswebcam
cd fswebcam
./configure --prefix=/usr
make
make install
```

Run `fswebcam --help` to see all the options and flags.

### Bash and Vim Niceness

We maintain a repo called [Digital Ocean Start](https://github.com/WARPAINTMedia/digital-ocean-start).

This is what we use for our Digital Ocean servers whenever we set them up. I think it has some sensible defaults for remote servers.

This project includes the `new-project` function, albeit a little different than the one in this example.