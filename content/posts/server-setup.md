+++
title = "Setting up a server from scratch"
date = 2026-01-07T00:00:00+01:00
draft = false
tags = ["linux", "server", "ubuntu", "self-hosting"]
+++


Setting up a server from scratch is not the hardest thing in the world. But finding a guide that gives you all of the info you need to understand a) what to do and b) why you do it is not that easy. Most documentations need you to have an understanding of server structures, coding, and so on. In this article, I will take you along with my first time setting up a server, learning what is happening, and maybe giving you some guidance for your project. 

Of course, AI tools like ChatGPT will give you all the code you need to set up a server, and they are definitely a good option for debugging and help when an error occurs. For me, using such tools will lead to a functional result, but I often wouldn't understand what I did or why. 

The first question was how to host a server. Of course, outsourced options are available but quite expensive, and for a novice like me, it was a nice challenge to build my own (and a good use for that old Thinkpad I had lying around).
Therefore, the next question was how to turn my Lenovo Thinkpad T470 into a server.
I did a bit of research and stumbled onto Linuxblog.io. They have great articles and documentation on setting up a server.
The Linux distribution that I chose was Ubuntu Server because it seemed good for my use case, easy enough for a beginner like me, and suitable for a SQL database.
  
### The Terminal
This project requires working in the terminal. While there are other options, such as using tools like VS Code, the basic terminal available on every system is sufficient for everything we need to set up this server.

Using the terminal for the first time can feel weird. There is no graphical interface, and all interaction happens through text-based commands. Take your time, and if you have to change parts of the code, I edited it in a different text editor and then simply copied and pasted it. Just make sure that if the code requires, for example, indentation to function, it is preserved during editing and pasting. 
I will briefly explain the most important commands used in this guide, such as `ls`, `cp`, `cat`, `nano`, and `sudo`. If you want to go deeper, you’ll find additional resources linked below.

For now, we will work directly on the laptop hosting the server. Later on, we will access the server remotely from a daily-use machine.

### Installing Ubuntu Server
The first step was to download and install Ubuntu Server onto my Thinkpad. For this, I followed the following tutorial (https://ubuntu.com/tutorials/install-ubuntu-server). This tutorial will walk you through creating a bootable USB drive, booting the laptop from it, and setting it up.
The installation ended up being quite a simple process. 
The next step was to set up a static IP address for the server. 

### Setting up a static IP
Why do we need a static IP? For servers, we use a static IP because we don't want to change the address when contacting the server. This would be equivalent to constantly changing phone numbers. Not a convenient solution. Thankfully, managing IP addresses is quite simple. To enable this, Ubuntu servers have Netplan pre-installed. Netplan allows you to configure IP addresses using YAML files. Here, we can set the server address. 

To find the netplan file, you have to search for it. 
In bash, "ls" is the command to list the contents of the directory you are in. 

```
ls /etc/netplan/
```

The "etc" part is the folder in a Linux system that contains the setup files, like the netplan config or the settings for the SSH configuration.
In my case, there were two YAML files. "00-installer-config.yaml" and "50-cloud-init.yaml". We will work on the 50-cloud-init.yaml document. On some systems, this file is managed by cloud-init, which can overwrite manual changes. For local learning setups, this is usually not an issue.
To look at the document, we enter cat /etc/netplan/50-cloud-init.yaml (`cat` will allow you to look at the file.)

This is the typical structure of the YAML file:

```
network:
  version: NUMBER
  renderer: STRING
  bonds: MAPPING
  bridges: MAPPING
  dummy-devices: MAPPING
  ethernets: MAPPING
  modems: MAPPING
  tunnels: MAPPING
  virtual-ethernets: MAPPING
  vlans: MAPPING
  vrfs: MAPPING
  wifis: MAPPING
  nm-devices: MAPPING
```
  
Unfortunately, mistakes, such as incorrect indentation, can lead to problems; we should first make a short backup. Back in the terminal, we will enter: 

```
sudo cp /etc/netplan/50-cloud-init.yaml ~/50-cloud-init.yaml.bak

```
`cp` stands for "copy". You simply copy and save the file as a backup. 

In case we make a mistake or break something, we can call this backup and undo our happy little accident.
Now we can finally tackle the IP. For that, we open the file using the `nano` command. This allows us to edit the file we are opening.

```
sudo nano /etc/netplan/50-cloud-init.yaml
```

We want to change the DHCP. DHCP stands for Dynamic Host Configuration Protocol. This means that when you boot up the server, the router it is connected to will assign it a dynamic IP address. Once we open the YAML file, we want to disable DHCP. For this, we set "dhcp4" to "no". You will find this under the "Ethernet" section. Make your life easier by setting up the server with an Ethernet cable. Using a laptop's built-in Wi-Fi might be tempting, but it will cause issues at this point (as I painfully learned).

```
network:
  version: 2
  renderer: networkd

  ethernets:
    enp0s31f6:
      dhcp4: no
      addresses:
        - 192.168.2.50/24
      routes:
        - to: default
          via: 192.168.2.1
      nameservers:
        addresses:
          - 192.168.2.135
          - 1.1.1.1
```

If you lose connection, undo the change using the backup file.

Then simply apply the changes.

```
sudo netplan apply
```

To check if the configuration of the IP address has worked, use:

```
ip a
ip route
```

The first step of server hygiene is done. 

### User management
Your default user already has sudo rights. `sudo` stands for "superuser do". This gives you a lot of power - and also the power to break things if you are not careful.
Still, it is good practice to understand how administrative access works.  
In Linux, permissions are managed through groups. Adding a user to the `sudo` group allows them to perform administrative tasks when needed.

```
sudo adduser username
```

To make this user an admin, you call the following line: 

```
sudo usermod -aG sudo yourusername 
```


### Setting up SSH
Of course, I don't want to do any further work on my old ThinkPad (part of the reason I didn’t use it much was that the trackpad and parts of the keyboard stopped working). So setting up SSH is the next logical step. This is where we actually turn that laptop into a real server.
To access it from my laptop, I need a way to communicate with it. This is where SSH comes into play. SSH stands for "Secure Shell". It is a protocol that allows you to access and manage the server while encrypting the communication between you and the server. 
Starting two SSH sessions in two separate terminals might be a good idea. If something in the setup process does not work out and you lock yourself out, you still have the other session with access. 
Ubuntu includes OpenSSH, so getting started is quite simple. Instead of passwords, we use authentication keys. They ensure that only the server and the user can access the data exchanged between them.
For this, you first create the keys:

```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

It will now ask you which file you want to use to save the keys. You can use the directory it recommends. 

Next, we have to give the server the public key. For this, run the line:

```
ssh-copy-id username@your_server_ip
```

Now we can test if the key works. 

```
ssh username@your_server_ip
```

If it works, we can disable the password login by editing the SSH configuration file...

```
sudo nano /etc/ssh/sshd_config
```

... and changing the following options to "no":

```
PasswordAuthentication no
PermitRootLogin no
```

Save the changes and close the file. 

To finish the setup, we just have to restart the SSH service: 

```
sudo systemctl restart sshd
```

SSH key authentication worked out if you can log in without a password.
Next, we make sure to do our housekeeping.


### Managing Server updates
Keeping the systems we use updated on our PC or laptop is normal. On a server, it is often even more crucial to keep everything up to date, because outdated software can quickly become a security risk.

Because this guide is aimed at beginners and one-person projects or small teams, we will take the recommended approach of enabling unattended upgrades. Take the time to read up on server security if you want to go deeper. Manual update management might be the better option in more complex setups.

Once again, Ubuntu shines through its straightforward usability. It comes with the unattended-upgrades package pre-installed. This ensures that the system and its security updates are applied automatically, reducing the risk of the server being exploited. It also keeps updates consistent across the entire system, which helps maintain stability and reliability. 

Because the package is already installed, there is not much left to do. Still, it is worth double-checking whether you want to adjust the update frequency (the default is daily), enable email notifications, or tweak other settings. A good starting point is this short article:  
[https://linuxblog.io/how-to-enable-unattended-upgrades-on-ubuntu-debian/](https://linuxblog.io/how-to-enable-unattended-upgrades-on-ubuntu-debian/)

Monitoring updates from time to time is also a good idea. You can do this by checking the log file at `/var/log/dpkg.log`.

### Server Security

#### Installing Fail2Ban
Next we will install Fail2Ban. Fail2Ban will help you protect your server from attacks by monitoring traffic and banning IP addresses that show suspicious behavior. In simple terms, it tracks how visitors to your server behave. 
One of the most common ways to attack a server is a brute-force attack. Here, the attacker will try to guess, for example, your password by running a script that iterates through every possible combination of characters until access to the server is gained. This is a topic of its own, but if you are interested, you can start reading into it here (https://en.wikipedia.org/wiki/Brute-force_attack)
Setting up Fail2Ban again is quite simple. In this case, we first have to install it. For our Ubuntu server, we will run the line:

```
sudo apt update
sudo apt install fail2ban  
```

Now we need to set up the configuration file. For this, we run:

```
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

In your jail.local file, you will make sure that the 'sshd' section is enabled. "Jails" are the security rules that are active. 
In our jail.local file we will check and or change the sshd section to: 

```
[sshd]
enabled = true
port = ssh
logpath = /var/log/auth.log  # Adjust for your distribution
maxretry = 5
```

Now restart:

```
sudo systemctl restart fail2ban
```

To verify that Fail2Ban is running, we can check its status:

```
sudo fail2ban-client status
```

This will show you if Fail2Ban is active.

To look at the banned IPs, you can enter:

```
sudo fail2ban-client status sshd
```

If you use a home server like me, there might be no banned IP. 

All of these steps and code are directly taken from the Fail2Ban section of the [Linux Server Setup – Part 1: A Beginner’s Guide](https://linuxblog.io/linux-server-setup/ "Linux Server Setup – Part 1: A Beginner’s Guide")


#### Setting up the Firewall
The firewall has the same task as security at a stadium. It checks who is going in or out and if they are on the guest list or banned. Keeping the stadium analogy, your server, just as the stadium has multiple entrances, so-called "ports". A firewall closes ports that are not needed, just as entrances to the stadium that are not used will be locked. 
Because I like it uncomplicated, I installed UFW (Uncomplicated Firewall) - but also because it is a common tool to manage your firewall. We just need to run the following commands. 
First, we need to allow SSH traffic - otherwise, we cannot access our server via SSH

```
sudo ufw allow OpenSSH
```

Now we simply enable UFW...

```
sudo ufw enable
```

... and check its status

```
sudo ufw status
```

You should see that SSH is allowed and all other incoming traffic is denied.
Again, all of these code snippets are available in the Linux Server Setup – Part 1: A Beginner’s Guide article, along with additional information and resources. 
This setup does not make the server “unhackable”, but it will protect the server from the most common attacks.

### Summary
The first step of our project is done. As in sports, building a foundation is often not the most fun part. It is a lot of (often boring) work, and it seems that you cannot see any results. But now we have laid the foundation to begin the main part of the project. 
As previously mentioned, Linuxblog.io was an excellent resource for me, and most of this article is based on their guide.
If you have any questions, feedback, or things that I forgot, let me know in the comments. 

We now have a server that is reachable and secured. The next step is to set up the database and start making the server actually useful.

### Resources
Canonical Ltd. (n.d.). _Install Ubuntu Server_.  
[https://ubuntu.com/tutorials/install-ubuntu-server]

Linuxblog.io. (n.d.). _Linux server setup – Part 1: A beginner’s guide_.  
[https://linuxblog.io/linux-server-setup/]

Linuxblog.io. (n.d.). _How to enable unattended upgrades on Ubuntu/Debian_.  
[https://linuxblog.io/how-to-enable-unattended-upgrades-on-ubuntu-debian/]

Netplan. (n.d.). _Netplan YAML configuration_.  
[https://netplan.readthedocs.io/en/stable/netplan-yaml/]

OpenBSD Project. (n.d.). _OpenSSH manual pages_.  
[https://www.openssh.com/manual.html]

Wikipedia contributors. (n.d.). _Brute-force attack_. In _Wikipedia_.  
[https://en.wikipedia.org/wiki/Brute-force_attack]

### Further Readings 
Cloudflare, Inc. (n.d.). _What is DHCP?_  
https://www.cloudflare.com/learning/network-layer/what-is-dhcp/

Cloudflare, Inc. (n.d.). _What is an IP address?_  
[https://www.cloudflare.com/learning/dns/glossary/what-is-my-ip-address/]

DigitalOcean. (n.d.). _SSH essentials: Working with SSH servers, clients, and keys_.  
[https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys]

DigitalOcean. (n.d.). _An introduction to securing your Linux VPS_.  
[https://www.digitalocean.com/community/tutorials/an-introduction-to-securing-your-linux-vps]

Fail2Ban Project. (n.d.). _Fail2Ban documentation_.  
[https://www.fail2ban.org/wiki/index.php/Main_Page]

Shotts, W. (n.d.). _The Linux command line_.  
[https://linuxcommand.org/tlcl.php]




