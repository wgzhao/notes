---
title: TCPDump Capture HTTP GET/POST requests
description: TCPDUMP is a swiss army knife for all the administrators and developers when it comes to troubleshooting. This post is written for the people who work in middleware technologies.
tags: ["tcpdump"]
---



# TCPDump Capture HTTP GET/POST requests

[Source](https://www.middlewareinventory.com/blog/tcpdump-capture-http-get-post-requests-apache-weblogic-websphere/ "Permalink to TCPDump Capture HTTP GET/POST requests | Devops Junction")

TCPDUMP is a [swiss army knife](https://en.wikipedia.org/wiki/Swiss_Army_knife) for all the administrators and developers when it comes to troubleshooting. This post is written for the people who work in middleware technologies. Web servers such as Apache, NGINX, Oracle HTTP, IHS web servers and application servers such as Weblogic, Websphere, Tomcat, Jboss

Consider yourself in any of the following scenarios

1. You want to monitor the **traffic inflow and outflow** of Apache httpd server on any specific port like port 80 or 443
2. You have to **track the HTTP calls between web and application servers** (or) to make sure that proxy is working fine.
3. When you want to examine the presence and values of specific headers like 

      ```
    X-Forwarded-For
    X-Forwarded-Proto
    X-Forwarded-Port
    X-Forwarded-Host
    X-Frame-Options
    X-Forwarded-By
    ETag
    ```
4. When you want to review the **Cookies** being placed in the request and response
5. when you want to review the data which is being posted to the server in the POST method like 

      ```
    Content-type: multipart/form-data;  
    Content-Type: application/x-www-form-urlencoded 
    Content-Type: application/json 
    Content-Type: text/plain
    ```
6. To track the incoming web service call, made using SOAP UI (or) any web service client.



## So, How do you do that?

**The answer is TCPDUMP. ** TCPDUMP is mostly misconceived as a network engineer's subject and it displays some incomprehensible binary data that none of us could understand.

With proper tools and little knowledge about protocols, anyone can easily make use of it and feel the magic lies within.

> Be informed that the industry standards have changed for good and the **HTTPS is becoming a basic requirement** for all webservices and websites. So it might make your troubleshooting little hard, since the packets are encrypted. **There is a solution to decrypt HTTPS traffic**
> 
> Refer my another article on [How to decrypt HTTPS traffic to see headers and request/response content](https://www.middlewareinventory.com/blog/how-to-decrypt-https-traffic/)

## The Objective

In this post, we are going to see how middleware administrators (or) developers could use tcpdump to accomplish their troubleshooting drama.

We are going to discuss the following items , practically as much as possible.

* How to monitor/track HTTP and HTTPS calls with tcpdump in weblogic,WebSphere,tomcat application servers and web servers like Apache **which runs on LINUX platform**
* How to tamper and read the incoming and outgoing HTTP traffic to our applications deployed in weblogic
* How to dig into the incoming (or) outgoing HTTP traffic and take a look at the concrete elements of HTTP protocol such as** headers, cookies, request body** as they gets transmitted.
* How to intercept the HTTP traffic initiated from the browser (or) SOAP UI to application server and sneak a peek into the content like **Request Body** like** XML,JSON **and **Username** and **Password** etc.
* How to monitor the incoming SOAP web service request body (or) request XML

* More practical examples on how to use TCPDUMP to analyze the HTTP traffic

## Before we proceed

Some basics about how to run tcpdump in your server in the right way.

**Make sure tcpdump is installed and configured properly**

    [root@mwiws01 ~]# tcpdump – version
    tcpdump version 4.9.2
    libpcap version 1.5.3
    OpenSSL 1.0.2k-fips 26 Jan 2017

**Use the right interface name (or) use any in the interface name.**

To Get the interface name of your IP which you need to specify it in the tcpdump command. you can execut the command `ifconfig` (or) `ip a`

In my case, My web server IP is `192.168.31.76` so I should pick and use the interface name of the same `wlp0s20f3`

```shell
$ ifconfig wlp0s20f3
wlp0s20f3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.31.76  netmask 255.255.255.0  broadcast 192.168.31.255
        inet6 2409:8a50:693:930:da38:cc7c:4d8f:8d93  prefixlen 64  scopeid 0x0<global>
        inet6 2409:8a50:693:930::d02  prefixlen 128  scopeid 0x0<global>
        inet6 fe80::cbf:9e72:4be8:6b73  prefixlen 64  scopeid 0x20<link>
        inet6 2409:8a50:693:930:72db:3ee4:8539:332d  prefixlen 64  scopeid 0x0<global>
        ether 6c:94:66:4b:c3:cd  txqueuelen 1000  
        RX packets 18758580  bytes 3461986404 (3.4 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2564530  bytes 960162238 (960.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Once you found your interface. In the requests you are going to track. you must mention the interface name

    tcpdump -i wlp0s20f3

you can optionally monitor all the available interfaces by mentioning

    tcpdump -i any

Now we are All ready to Play.

## tcpdump Examples

TCPDUMP does the same job irrespective to what technology (or) server you are using it for. In other words, if you would like capture HTTP calls for Apache. You mostly going to be using the port 80 or 443.

If you would like to capture the traffic of weblogic (or) Websphere or any application servers. All you have to do is change the port number in which your application server is listening

Though we have given examples for both web and application servers here. All you should be aware of is that. There is nothing specific to technology. We just change the port number (or) interface. That's all.

### How to capture All incoming HTTP GET traffic (or) requests

```shell
sudo tcpdump -i wlp0s20f3 -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

**Explanation:-**

**`tcp[((tcp[12:1] & 0xf0) >> 2):4]`** first determines the location of the bytes we are interested in (after the TCP header) and then selects the 4 bytes we wish to match against.

To know more about how this segment syntax has been derived. refer this [link](https://security.stackexchange.com/questions/121011/wireshark-tcp-filter-tcptcp121-0xf0-24)

> Here `0x47455420` depicts the ASCII value of characters `'G' 'E' 'T' ' '`

| Character | ASCII Value |
| --------- | ----------- |
| G         | 47          |
| E         | 45          |
| T         | 54          |
| Space     | 20          |

> Refer this [ASCII map table](https://www.middlewareinventory.com/ascii-table/) for more reference

### How to capture All incoming **HTTP POST** requests

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354'
```

Here `0x504F5354` represents the ASCII value of `'P' 'O' 'S' 'T'`

**Sample Output**

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354'

tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlp0s20f3, link-type EN10MB (Ethernet), snapshot length 262144 bytes

13:48:21.019415 IP 192.168.31.205.54511 > wgzhao-laptop.http: Flags [P.], seq 952980253:952980331, ack 3271140678, win 2058, options [nop,nop,TS val 2736792397 ecr 1034997687], length 78: HTTP: POST / HTTP/1.1
E.....@.@.z........L...P8.S....F...
<......
. #M=...POST / HTTP/1.1
Host: wgzhao-laptop
User-Agent: curl/7.86.0
Accept: */*
```

### How to capture only **HTTP GET** requests Incoming to port 80 ( Apache/NGINX)

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

**Sample Output**

    tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
    
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on wlp0s20f3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    13:52:30.348948 IP 192.168.31.205.57439 > wgzhao-laptop.http: Flags [P.], seq 2517601825:2517601912, ack 3234167917, win 2058, options [nop,nop,TS val 3003354219 ecr 1035247016], length 87: HTTP: GET /index.html HTTP/1.1
    E.....@.@.z........L._.P...!..|m...
    px.....
    ...k=...GET /index.html HTTP/1.1
    Host: wgzhao-laptop
    User-Agent: curl/7.86.0
    Accept: */*
    

### How to capture only **HTTP** **POST** requests Incoming to port 80 ( Apache/NGINX)

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354'
```

**Sample Output**

    tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354'
    
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on wlp0s20f3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    
    13:54:03.126400 IP 192.168.31.205.58596 > wgzhao-laptop.http: Flags [P.], seq 207644020:207644108, ack 3869953719, win 2058, options [nop,nop,TS val 1754598356 ecr 1035339794], length 88: HTTP: POST /index.html HTTP/1.1
    E.....@.@.z........L...P.`et.......
    .b.....
    h...=...POST /index.html HTTP/1.1
    Host: wgzhao-laptop
    User-Agent: curl/7.86.0
    Accept: */*

### How to capture only HTTP GET calls Incoming to port 443 ( Apache/NGINX)

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 443 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

### How to capture only HTTP POST calls Incoming to port 443 ( Apache/NGINX)

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 443 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354'
```

### How to capture both **HTTP GET (or) POST** **Incoming** calls to **port 80 (or) 443** ( Apache/NGINX) Originating from 192.168.31.205 Host**
```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 or tcp dst port 443 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354 and host 192.168.31.205'

tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlp0s20f3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
13:59:31.404400 IP 192.168.31.205.62507 > wgzhao-laptop.http: Flags [P.], seq 3732089400:3732089488, ack 1222075730, win 2058, options [nop,nop,TS val 783516821 ecr 1035668072], length 88: HTTP: POST /index.html HTTP/1.1
E.....@.@.z........L.+.P.s*8H.eR...
.C.....
....=.
hPOST /index.html HTTP/1.1
Host: wgzhao-laptop
User-Agent: curl/7.86.0
Accept: */*


13:59:36.729382 IP 192.168.31.205.62568 > wgzhao-laptop.http: Flags [P.], seq 1815931667:1815931754, ack 1237652196, win 2058, options [nop,nop,TS val 3905580542 ecr 1035673396], length 87: HTTP: GET /index.html HTTP/1.1
E.....@.@.z........L.h.Pl<..I......
p......
..m.=..4GET /index.html HTTP/1.1
Host: wgzhao-laptop
User-Agent: curl/7.86.0
Accept: */*

```

### How to capture a Complete HTTP Transmission incoming and outgoing GET and POST

Let's suppose I access a page hosted in `192.168.31.76` web server from my base machine with ip address `192.168.31.205.` using both GET and POST methods. How do we track the HTTP request and response between the server and the client

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x3C21444F and host 192.168.31.205'
```

Here additionally, we are using two more ASCII values to capture the outgoing HTML files and HTTP calls they are as follows.

`0x3C21444F` represents the ASCII value of `'<' 'D' 'O' 'C'` this is to capture the outgoing HTML file

`0x48545450` represents the ASCII value of `'H' 'T' 'T' 'P'` this is to capture the outgoing HTTP traffic (HTTP response)

**Trying with GET**

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x3C21444F and host 192.168.31.205'

tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlp0s20f3, link-type EN10MB (Ethernet), snapshot length 262144 bytes


14:06:37.059470 IP 192.168.31.205.51047 > wgzhao-laptop.http: Flags [P.], seq 4216390795:4216390882, ack 769730052, win 2058, options [nop,nop,TS val 905239047 ecr 1036093728], length 87: HTTP: GET /index.html HTTP/1.1
E.....@.@.z........L.g.P.Q..-.&....
.......
5...=.. GET /index.html HTTP/1.1
Host: wgzhao-laptop
User-Agent: curl/7.86.0
Accept: */*


14:06:37.059885 IP wgzhao-laptop.http > 192.168.31.205.51047: Flags [P.], seq 1:863, ack 87, win 128, options [nop,nop,TS val 1036093732 ecr 905239047], length 862: HTTP: HTTP/1.1 200 OK
E.....@.@._!...L.....P.g-.&..Q.............
=..$5...HTTP/1.1 200 OK
Server: nginx/1.22.0 (Ubuntu)
Date: Wed, 22 Mar 2023 06:06:37 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 22 Mar 2023 05:44:43 GMT
Connection: keep-alive
ETag: "641a95cb-267"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


```



**Trying with POST**

```shell
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x3C21444F and host 192.168.31.205'

14:09:16.523346 IP 192.168.31.205.52920 > wgzhao-laptop.http: Flags [P.], seq 1264301850:1264301928, ack 2774851506, win 2058, options [nop,nop,TS val 1964671602 ecr 1036253191], length 78: HTTP: POST / HTTP/1.1
E.....@.@.z........L...PK[...d.....
U......
u..r=...POST / HTTP/1.1
Host: wgzhao-laptop
User-Agent: curl/7.86.0
Accept: */*


14:09:16.523609 IP wgzhao-laptop.http > 192.168.31.205.52920: Flags [P.], seq 1:333, ack 78, win 128, options [nop,nop,TS val 1036253196 ecr 1964671602], length 332: HTTP: HTTP/1.1 405 Not Allowed
E....K@.@......L.....P...d..K[.h...........
=...u..rHTTP/1.1 405 Not Allowed
Server: nginx/1.22.0 (Ubuntu)
Date: Wed, 22 Mar 2023 06:09:16 GMT
Content-Type: text/html
Content-Length: 166
Connection: keep-alive

<html>
<head><title>405 Not Allowed</title></head>
<body>
<center><h1>405 Not Allowed</h1></center>
<hr><center>nginx/1.22.0 (Ubuntu)</center>
</body>
</html>
```

In the preceding illustrations, you can see the output of tcpdump command  shows the **REQUEST and RESPONSE** data along with the **HTML code**. 

### How to monitor all the incoming HTTP Request URL's (POST or GET)

```bash
tcpdump -i wlp0s20f3 -s 0 -v -n -l | egrep -i "POST /|GET /|Host:"
```

### How to capture HTTP Passwords in POST Requests

```bash
tcpdump -i wlp0s20f3 -s 0 -A -n -l | egrep -i "POST /|pwd=|passwd=|password=|Host:"
```

### How to capture the Cookies from Server and from Client ( Request & Response)

```bash
tcpdump -i wlp0s20f3 -nn -A -s0 -l | egrep -i 'Set-Cookie|Host:|Cookie:'
```

## How to Filter HTTP User Agents

Extract HTTP User Agent from HTTP request header.

```bash
tcpdump -vvAls0 | grep 'User-Agent:'
```

## How to capture the HTTP packets being transmitted between Webserver and Application server both GET & POST?

In cases, where you have check the HTTP traffic between webserver and application server. you can use tcpdump to diagnose and troubleshoot the issue. It will be helpful for many middleware administrators.

Let's suppose this is your environment

| Backend Server IP | 192.168.31.76  |
| ----------------- | -------------- |
| Nginx Server IP   | 192.168.31.205 |
| Nginx Server Port | 8080           |

You can run the following command at the application server to accomplish your requirement

```bash
tcpdump -i any -s 0 -A 'tcp dst port 8080 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x3C21444F and host 192.168.31.76'
```

### How to capture a Complete HTTP Transmission with TCPDUMP ( GET & POST) on specific port

All the examples we have given above can be used for weblogic with just a little change in port number. However. we are giving an example here.

Here we are going to make a call from the client with up `192.168.31.205` to our  Application using GET and POST methods and capture the HTTP traffic data at the server end

```bash
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 18001 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x3C21444F and host 192.168.31.76'
```

Sample Output

**Request from **`192.168.31.205` **using **`curl -v`

```bash
 curl -v -XPOST -d "hello"  http://localhost:8080/server
 Unnecessary use of -X or --request, POST is already inferred.
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> POST /server HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.86.0
> Accept: */*
> Content-Length: 5
> Content-Type: application/x-www-form-urlencoded
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.23.3
< Date: Wed, 22 Mar 2023 06:34:08 GMT
< Transfer-Encoding: chunked
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
Received POST request with data: hello%
```

**Response**

```bash
tcpdump -i wlp0s20f3 -s 0 -A 'tcp dst port 18001 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450 or tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x3C21444F and host 192.168.31.76'

tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlp0s20f3, link-type EN10MB (Ethernet), snapshot length 262144 bytes

14:35:08.235142 IP 192.168.31.205.54678 > wgzhao-laptop.8888: Flags [P.], seq 3387431051:3387431226, ack 2818253595, win 2058, options [nop,nop,TS val 1305195002 ecr 1037804904], length 175
E.....@.@.y........L..".......#....
.......
M...=..hPOST / HTTP/1.0
Host: 192.168.31.76:8888
Connection: close
Content-Length: 5
User-Agent: curl/7.86.0
Accept: */*
Content-Type: application/x-www-form-urlencoded

hello
14:35:08.235550 IP wgzhao-laptop.8888 > 192.168.31.205.54678: Flags [P.], seq 1:95, ack 175, win 131, options [nop,nop,TS val 1037804908 ecr 1305195002], length 94
E...g.@.@..:...L....".....#....:...........
=..lM...HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.10.7
Date: Wed, 22 Mar 2023 06:35:08 GMT

14:35:26.047089 IP blackcat.canonical.com.http > wgzhao-laptop.24674: Flags [P.], seq 1899838163:1899838310, ack 531714235, win 453, options [nop,nop,TS val 302839182 ecr 3954227041], length 147: HTTP: HTTP/1.1 204 No Content
E.....@.@..a[.[1...L.P`bq=:...P.....1......
.......aHTTP/1.1 204 No Content
server: nginx/1.14.0 (Ubuntu)
date: Wed, 22 Mar 2023 06:35:25 GMT
x-networkmanager-status: online
connection: close
```

### How to record a TCPDUMP Session (or) Capture packets with tcpdump

To record the tcpdump session, you can use the following command

**Note:** here I have used any as an interface to capture all the packets across all the channels/interfaces available in my server

```bash
tcpdump -i any -s 0 -X -w /tmp/tcpdump.pcap
```

`pcap` is a widely accepted extension for the tcpdump output.

### How to read the TCPDUMP recorded session (or) packet capture - pcap file

```bash
tcpdump -A -r /tmp/tcpdump.pcap
```

This way. you would be able to read the recorded session and it will offer more information than the ASCII matching commands.

You can also use commands like `less` for better readability and search.

```bash
tcpdump -A -r /tmp/tcpdump.pcap|less
```
