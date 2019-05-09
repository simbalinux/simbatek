---
layout: post
title:  Look what we have done
date:   2015-04-14 10:52:12
images: images/@stock/blog-1.jpg
excerpt:
  Iste neque doloribus dolor quis ad sit dolores dolor sit perferendis. nemo in rerum ducimus possimus aspernatur quas est. dolorem eaque vel id quasi voluptatem eligendi rerum et quo ut. fuga qui ea voluptates sunt
categories: Design Development
---

# SSH tunnelling for fun and profit: Autossh

Now that you are able to create various forward or reverse SSH tunnels with lots of options and even simplify your live with `~/.ssh/config` you probably also want to know how make a tunnel persistent. By persistent I mean, that it is made sure the tunnel will always run. For example, once your ssh connection times out (By server-side timeout), your tunnel should be re-established automatically.

I know there are plenty of scripts out there which try to do that somehow. Some scripts use a while loop, others encourage you to run a remote command (such as tail) to make sure you don't run into timeout and various others. But actually, you don't want to re-invent the wheel and stick to bullet-proof already existing solutions. So the game-changer here is [AutoSSH][1].

## TL;DR
    
    
    autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -L 5000:localhost:3306 cytopia@everythingcli.org
    

or fully configured (via `~/.ssh/config`) for background usage
    
    
    autossh -M 0 -f -T -N cli-mysql-tunnel
    

## What is AutoSSH



> Autossh is a program to start a copy of ssh and monitor it, restarting it as necessary should it die or stop passing traffic.

## Install AutoSSH

How to install AutoSSH on various systems via their package manager.

| OS                     | Install method                   |  
| ---------------------- | -------------------------------- |  
| Debian / Ubuntu        | `$ sudo apt-get install autossh` |  
| CentOS / Fedora / RHEL | `$ sudo yum install autossh`     |  
| ArchLinux              | `$ sudo pacman -S autossh`       |  
| FreeBSD                | `# pkg install autossh`    

or  
`# cd /usr/ports/security/autossh/ && make install clean` |  
| OSX |  `$ brew install autossh` | 

Alternatively you can also compile and install AutoSSH from source:
    
    
    wget http://www.harding.motd.ca/autossh/autossh-1.4e.tgz
    gunzip -c autossh-1.4e.tgz | tar xvf -
    cd autossh-1.4e
    ./configure
    make
    sudo make install
    

**Note:** Make sure to grab the latest version which can be found here: .

## Basic usage
    
    
    usage: autossh [-V] [-M monitor_port[:echo_port]] [-f] [SSH_OPTIONS]
    

Ignore `-M` for now. `-V` simply displays the version and exits. The important part to remember is that `-f` (run in background) is not passed to the `ssh` command, but handled by `autossh` itself. Apart from that you can then use it just like you would use ssh to create any forward or reverse tunnels.

Let's take the basic example from part one of this article series (forwarding a remote MySQL port to my local machine on port 5000):
    
    
    ssh -L 5000:localhost:3306 cytopia@everythingcli.org
    

This can simply be turned into an autossh command:
    
    
    autossh -L 5000:localhost:3306 cytopia@everythingcli.org
    

This is basically it. Not much magic here.

**Note 1:** Before you use `autossh`, make sure the connection works as expected by trying it with `ssh` first.

**Note 2:** Make sure you use public/private key authentification instead of password-based authentification when you use `-f`. This is required for `ssh` as well as for `autossh`, simply because in a background run a passphrase cannot be entered interactively.

## AutoSSH and `-M` (monitoring port)

With `-M` AutoSSH will continuously send data back and forth through the pair of monitoring ports in order to keep track of an established connection. If no data is going through anymore, it will restart the connection. The specified monitoring and the port directly above (+1) must be free. The first one is used to send data and the one above to receive data on.

Unfortunately, this is not too handy, as it must be made sure both ports (the specified one and the one directly above) a free (not used). So in order to overcome this problem, there is a better solution:

`ServerAliveInterval` and `ServerAliveCountMax` – they cause the SSH client to send traffic through the encrypted link to the server. This will keep the connection alive when there is no other activity and also when it does not receive any alive data, it will tell AutoSSH that the connection is broken and AutoSSH will then restart the connection.

The [AutoSSH man page][2] also recommends the second solution:

> -M [:echo_port],  
…  
In many ways this [ServerAliveInterval and ServerAliveCountMax options] may be a better solution than the monitoring port.

You can disable the built-in AutoSSH monitoring port by giving it a value of 0:
    
    
    autossh -M 0
    

Additionally you will also have to specify values for `ServerAliveInterval` and `ServerAliveCountMax`
    
    
    autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3"
    

So now the complete tunnel command will look like this:
    
    
    autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -L 5000:localhost:3306 cytopia@everythingcli.org
    

| Option              | Description                                                                                                                                                                                                                                                    |  
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |  
| ServerAliveInterval | ServerAliveInterval: number of seconds that the client will wait before sending a null packet to the server (to keep the connection alive).    
Default: 30           |  
| ServerAliveCountMax | Sets the number of server alive messages which may be sent without ssh receiving any messages back from the server. If this threshold is reached while server alive messages are being sent, ssh will disconnect from the server, terminating the session.    
Default: 3            |  

## AutoSSH and `~/.ssh/config`

In the previous article we were able to simplify the tunnel command via `~/.ssh/config`. Luckily autossh is also aware of this file, so we can still keep our configuration there.

This was our very customized configuration for ssh tunnels which had custom ports and custom rsa keys:
    
    
    $ vim ~/.ssh/config
     Host cli-mysql-tunnel
        HostName      everythingcli.org
        User          cytopia
        Port          1022
        IdentityFile  ~/.ssh/id_rsa-cytopia@everythingcli
        LocalForward  5000 localhost:3306
    

We can also add the `ServerAliveInterval` and `ServerAliveCountMax` options to that file in order to make things even easier.
    
    
    $ vim ~/.ssh/config
     Host cli-mysql-tunnel
        HostName      everythingcli.org
        User          cytopia
        Port          1022
        IdentityFile  ~/.ssh/id_rsa-cytopia@everythingcli
        LocalForward  5000 localhost:3306
        ServerAliveInterval 30
        ServerAliveCountMax 3
    

If you recall all the ssh options we had used already, we can now simply start the autossh tunnel like so:
    
    
    autossh -M 0 -f -T -N cli-mysql-tunnel
    

## AutoSSH environment variables

AutoSSH can also be controlled via a couple of environmental variables. Those are useful if you want to run AutoSSH unattended via `cron`, using shell scripts or during boot time with the help of `systemd` services. The most used variable is probably `AUTOSSH_GATETIME`:

> **AUTOSSH_GATETIME**  
How long ssh must be up before we consider it a successful connection. Default is 30 seconds. If set to 0, then this behaviour is disabled, and as well, autossh will retry even on failure of first attempt to run ssh.

Setting `AUTOSSH_GATETIME` to 0 is most useful when running AutoSSH at boot time.

All other environmental variables including the once responsible for logging options can be found in the [AutoSSH Readme][2].

## AutoSSH during boot with systemd

If you want a permanent SSH tunnel already created during boot time, you will (nowadays) have to create a systemd service and enable it. There is however an important thing to note about systemd and AutoSSH: `-f` (background usage) already implies `AUTOSSH_GATETIME=0`, however `-f` is not supported by systemd.

>   
[…] running programs in the background using "&", and other elements of shell syntax are not supported.

So in the case of `systemd` we need to make use of `AUTOSSH_GATETIME`. Let's look at a very basic service:
    
    
    $ vim /etc/systemd/system/autossh-mysql-tunnel.service
    [Unit]
    Description=AutoSSH tunnel service everythingcli MySQL on local port 5000
    After=network.target
    
    [Service]
    Environment="AUTOSSH_GATETIME=0"
    ExecStart=/usr/bin/autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -NL 5000:localhost:3306 cytopia@everythingcli.org -p 1022
    
    [Install]
    WantedBy=multi-user.target
    

Tell systemd that we have added some stuff:
    
    
    systemctl daemon-reload
    

Start the service
    
    
    systemctl start autossh-mysql-tunnel.service
    

Enable during boot time
    
    
    systemctl enable autossh-mysql-tunnel.service
    

_

This is basically all I found useful about AutoSSH. If you thing I have missed some important parts or you know any other cool stuff, let me know and I will update this post.

**Discussions**

[1]: http://www.harding.motd.ca/autossh/
[2]: http://www.harding.motd.ca/autossh/README

  
