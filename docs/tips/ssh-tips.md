---
title: SSH examples, tips and tunnels
description: examples to take your remote system admin game to the next level. Commands and tips to not only use SSH but master ways to move around the network. 
tags: ["ssh", "forward", "tunnel", "proxy"]
---

# SSH examples, tips and tunnels

[Source](https://hackertarget.com/ssh-examples-tunnels/ "Permalink to 22 SSH Examples, Practical Tips & Tunnels | HackerTarget.com")

![SSH examples, tips and tunnels](https://hackertarget.com/wp-content/uploads/2018/12/ssh-examples-tunnel.png)Practical [SSH](https://en.wikipedia.org/wiki/Secure_Shell) examples to take your **remote system admin** game to the next level. Commands and tips to not only use `SSH` but master ways to move around the network. 

Knowing a few `ssh` tricks will benefit any **system administrator**, **network engineer** or **security professional**.

## Practical SSH examples

 Even if you are an experienced **\*nix guru** there are a couple of examples further down that are only available in later versions of [OpenSSH](https://www.openssh.com/releasenotes.html). Take a look at [Proxy Jump **-J**](https://hackertarget.com/ssh-examples-tunnels/#ssh-jump) and [reverse dynamic forwarding **-R**](https://hackertarget.com/ssh-examples-tunnels/#ssh-tunnel-reverse-proxy).

## First The Basics

#### Breaking down the SSH Command Line

The following `ssh` example command uses common parameters often seen when connecting to a remote `SSH` server.

    localhost:~$ ssh -v -p 22 -C neo@remoteserver

-v : Print debug information, particularly helpful when debugging an authentication problem. Can be used multiple times to print additional information.
-p 22 : Specify which **port to connect to** on the remote SSH server. **22** is not required as this is the default, but if any other port is listening connect to it using the `-p` parameter. The listening port is configured in the `sshd_config` file using the `Port 2222` format.
-C : Compression is enabled on the connection using this parameter. If you are using the terminal over a slow link or viewing lots of text this can speed up the connection as it will compress the data transferred on the fly.
neo@ : The string before the **@** symbol denotes the username to authenticate with against the remote server. Leaving out the **user@** will default to using the username of the account you are currently logged in to (~$ whoami). User can also be specified with the `-l` parameter.
remoteserver : The hostname that the `ssh` is connecting to, this can be a fully qualified domain name, an IP address or any host in your local machines hosts file. To connect to a host that resolves to both **IPv4** and **IPv6** you can specify add a parameter `-4` or `-6` to the command line so it resolves correctly.

Each of the above a parameters are optional apart from the remoteserver.

#### Using a Configuration File

While many users are familiar with the `sshd_config` file, there is also a client configuration file for the `ssh` command. This defaults to `~/.ssh/config` but can also be specified as a parameter with the `-F` option.

    Host remoteserver
         HostName remoteserver.thematrix.io
         User neo
         Port 2112
         IdentityFile /home/test/.ssh/remoteserver.private_key

    Host *
         Port 2222

In the above **example ssh configuration** file you can see that there are two Host entries. The first is a specific host entry with **Port 2112** configured, as well as a custom IdentifyFile and username. The second is a wildcard value of **\*** that will match all hosts. Note that **the first configuration option found will be used**, so the most specific should be at the top of the configuration. More information is found in the man page (`man ssh_config`).

The configuration file can save a lot of typing by including advanced configuration shortcuts any time a connection is made to particular hosts.

#### Copy Files over SSH with SCP

The `ssh` client comes with two other very handy tools for moving files around over an **encrypted ssh connection**. The commands are `scp` and `sftp`, see the examples below for basic usage. Note that many parameters for the `ssh` can be applied to these commands also.

    localhost:~$ scp mypic.png neo@remoteserver:/media/data/mypic_2.png

In this example the file **mypic.png** was copied to the **remoteserver** to file system location **/media/data** and was renamed to **mypic\_2.png**.

Don't forget **the difference in the port parameter**. This is a gotcha that hits everyone using `scp` on the command line. The port parameter is `-P` not `-p` as it is in the `ssh` client.!. You will forget, but don't worry everyone does.

For those familiar with command line `ftp`, many of the commands are similar when using `sftp`. You can **push**, **put** and **ls** to your hearts desire.

    sftp neo@remoteserver

## Practical Examples

In many of these examples we could achieve the result using a number of methods. As in all our [tutorials](https://hackertarget.com/research/#tutorial) and example command sheets, the focus is practical examples that get the job done.

### 1\. Proxy Traffic over SSH using SOCKS

The SSH Proxy feature has been placed at number 1 for good reason. It is more powerful than many users realise giving you access to any system that the remote server can reach, using almost any application. The `ssh` client can tunnel traffic over the connection using a **SOCKS** proxy server with a quick one liner. A key thing to understand is that traffic to the remote systems will have a source of the remote server. For example in a web server log file.

    localhost:~$ ssh -D 8888 user@remoteserver

    localhost:~$ netstat -pan | grep 8888
    tcp        0      0 127.0.0.1:8888       0.0.0.0:*               LISTEN      23880/ssh

Here we start the socks proxy server running on TCP port 8888, the second command checks that the port is now listening. The `127.0.0.1` indicates the service is running on localhost only. We can use a slightly different command to listen on all interfaces including ethernet or wifi, this will allow other applications (browsers or other) on our network to connect to the **ssh socks proxy** service.

    localhost:~$ ssh -D 0.0.0.0:8888 user@remoteserver

Now we can configure our browser to connect to the socks proxy. In Firefox select **preferences** | **general** | **network settings**. Add the IP address and the port for the browser to connect to.

![SSH Socks Proxy with DNS](https://hackertarget.com/wp-content/uploads/2018/12/ssh-socks-proxy.png)

Note the option at the bottom of the form to **force browser DNS requests to also go over the socks proxy**. If you are using the proxy to encrypt your web traffic on the local network you will definitely want to select this option so the DNS requests are also tunnelled over the SSH connection.

##### Enable Socks Proxy on Chrome

Using a command line parameter when launching Chrome will use the **socks proxy** and also **tunnel DNS** requests from the browser over the socks5 proxy. Trust but verify, use [tcpdump](https://hackertarget.com/tcpdump-examples/) (tcpdump not port 22) to confirm the DNS requests are no longer visible.

    localhost:~$ google-chrome --proxy-server="socks5://192.168.1.10:8888"

##### Using other applications with the Proxy

Keep in mind that there are many other applications that can utilise a **socks proxy**. A web browser is simply the most popular. Some applications will have configuration options for use of the proxy. Others may need some help by using a helper program that talks the socks protocol. An example of this is [proxychains](https://github.com/haad/proxychains). Using this tool we can for example use Microsoft RDP over the socks proxy.

    localhost:~$ proxychains rdesktop $RemoteWindowsServer

The configuration options for the socks proxy are set in the proxychains configuration file.

Hot Tip: Using remote desktop from Linux to Windows? Try the [FreeRDP](https://github.com/FreeRDP/FreeRDP) client. A more modern implementation than `rdesktop` with much smoother interaction.

##### Use Case for the SSH Socks Proxy

You are in a cafe or hotel having to use the somewhat sketchy WIFI. From our Laptop we run the `ssh` proxy locally and establish an `ssh` tunnel into our home network using a our local Rasberry Pi. Using the browser or other applications configured for the SOCKS proxy we can access any network services on our home network or browse to the Internet via our Home Network Connection. Everything between our Laptop and the Home Server (across the WIFI and Internet to home) is encrypted in the SSH tunnel.

### 2\. SSH Tunnel (port forward)

In its simplest form an SSH tunnel opens a port on your local system that connects through to another port at the other end of the tunnel. 

    localhost:~$ ssh  -L 9999:127.0.0.1:80 user@remoteserver

Lets break down the -L parameter. Think of -L as the Local listening side. So in our example above the port 9999 is listening on localhost and port forwards through to port 80 on **remoteserver**, note that the 127.0.0.1 refers to localhost on the remote server!

Lets take it up a notch. In this following example the port that is listening can be connected to from other hosts on the local network.

    localhost:~$ ssh  -L 0.0.0.0:9999:127.0.0.1:80 user@remoteserver

In these examples the port we are connecting is a listening web server. It could also be a proxy server or any other TCP service.

### 3\. SSH Tunnel Forward to Secondary Remote host

We can use the same options seen above to have the tunnel connect to **another service running on a secondary system** from the remote server.

    localhost:~$ ssh  -L 0.0.0.0:9999:10.10.10.10:80 user@remoteserver

In this example we are forwarding the tunnel from **remoteserver** to the web server running on 10.10.10.10. The traffic from **remoteserver** -\> **10.10.10.10** is **no longer within the ssh tunnel**. The web server on **10.10.10.10** will see **remoteserver** as the source of the web requests.

### 4\. SSH Reverse Tunnel

In this scenario we want to setup a listening port on the remote server that will connect back to a local port on our localhost (or other system).

    localhost:~$ ssh -v -R 0.0.0.0:1999:127.0.0.1:902 192.168.1.100 user@remoteserver

With this ssh session established a connection to the **remoteserver** port **1999** will be forwarded to port **902** on our local client.

### 5\. SSH Reverse Proxy

In this case we are establishing a SOCKS proxy with our ssh connection, however the proxy is listening at the remote server end. With connections to that remote socks proxy now emerging from the tunnel as traffic originating from our localhost. **Requires OpenSSH version 7.6+**.

    localhost:~$ ssh -v -R 0.0.0.0:1999 192.168.1.100 user@remoteserver

##### Troubleshooting Remote SSH Tunnels

If you are having trouble getting the remote SSH options to work, check with `netstat` which interface the listening port is attached too. Even though we have specified **0.0.0.0** in the above examples, if **GatewayPorts** is set to no in the **sshd\_config** then the listener will only bind to **localhost** (127.0.0.1).

**Security Warning**
 Note that when you are opening tunnels and socks proxies you may be exposing internal network resources to untrusted networks (like the Internet!). This can be a serious security risk so ensure you understand what is listening and what it has access too.

### 6\. Establish a VPN over SSH

A common term amongst offensive security folks (pentesters / red teams / etc), is to pivot into a network. Once you have a connection established on one system that system becomes a gateway point for further access to the network. This is known as pivoting and enables lateral movement through the network.

We can use the SSH proxy for this and **proxychains**, however there are some limitations. For example we cannot use raw sockets, so [Nmap](https://hackertarget.com/nmap-online-port-scanner/) `SYN` scans cannot be used to port scan the Internal network.

Using this more advanced VPN option we move the connectivity down to **layer 3**. We can then route traffic through the tunnel using standard network routing.

This technique uses `ssh`, `iptables`, `tun interfaces` and routing.

First we need these options set in the `sshd_config`. Since we are making interface changes on the remote system and the client system, we will **need root privileges on both sides**.

    PermitRootLogin yes
    PermitTunnel yes

Then we will establish our `ssh` connection using the parameter that requests `tun` devices be initialised.

    localhost:~# ssh -v -w any root@remoteserver

Now you should have a `tun` device when you show interfaces (`# ip a`). Next step is to add IP addresses to the tunnel interfaces.

SSH Client Side:

    localhost:~# ip addr add 10.10.10.2/32 peer 10.10.10.10 dev tun0
    localhost:~# ip tun0 up

SSH Server Side:

    remoteserver:~# ip addr add 10.10.10.10/32 peer 10.10.10.2 dev tun0
    remoteserver:~# ip tun0 up

Now we should have a direct route to the other host (`route -n` and `ping 10.10.10.10`).

It is now possible to route any subnet through the other side host. 

    localhost:~# route add -net 10.10.10.0 netmask 255.255.255.0 dev tun0

On the remote side we need to enable `ip_forward` and `iptables`.

    remoteserver:~# echo 1 > /proc/sys/net/ipv4/ip_forward
    remoteserver:~# iptables -t nat -A POSTROUTING -s 10.10.10.2 -o enp7s0 -j MASQUERADE

Boom! **Layer three VPN through an SSH tunnel**. Now that's winning.

Any trouble, try [tcpdump](https://hackertarget.com/tcpdump-examples/) and `ping` to see where its broken. Since we are playing at layer 3 our `icmp` packets should be jumping through that tunnel.

### 7\. Copy your SSH key (ssh-copy-id)

There are multiple ways to achieve this however this command is a shortcut that saves time. What does it actually do? This command replicates what you can also do manually. Copying the `~/.ssh/id_rsa.pub` (or the default) key from your system and adds it to an `~/.ssh/authorized_keys` file on the remote server.

    localhost:~$ ssh-copy-id user@remoteserver

### 8\. Run Command Remotely (non-interactive)

The `ssh` command can be chained to other commands for the usual piping fun. Add the command you want to run on the remote host as a final parameter in quotes.

    localhost:~$ ssh remoteserver "cat /var/log/nginx/access.log" | grep badstuff.php

In this example the `grep` is being performed on the local system after the log file has been pushed across the `ssh` session. If the file is large it would be more efficient to run the `grep` on the remote side by enclosing the pipe and `grep` in the double quotes.

Another example performs the same function as the `ssh-copy-id` short cut in Tip 7.

    localhost:~$ cat ~/.ssh/id_rsa.pub | ssh remoteserver 'cat >> .ssh/authorized_keys'

### 9\. Remote Packet Capture & View in Wireshark

I grabbed this one from our [tcpdump examples](https://hackertarget.com/tcpdump-examples/). Use it for a remote packet capture with the results feeding directly into your local Wireshark GUI.

    :~$ ssh root@remoteserver 'tcpdump -c 1000 -nn -w - not port 22' | wireshark -k -i -

### 10\. SSH Copy Folder from Local to Remote

A neat trick that compresses a folder using `bzip2` (that's the -j in the `tar` command), then extracts the `bzip2` stream on the other side creating a duplicate of the folder on the remote server.

    localhost:~$ tar -cvj /datafolder | ssh remoteserver "tar -xj -C /datafolder"

**Copy remote folder to local tar archive**

To go the other way, copying a remote folder to a local archive. Handy for quick backups of remote resources.

    localhost:~$ ssh user@remoteserver "tar -jcf - /path/to/backup" > dir.tar.bz2

### 11\. Remote GUI Applications with SSH x11 Forwarding

If the client and remote server both have X installed. It is possible to run a GUI command remotely, with the Window appearing on your local desktop. This feature has been around since the beginning of time, but can still be very useful. Run a remote web browser or even the VMWawre Workstation console as I do in this example.

    localhost:~$ ssh -X remoteserver vmware

Requires `X11Forwarding yes` in the `sshd_config`.

### 12\. Copy files remotely with rsync and SSH

Using the `rsync` has many advantages over `scp`, if periodically need to backup a directory, large numbers of files or very large files it should be used. It has the ability to recover from failed transfers and only copy differences between two locations saving bandwidth and time.

The example here uses `gzip` compression (-z) and archive mode (-a) that includes recursive copy.

    :~$ rsync -az /home/testuser/data remoteserver:backup/

### 13\. SSH over Tor Network

The anonymised **Tor Network** can tunnel SSH traffic by using the `torsocks` command. The following command will proxy the `ssh` connection through the Tor network. 

    localhost:~$ torsocks ssh myuntracableuser@remoteserver

[Torsocks](https://github.com/dgoulet/torsocks) will use the **localhost** port **9050** to proxy traffic. As always when using `tor` serious consideration must be taken to understand what traffic is being tunnelled and other **operational security** (opsec) concerns. **Where are your DNS requests going**?

### 14\. SSH to EC2 instance

When using SSH to connect to your EC2 instance within Amazon you will need to use a private key. Download the key (extension .pem) from your Amazon EC2 control panel and change the permissions (`chmod 400 my-ec2-ssh-key.pem`. Keep this key somewhere safe or put it in your `~/.ssh/` folder.

    localhost:~$ ssh -i ~/.ssh/my-ec2-key.pem ubuntu@my-ec2-public

The -i parameter tells the `ssh` client to use this key. This would be an ideal example of where to use the `~/.ssh/config` to configure the use of the key automatically when connecting to the ec2 host.

    Host my-ec2-public
       Hostname ec2???.compute-1.amazonaws.com
       User ubuntu
       IdentityFile ~/.ssh/my-ec2-key.pem

### 15\. Edit text files with VIM over ssh/scp

For all those `vim` users out there, this one can save some time. Using `vim` we can edit files over `scp` with one command. Using this method creates a file in `/tmp` on the local system and then copies it back once we write the file in `vim`.

    localhost:~$ vim scp://user@remoteserver//etc/hosts

Note the format is slightly different to regular `scp`. After the host we have a double `//`. This references the absolute path. A single slash will have a path that is relative to the users home directory.

    **warning** (netrw) cannot determine method (format: protocol://[user@]hostname[:port]/[path])

If you see this error, double check the format of your command. It usually means there is a syntax error.

### 16\. Mount remote SSH location as local folder with SSHFS

Using `sshfs` - an `ssh` filesystem client, we can mount a local directory to a remote location with all file interaction taking place over the encrypted `ssh` session.

    localhost:~$ apt install sshfs

On Ubuntu and Debian based system we install the `sshfs` package and then mount the remote location.

    localhost:~$ sshfs user@remoteserver:/media/data ~/data/

### 17\. SSH Multiplex using ControlPath

By default when you have an existing connection to a remote server with `ssh`, a second connection using `ssh` or `scp` will establish a new session with the overhead of authentication. Using the `ControlPath` options we can have the existing session be used for all subsequent connections. This will speed things up significantly. It is noticeable even on a local network but even more so when connecting to remote resources.

    Host remoteserver
            HostName remoteserver.example.org
            ControlMaster auto
            ControlPath ~/.ssh/control/%r@%h:%p
            ControlPersist 10m

**ControlPath** denotes a socket that is checked by new connections to see if there is an existing `ssh` session that can be used. The **ControlPersist** option above means even after you exit the terminal, the existing session will remain open for 10 minutes, so if you were to reconnect within that time you would use that existing socket. See the `ssh_config` `man` page for more information.

### 18\. Stream Video over SSH using VLC + SFTP

Long time users of `ssh` and `vlc` (Video Lan Client) are not always of aware of this handy option for when you need to watch video over the network. Using the `vlc` option to File | Open Network Stream one can enter the location as a an `sftp://` location. A prompt will appear for authentication details if password is required.

    sftp://remoteserver//media/uploads/myvideo.mkv

### 19\. Two Factor Authentication

Most readers will understand the value in using **Two Factor Authentication**, the same benefits that apply to your banking or Google Account can be applied to your SSH service.

Of course `ssh` comes with a form of Two Factor capability included, that being a passphrase and an SSH key. An advantage of using a hardware based token or the [Google Authenticator App](https://hackertarget.com/ssh-two-factor-google-authenticator/) is the fact that they are generally coming from a second physical device.

See our 8 minute guide to getting started with [Google Authenticator and SSH](https://hackertarget.com/ssh-two-factor-google-authenticator/).

### 20\. Bouncing through jump hosts with ssh and -J

When network segmentation means you are jumping through multiple `ssh` hosts to get to a final destination network or host, this jump host shortcut might be just what you need. **Requires OpenSSH version 7.3+**.

    localhost:~$ ssh -J host1,host2,host3 user@host4.internal

A key thing to understand here is that this is not the same as `ssh host1` then `user@host1:~$ ssh host2`, the `-J` jump parameter uses forwarding trickery so that the **localhost** is establishing the session with the next host in the chain. So our **localhost** is authenticating with **host4** in the above example; meaning our **localhost keys** are used and the session from **localhost** to **host4** is **encrypted end to end**.

To use this ability in the `ssh_config` use the **ProxyJump** configuration option. If you regularly have to jump through multiple hosts; use the config file and your alias to `host4` will save you a lot of time.

### 21\. Block SSH Brute Force Attempts with iptables

Anyone who has managed an **SSH** service on the Internet, and viewed the logs will be aware of the amount of SSH **brute force attempts** that take place every hour of every day. An immediate way to reduce the noise in your logs is to move SSH to a port other than **22**. Make the change in the `sshd_config` file using the **Port \#\#** configuration option.

Using `iptables` we can also block attempts to connect to the port from sources that reach a certain threshold. A way to do this is to use [OSSEC](https://hackertarget.com/enable-ossec-active-response/), as this not only blocks SSH but will also perform a bunch of other host based intrusion detection functions (HIDS).

### 22\. Modify Port Forwarding within a session with ~C

And our final `ssh` example is for modifying port forwarding on the fly within an existing `ssh` session. Picture this example scenario. You are deep in a network; perhaps you have jumped through half a dozen jump hosts and need a local port on your workstation forwarded to Microsoft SMB on the old Windows 2003 system you spotted (ms08-67 anyone?).

After hitting `enter` try typing `~C` in your terminal. This a control escape sequence within the session that allows to make changes to the existing connection.

    localhost:~$ ~C
    ssh> -h
    Commands:
          -L[bind_address:]port:host:hostport    Request local forward
          -R[bind_address:]port:host:hostport    Request remote forward
          -D[bind_address:]port                  Request dynamic forward
          -KL[bind_address:]port                 Cancel local forward
          -KR[bind_address:]port                 Cancel remote forward
          -KD[bind_address:]port                 Cancel dynamic forward
    ssh> -L 1445:remote-win2k3:445
    Forwarding port.

You can see here we have forwarded our local port 1445 to the **Windows 2003** host we found on the internal network. Now launch `msfconsole` and we are good to go (assuming you were planning on **exploiting that host**).

## Wrapping Up

These `ssh` examples, tips and commands are intended to give you a starting point; additional detail on each of the commands and capabilities is available using the `man` pages (`man ssh`, `man ssh_config`, `man sshd_config`).

Being able to reach out and run commands on systems anywhere in the world has always fascinated me. By developing your skills with tools such as `ssh` you will become more productive and effective at whatever game you play.

Thanks for reading and if you have any comments or suggestions please drop me a note using the contact form. Have fun!

Know Your Attack Surface

[USE CASES & MORE INFO](https://hackertarget.com/use-cases/)