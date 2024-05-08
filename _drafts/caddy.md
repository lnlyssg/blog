---
layout: post
title:  "Setting up caddy with AdGuard and Tailscale"
date:   2024-03-31 14:35:41
categories: tips
author: Jim
---
## What?
I have a lot of self-hosted Docker services, some of which I wanted to be available externally and some I wanted to be private. I also wanted nice easy to remember subdomains for each service but I didn't want all of them to also be available on the public internet. Previously I had been using [Cloudflare Zero Trust](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/) to access some of these which required a logon to access but this needed me to sign-in every so often as well as do all the Cloudflare DNS and Tunnels set-up each time I wanted to add a new service, whilst this wasn't difficult it was time-consuming and disjointed. I thought there must be a better way and after giving it some thought I realised I had most of the constituent parts already set-up.  

### Pre-requisites:  
1) A domain  
2) [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome)  
3) [TailScale](https://tailscale.com)  
4) [Caddy](https://caddyserver.com)  

## A domain
You can choose to use something that wouldn't exist on the internet like `mycooldomain.localhost` but I would recommend purchasing your own domain name.  

## AdGuard Home  
There are lots of guides and articles on getting this installed and set-up, in my case I have it running on a Raspberry Pi 2 and Pi 3 at home as well as running externally on an Oracle Cloud server with the excellent [adguardhome-sync](https://github.com/bakito/adguardhome-sync) running to keep them all in sync. We'll return to the AdGuard Home settings further down....  
  
## Tailscale  
I love Tailscale, there's a nice explainer [here](https://www.caseyliss.com/2024/3/27/tailscale) extolling the virtues which says it better than I can and saves me having to say all the same stuff. The two important things to set-up are:
1) The aforementioned AdGuard Home servers must be joined to your tailnet with `--accept-dns=false` (as we don't want to cause a recursive loop.)  
2) MagicDNS must be enabled and the tailscale IPs of the AdGuard Home servers must be entered as Global nameservers on the [DNS settings page](https://login.tailscale.com/admin/dns). That way all tailscale traffic will use your AdGuard servers for DNS.  

## Caddy  
This was the part I struggled the most with, I am no stranger to reverse proxies but I couldn't get Caddy working at all when run in Docker on my Synology, this is because by default the Synology runs a built-in nginx server which is capturing 80 and 443 for Synology services.  Although I could start Caddy up OK I wasn't able to get certs or properly test the webserver. In the end I found a guide and script [here](https://www.smarthomebeginner.com/free-ports-80-and-443-on-synology/) which managed to get me past this blocker[^1]. I chose to use [this Docker image](https://github.com/IAreKyleW00t/docker-caddy-cloudflare) as it had already been built with the Cloudflare DNS plugin I wanted. Once that was all done I created a Caddyfile that looked like this:  

```
{
	acme_dns cloudflare {env.CF_API_TOKEN}
}

subdomain.mydomain.com {
	reverse_proxy http://192.168.1.2:3001
}
```

Almost there....   

a) Set-up a port-forward on your router to send port 80 to the server that's running Caddy (in my cae it was my Synology). That way Caddy can deal with all the DNS challenges for certificates.    
b) Back to AdGuard Home: you need to create a DNS rewrite entry pointing the base domain (e.g. mycooldomain.com) from step 1) to the IP of where Caddy is running.  
c) Start, or restart Caddy, if all has worked as planned it should automatically obtain a cert and begin serving your service originally running at http://192.168.1.2:3001 as subdomain.mydomain.com.  

Now the clever part, because of the fact you also carried out the Tailscale setup this means that no matter where you are in the world, when you are connected to Tailscale you can always access your home services via your own domain name without needing to expose them to the internet. Tailscale does offer this service with their own "random" domains e.g. cat-crocodile.ts.net however I found using my own domain name to be a nicer experience.     

[^1]: Before I managed to get that working I had instead got caddy up and running on a homeserver PC as per the [general Linux instructions](https://caddyserver.com/docs/install#debian-ubuntu-raspbian). Once installed and I'd confirmed it was working by following the basic tutorial I then used xcaddy to add the Cloudflare DNS plugin so that Caddy could automatically issue certs via Cloudflare's DNS. Guides for this can be found [here](https://caddyserver.com/docs/build) note that I used `xcaddy build --with github.com/caddy-dns/cloudflare` followed by the steps at the bottom of the article to divert dpkg. I also needed a Cloudflare API key as per the guide [here](https://caddyserver.com/docs/modules/dns.providers.cloudflare).