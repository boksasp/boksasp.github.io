---
layout: post
title:  "Hosting multiple services on one home server with the help of Pi-hole"
date:   2022-01-26 22:20:00 +0100
tags: pi-hole nginx raspberrypi dns nodejs
---
The purpose of this project is to verify that I can host multiple web sites/services on the same machine,
 where each site/service has a unique domain name.

I don't want to add host file entries for all my services on all my devices, so it became obvious I needed
 a local DNS server where I could control DNS records myself. I have an old Raspberry Pi (Model B, 512MB RAM)
 which I installed [Pi-hole](https://pi-hole.net/) on. 

For serving the different services, I decided to use the Nginx web server. By adding a new server block for each service,
 I'm able to forward requests for specific domains to a specific address and port with `proxy_pass`. If you're using the Apache web server,
 the same can be achieved by using virtual hosts.

I tested the whole thing with two "hello-world" type node apps running on different ports.

Summary:  
- **Pi-hole** for local DNS resolution
- **Nginx** as web server with proxy pass to multiple services running on the same machine
- **NodeJS apps** for having content to access when testing

## Details

### Pi-hole
I'm using a default installation of Pi-hole. I turned all the privacy settings to the strictest setting,
 as I don't want to monitor all my family's network activities (Settings -> Privacy -> Anonymous mode).

The only other change I did to Pi-hole was to add two local DNS records, each pointing to my Windows desktop.

1. test1.home -> 192.168.86.213
2. test2.home -> 192.168.86.213

**Main router DNS**  
For all the devices on my home network to go through Pi-hole, I set the primary DNS on my Google Wifi router
 to my Raspberry Pi address. I also gave the Raspberry Pi a static IP.

I'm so impressed by the Raspberry Pi handling all that traffic without noticably slowing down the network 🤩

### Nginx
I ran the web server on my Windows desktop computer for testing purposes.  
Standard installation of Nginx for Windows. It doesn't run as a service, so I have to start it through the nginx executable provided with the install.

To `nginx.conf` I added a server block for each of the node apps I was to run on the server:

```conf
# nginx.conf

...

server {
    listen 80;
    server_name test1.home;

    location / {
        proxy_pass http://127.0.0.1:3031;
    }
}

server {
    listen 80;
    server_name test2.home;

    location / {
        proxy_pass http://127.0.0.1:3032;
    }
}

```

### Node apps
I started up two node apps just so I could have something to test against. These apps are running on the same Windows desktop that Nginx is running on.

```javascript
const http = require('http');

const host = '127.0.0.1';
const port = 3031; // change port for the other node app

const server = http.createServer((req, res) => {
  res.end('Hello test 1!'); // change number for the other node app
});

server.listen(port, host, () => {
   console.log('Web server running at http://%s:%s',host,port );
});
```

And with the node apps and nginx running, I was able to access both web apps from any device on my local network. One on `http://test1.home` and the other on `http://test2.home` 🚀

![test1.home in browser from device on network](/assets/2022-01-26/test1.home.png)
![test2.home in browser from device on network](/assets/2022-01-26/test2.home.png)
# Notes

❗ **Troubleshooting notes on Local DNS Records**  
The first DNS record I tried was `test1.local`. It worked like a charm when I was testing it from my Windows computer, where the apps where running.

But when testing how it resolved over the network, I used my MacBook. Turns out Mac OS doesn't use the network assigned DNS server when
 resolving `*.local` domains, so I just got a DNS error. It turns out Apple devices
 use `.local` for `bonjour`, an implementation of zero-configuration networking (zeroconf) used for local network service discovery and _name resolution_.

Bad idea to use that as a custom local domain name with a lot of Apple devices...

Zeroconf isn't an Apple-only concept, so it's a bad idea for anyone.  
More info here [https://en.wikipedia.org/wiki/.local](https://en.wikipedia.org/wiki/.local)
