---
layout: post
title: "How to install node.js and npm without sudo and curl on remote Linux."
date: 2016-03-18 09:25:00 +0100
comments: true
---

There is sometimes a need to install  on a limited system. The account has no
sudo, no curl. It has only ssh and sftp access, home folder and maybe some open
ports to the internet. In such case, standard installation won't
work.

This happened to me, when I tried to install Node.js to our GNU/Linux log
server. In past I have written in Java a really fast search engine to seek our ~20TB logs. Now I wanted to play with Node.js and make some functionality in
Javascript. 

Unfortunately, companies do not like the idea of granting `sudo`
rights very much (and this is quite understandable). The other option was to
ask the admins to perform the installation. But it would start company heavy weight internal bureaucratic
 process with no end visible, and would certainly summon a lot of
"why" questions, and I do not have right answers. 

To save my time, and my
nerve I have tried to find a way how to install `node.js` and `npm` without `sudo`
and without `curl`. I have succeeded and you can read my findings in following
instructions.

Credits: in this great post [How to use npm global without sudo on
OSX](http://www.johnpapa.net/how-to-use-npm-global-without-sudo-on-osx/) John
Papa shows how to install npm global without sudo.  I have used this approach
with little modifications to install node.js and npm on Linux without sudo on your account as well.


>Disclaimer: I offer no warranty nor guarantee of the success of these steps. These are simply what works in my experience. Follow these steps with a cautious eye as you would any steps you find on the internet.

Main Idea
=========

We install Node.js and npm without sudo in "sandbox" environment - virtual machine. Then we
copy the installed files to the target system and set all necessary settings we have made in virtual machine.   


Instructions
============

##### Prepare virtual machine
* download, install and run new virtual machine with the same or similar system, as your target system. In my case I have chosen free Ubuntu 64bit, offered by Parallels Desktop. You can of course use another application for VM, like VMware or VirtualBox.

* run terminal in your virtual machine and get the curl for your virtual machine. `sudo apt-get update`, then `sudo apt-get install curl`

##### Install Node.js
* go to [https://nodejs.org/en/download/](https://nodejs.org/en/download/) and download appropriate binary package for your system. In case you do not know whether to download 32 or 64 bit version use `uname -a` command to see details of your system. In my virtual Ubuntu the downloaded file was located at `~/Downloads/node-v4.4.0-linux-x64.tar.xz ` If you get another version name, please replace this name throughout these steps.

* extract the archive to your home directory
```
tar -xvf Downloads/node-v4.4.0-linux-x64.tar.xz
```

* There is `~/node-v4.4.0-linux-x64` directory now.
Ensure you'll find installed binaries by adding the following to your `.bashrc`. `echo PATH="${HOME}/node-v4.4.0-linux-x64/bin\:\$PATH" >> ${HOME}/.bashrc` 

* Close your terminal window and open it again. Node.js can be run by invoking the command
```
node -v
```
which should display your node.js version.

##### Install Npm
* create a directory for your global packages. I prefer to name my folder .npm-packages, you can choose what you want as long as you replace it throughout these steps. The `${HOME}` is a variable that translates to `~/` or `/home/parallels/` for me. 
```
mkdir "${HOME}/.npm-packages"
```

* Reference this directory for future usage in your `.bashrc` file. The echo command helps write the statement. The `>>` tells it where to write it which is followed by the file to write it to. `echo NPM_PACKAGES="${HOME}/.npm-packages" >> ${HOME}/.bashrc`

* Indicate to npm where to store your globally installed package, in your `~/.npmrc` file. `echo prefix=${HOME}/.npm-packages >> ${HOME}/.npmrc`

* This gets and installs the latest npm, which is 3.8.2 at the time of this writing. `curl -L https://www.npmjs.org/install.sh | sh`	

* Ensure node will find the packages by adding the path to your `.bashrc`. Notice the escape characters before special characters. `echo NODE_PATH=\"\$NPM_PACKAGES/lib/node_modules\:\$NODE_PATH\" >> ${HOME}/.bashrc`

* Ensure you'll find installed binaries by adding the following to your `.bashrc`. `echo PATH=\"\$NPM_PACKAGES/bin\:\$PATH\" >> ${HOME}/.bashrc`

* Ensure that you source your `.bashrc` file by adding the following to your `.bash_profile`. `echo source "~/.bashrc" >> ${HOME}/.bash_profile`


* Close your terminal window and open it again. Npm can be run by invoking the command
```
npm list -g --depth=0
```
which should display where is npm now located and your npm version.

As far as we have working virtual machine installation, we can use it for our target system.

#### Transfer installation to your target system

We don't have curl on our target system, do you remember? So we need to copy files from our virtual machine installation to our target system. 

* In my virtual Ubuntu, I went to `Files` on left bar, then pressed `Ctrl+H` to show hidden files. Then I copied `node-v4.4.0-linux-x64` and `.npm-packages` to my home OSX folder which Parallels kindly mapped for Ubuntu.
 
* Move folders `node-v4.4.0-linux-x64` and `.npm-packages` located now in your home OSX folder to your home folder on the target system. I personally prefer ForkLift for sftp tranfers.

Link to npm will be most probably replaced by actual file during the sftp transfer. So we need to restore it. Firstly remove the file and then restore the link.


* Log into your target system using ssh. Then 

```
rm ~/.npm-packages/bin/npm

ln -s /home/yourtargethome/.npm-packages/lib/node_modules/npm/bin/npm-cli.js /home/yourtargethome/.npm-packages/bin/npm
```
* You can also remove obsolete npm in node folder:

```
rm ~/node-v4.4.0-linux-x64/bin/npm
```

Now we need to set all settings made in Ubuntu for our target sytem. Pls check your Ubuntu conf files and make sure you copy these settings to your target system.

* `~/.npmrc`

```
prefix=/home/yourhomefolder/.npm-packages
```

* `~/.bashrc`

```
PATH=/home/yourhomefolder/node-v4.4.0-linux-x64/bin\:$PATH
NPM_PACKAGES=/home/yourhomefolder/.npm-packages
NODE_PATH="$NPM_PACKAGES/lib/node_modules:$NODE_PATH"
PATH="$NPM_PACKAGES/bin:$PATH"
```

* `~/.bash_profile`

Make sure you have the `source ~/.bashrc` in your `~/.bash_profile`


That's all. Log using ssh again and try `node -v` and `npm list -g --depth=0` on your target system. Node.js and npm are now installed within home folder on your account.

Comments:
========