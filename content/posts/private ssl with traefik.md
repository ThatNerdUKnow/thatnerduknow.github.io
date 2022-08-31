+++
title = "Securing applications on my home network"
date = "2022-08-28T06:49:44-05:00"
author = "Brandon Piña"
authorTwitter = "" #do not include @
cover = ""
tags = ["project","homelab"]
keywords = ["networking", "dns","ssl","traefik"]
showFullContent = false
hideComments = false
draft = true
+++

I have a few applications on my network that I would prefer that not just anyone access as they provide basically unhindered access to the entire directory of my file server. The culprits being containers `makemkv`, `mkvtoolnix`, `krusader`, and `duplicati`. Usually, homelabbers secure their web applications either by using the letsencrypt certificate authority and certbot as an ACME client or by issuing a long lived static certificate which I don't want to use because setting up a private certificate authority is more difficult and I like to feel smart. There's a problem though if you want to use letsencrypt, your ACME client needs to be visible to letsencrypt via the requested domain name. This verification process is done through what are called *challenges* You can read more about all the challenges and how they work [here](https://letsencrypt.org/docs/challenge-types/) but for my purposes, I only really need the http-01 challenge since it's the easiest to set up

## Let the suffering begin
To set up a private certificate authority I'll need a few things
* A certificate authority
* A dns server on your LAN
* A reverse proxy
    * An ACME Client (certbot is a standalone program but some reverse proxies include it)

### Reverse Proxy
Originally I planned to use `nginxproxymanager` as my reverse proxy because it included certbot but soon discovered that it only supports letsencrypt as a certificate authority. When I went looking for alternatives I found `traefik`. Now I've seen traefik out and about and didn't think very much of it before now but man, does this guy have lots of cool features
* Service Discovery
    * traefik can read `docker.sock` and can detect services running on your docker host
* Dynamic Routing
    * routing configuration can be done via container labels, so when you're deploying your container, you don't have to go to traefik to set up routing rules
* Dynamic Configuration
    * in the same vein as routing configuration, you can configure quite literally everything about how traefik handles your container, importantly, even SSL
* It has an ACME client built in (and it's BYOCA ***Bring your own Certificate Authority***)

Where have you been all my life!

We need to configure traefik to sit on both a protected docker network (which, importantly we can't access from LAN) and also to the LAN network  

Let's take a look at the LAN configuration
```json
[
    {
        "Name": "br0",
        "Id": "66228ea7167460c10d229f8c021c3214e8f15de364fee48d07bb45cd68bf47e0",
        "Created": "2022-08-06T16:31:08.557703061-05:00",
        "Scope": "local",
        "Driver": "macvlan",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.0.0/24",
                    "Gateway": "192.168.0.1",
                    "AuxiliaryAddresses": {
                        "server": "192.168.0.166"
                    }
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            # Truncated for brevity
        },
        "Options": {
            "parent": "br0"
        },
        "Labels": {}
    }
]
```
This might be intimidating at first, but what we're looking at here is the configuration for a docker network. The network which we're looking at is a `macvlan` network. What this means is that each container can get its own mac address and therefore its own IP address on the LAN network. Importantly, we also have to assign the network to the same subnet that our physical devices live. This network is typically `192.168.0.0/24`

Now let's look at the protected network
```json
[
    {
        "Name": "protected",
        "Id": "1b446fb6b0c746d3300550726f5683f8dfe460ad591a6fc9178041d67329b63d",
        "Created": "2021-11-08T19:20:53.414390514-06:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            # Truncated for brevity
        },
        "Options": {},
        "Labels": {}
    }
]
```
Note that the driver is now a `bridge` network and that the subnet is `172.18.0.0/16`. Any containers connected to this network should be inaccessible via LAN.

We're now ready to deploy Traefik. Here's a docker-compose file
```yml
networks:
  br0:
    external: true
  protected:
    external: true

services:
  proxy:
    image: traefik:latest
    ports: 
      - 80:80
      - 443:443
      - 8080:8080
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /mnt/user/appdata/traefik:/etc/traefik
    networks:
      protected:
      br0:
        ipv4_address: 192.168.0.5
    dns:
      - 192.168.0.2
    labels:
      - traefik.enable=false
```
Lot's of things going on here so I'll cover each one. 
* Networking
    * Under networks we're importing both our `macvlan` and private `bridge` network. 
    * Under services, For the proxy service we're exposing ports 80,443, and 8080. 
    * Under `services.proxy.networks` we see that we're giving traefik access to both the protected network and our LAN network. 
        * Note that we're assigning traefik a static IP address on LAN
        * We do not require a static ip address on the protected network, so we'll leave the configuration for `protected` blank
    * Note that we're assigning the dns resolver for our traefik container to `192.168.0.2`
        * This is important so that traefik can find the certificate authority we will set up later
* Volumes
    * We're binding `/etc/traefik` to our host's filesystem which is where we can make changes to the static traefik configuration
    * We're also binding to `/var/run/docker.sock`. This is where traefik gets its service discovery magick. 
        * I cannot overstate to **ONLY PROVIDE READ ONLY ACCESS TO docker.sock**. If someone were to manage to get into our traefik container, they would be able to have access to the docker.sock of our host system. This means that they can kill/start containers at will. If we provide read only access to docker.sock we can mitigate the severity of a breach of our traefik container. Even then, if you're running containers that use environment variables for secrets, they would be able to use docker inspect to see those variables. For this reason I would advise against exposing a traefik container that has access to docker.sock to the internet. You have been warned.

### DNS Server
Now for the DNS. I chose `bind9` as my DNS because it's one the first things that come up when you google "docker dns server" First, we need to decide what services will live on what IP addresses. Traefik is already assigned a static ip address of `192.168.0.5` we also need to decide which address our certificate authority will use. This isn't strictly necessary but when we're setting up SSL on containers it'll be nicer to use a name instead of an IP address. Let's use `192.168.0.4`

Once we stand up a bind9 instance and all logged in we're greeted by the Admin Panel. From here we'll create a new master zone. I've already created mine but you can name it whatever you wish.
![Bind Admin Panel](/private-ssl-with-traefik/bind-admin-global.PNG)

 When we click on our newly created zone, we're greeted by the zone settings. From here we're going to click on 'Address'
![Zone Admin Panel](/private-ssl-with-traefik/bind-admin-zone.PNG)

Finally, we're in the menu to edit Address records or "A records". This is how our dns server knows which server to point to when we request our proxy. Here you can see I've already added some records. The records to note are `*.dogma.net`, and `dogma.net` we're binding both those addresses to `192.168.0.5` which is the static ip address we set for our traefik container. Next we bind `ca.dogma.net` to `192.168.0.4` which should map to our certificate authority that we'll stand up later
![Zone Address Records](/private-ssl-with-traefik/bind-admin-arecords.PNG)

Once we're done we apply the configuration by clicking on the "apply configuration" button on the top right of the page. To test out that 

## Network Diagram
![Network layout diagram](/private-ssl-with-traefik/traefik-network-diagram.png)
~~Graphic Design is my passion~~