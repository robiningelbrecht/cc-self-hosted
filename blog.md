
> medium-to-markdown@0.0.3 convert
> node index.js

### A tour through my self-hosted setup

Back in November 2024 I came across [r/selfhosted](https://www.reddit.com/r/selfhosted/), a subreddit to share, discuss and discover self-hosted alternatives to our favorite web apps and online tools. I was hooked instantly.

This post walks you through my current self-hosted setup: how it runs, how I run updates and how I (try to) keep it all from catching fire.

![](https://cdn-images-1.medium.com/fit/c/800/533/1*4EOaw3jbke-vGNqkyU2jHg.png)✍︎Intro image

The hardware
------------

No self-hosting setup is complete without the right hardware. After comparing a bunch of options, I knew I wanted an affordable mini PC that could run Ubuntu Server reliably. That search led me to the [Beelink EQR5 MINI PC AMD Ryzen](https://www.bee-link.com/products/beelink-eqr5).

![](https://cdn-images-1.medium.com/fit/c/480/480/1*YVarvlmMccQ_MjJirs8LWg.png)✍︎Beelink EQR5 MINI PC AMD Ryzen 32GB, 500GB SSD

For the routing layer, I didn’t bother replacing the hardware, my ISP’s default router does the job just fine. It gives me full control over DNS and DHCP, which is all I need.

The hardware cost me exactly $319.

Creating the proper accounts
----------------------------

To get things rolling, I set up accounts with both **[Tailscale](https://tailscale.com/)** and **[Cloudflare](https://www.cloudflare.com/)**. They each offerfree tiers, and everything in this setup fits comfortably within those limits, so there’s no need to spend a cent.

### Tailscale

> Securely connect to anything on the internet

I created a Tailscale account to handle VPN access. No need to configure anything at this stage, just sign up and be done with it.

### Cloudflare

> Protect everything you connect to the Internet

For Cloudflare, I updated my domain registrar’s default nameservers to point to Cloudflare’s. With that in place, I left the rest of the configuration for later when we start wiring up DNS and proxies.

Before installing any apps
--------------------------

Before diving into the fun part, running apps and containers, I first wanted a solid foundation. So after wiping the Beelink and installing Ubuntu Server, I spent some time getting my router properly configured.

### Configuring my router

I set up [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) reservations for the devices on my network so they always receive a predictable IP address. This makes everything much easier to manage later on. I created DHCP entires for:

*   My Beelink server
*   My network printer
*   A [Raspberry Pi](https://www.raspberrypi.com/) I purchased a few years back

### Configuring Ubuntu server

With the router sorted out, it was time to prepare the server itself.

I started by installing [Docker](https://www.docker.com/) and ensuring its system service is set to start automatically on boot.

```
\# Install Docker
sudo apt update
sudo apt upgrade -y
curl -sSL https://get.docker.com | sh
# Add current user to the docker group
sudo usermod -aG docker $USER
logout
# Run containers on boot
sudo systemctl enable docker
```

Next, I added my first device to Tailscale and installed the Tailscale client on the server.

![](https://cdn-images-1.medium.com/fit/c/800/545/1*E45scn-gt-c3ZDbcLs5dPw.png)✍︎Adding a Linux device

After that, I headed over to Cloudflare and configured my domain (which I had already purchased) so that all subdomains pointed to my Tailscale device’s IP address, my Ubuntu server:

![](https://cdn-images-1.medium.com/fit/c/800/310/1*YSk0MiwA6qViltwTqrF-kA.png)✍︎Configure DNS A records in Cloudflare

At this point, the server was fully reachable over the VPN and ready for the next steps.

Traefik, the reverse proxy I fell in love with
----------------------------------------------

> A reverse proxy is an intermediary server that receives incoming network requests and routes them to the correct backend service.

I wanted to access all my self-hosted services through subdomains rather than a root domain with messy port numbers. That’s where **[Traefik](https://doc.traefik.io/)** comes in. Traefik lets you reverse-proxy Docker containers simply by adding a few labels to them, no complicated configs needed. It takes care of all the heavy lifting behind the scenes.

```
services:
  core:
    image: ghcr.io/a-cool-docker-image
    restart: unless-stopped
    ports:
      - 8080:8080
    labels:
      - traefik.enable=true
      - traefik.http.routers.app-name.rule=Host(\`subdomain.root.tld\`)
    networks:
      - traefik\_default
networks:
  traefik\_default:
    external: true
```

The configuration above tells Traefik to route all traffic hitting `https://subdomain.root.tld` directly to that container.

### Securing Everything with HTTPS

Obviously, I wanted all my services to be served over HTTPS. To handle this, I used Traefik together with Cloudflare’s certificate resolver. I generated an API key in Cloudflare so Traefik could automatically request and renew TLS certificates.

![](https://cdn-images-1.medium.com/fit/c/800/348/1*e7hgXNnb7IQq-O7BC9gbcA.png)✍︎Creating an API token to be able to create certificates trough Traefik

The final step is to reference the Cloudflare certificate resolver and the API key in the Traefik Docker container.

```
services:
  # Redacted version
  traefik:
    image: traefik:v3.2
    container\_name: traefik
    restart: unless-stopped
    privileged: true
    command:
      - --entrypoints.websecure.http.tls=true
      - --entrypoints.websecure.http.tls.certResolver=dns-cloudflare
      - --entrypoints.websecure.http.tls.domains\[0\].sans=\*.root.tld
      - --certificatesresolvers.dns-cloudflare.acme.dnschallenge=true
      - --certificatesresolvers.dns-cloudflare.acme.dnschallenge.provider=cloudflare
      - --certificatesresolvers.dns-cloudflare.acme.dnschallenge.delayBeforeCheck=10
      - --certificatesresolvers.dns-cloudflare.acme.storage=storage/acme.json
    environment:
      - CLOUDFLARE\_DNS\_API\_TOKEN=${CLOUDFLARE\_DNS\_API\_TOKEN}
networks: {}
```

Managing all my containers
--------------------------

Now that the essentials were in place, I wanted a clean and reliable way to manage all my (future) apps and Docker containers. After a bit of research, I landed on **[Komodo 🦎](https://github.com/moghtech/komodo)** to handle configuration, building, and updates.

> A tool to build and deploy software on many servers

![](https://cdn-images-1.medium.com/fit/c/800/444/1*W7GF0iBxGFO2BDCrfc0dqQ.png)✍︎Overview of deployed Docker containers

Documentation is key
--------------------

As a developer, I know how crucial documentation is, yet it’s often overlooked. This time, I decided to do things differently and start documenting everything from the very beginning. One of the first apps I installed was **[wiki.js](https://github.com/requarks/wiki)**, a modern and powerful wiki app. It would serve as my guide and go-to reference if my server ever broke down and I needed to reconfigure everything.

I came up with a sensible structure to categorize all my notes:

![](https://cdn-images-1.medium.com/fit/c/800/624/1*BjUuTOePNJ4mWVivl_vtjA.png)✍︎Menu structure of my internal wiki

Wiki.js also lets you back up all your content to private [Git](https://en.wikipedia.org/wiki/Git) repositories, which is exactly what I did. That way, if my server ever failed, I’d still have a Markdown version of all my documentation, ready to be imported into a new Wiki.js instance.

Organizing my apps in one place
-------------------------------

Next, I wanted an app that could serve as a central homepage for all the other apps I was running, a dashboard of sorts. There are plenty of dashboard apps out there, but I decided to go with **[Homepage](https://github.com/gethomepage/homepage)**.

> A highly customizable homepage (or startpage / application dashboard) with Docker and service API integrations.

The main reason I chose Homepage is that it lets you configure entries through Docker labels. That means I don’t need to maintain a separate configuration file for the dashboard

```
services:
  core:
    image: ghcr.io/a-cool-docker-image
    restart: unless-stopped
    ports:
      - 8080:8080
    labels:
      - homepage.group=Misc
      - homepage.name=Stirling PDF
      - homepage.href=https://stirlingpdf.domain.tld
      - homepage.icon=sh-stirling-pdf.png
      - homepage.description=Locally hosted app that allows you to perform various operations on PDF files
```![](https://cdn-images-1.medium.com/fit/c/800/574/1*WbxWhGlMC6vneCltMaFBpQ.png)✍︎Clean and simple dashboard

Keeping an eye on everything
----------------------------

Installing all these apps is great, but what happens if a service suddenly goes down or an update becomes available? I needed a way to stay informed without constantly checking each app manually.

### Notifications, notifications everywhere

I already knew about **[ntf.sh](https://ntfy.sh/)**, a simple HTTP-based [pub-sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) notification service. Until this point, I had been using the free cloud version, but I decided to self-host it so I could use private notification channels and keep everything under my own control.

![](https://cdn-images-1.medium.com/fit/c/800/411/1*Q2O5lKjpQLOrU-LEzIYSCQ.png)✍︎Notification channels in ntfy.sh

I have 3 channels configured:

*   One for my backups (yeah I have backups configured)
*   One for available app updates
*   One for an open-source project I’m maintaining where I need to keep an eye on.

### What’s Up Docker?

**[WUD](https://github.com/getwud/wud)** **(What’s Up Docker?)** is a service to keep your containers up to date. It monitors your images and sends notifications whenever a new version is released. It also integrates nicely with ntfy.sh.

![](https://cdn-images-1.medium.com/fit/c/800/436/1*NfDEPuaPRL04uf6fg5H4Zg.png)✍︎[https://getwud.github.io/wud/assets/wud-arch.png](https://getwud.github.io/wud/assets/wud-arch.png)

### Uptime monitor

To monitor all my services, I installed **[Uptime Kuma](https://github.com/louislam/uptime-kuma)**. It’s a self-hosted monitoring tool that alerts you whenever a service or app goes down, ensuring you’re notified the moment something needs attention.

Backups, because disaster will strike
-------------------------------------

I’ve had my fair share of _whoopsies_ in the past, accidentally deleting things or breaking setups without having proper backups in place. I wasn’t planning on making that mistake again. After some research, it quickly became clear that a **3–2–1 backup strategy** would be the best approach.

> The 3–2–1 backup rule is a simple, effective strategy for keeping your data safe. It advises that you keep three copies of your data on two different media with one copy off-site.

I accidentally stumbled upon **[Zerobyte](https://github.com/nicotsx/zerobyte)**, which is IMO the best tool out there for managing backups. It’s built on top of **Restic**, a powerful CLI-based backup tool.

I configured three repositories following the 3–2–1 backup strategy: one pointing to my server, one to a separate hard drive, and one to [Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/). After that, I set up a backup schedule and from here on out, Zerobyte takes care of the rest.

![](https://cdn-images-1.medium.com/fit/c/800/385/1*FTixOu72EzJNl_yJkT2L0Q.png)✍︎My backup strategy

Exposing my apps to the world wide web
--------------------------------------

Some of the services I’m self-hosting are meant to be publicly accessible, for example, [my resume](https://resume.robiningelbrecht.be/robin/php-developer). Before putting anything online, I looked into how to do this securely. The last thing I want is random people gaining access to my server or local network because I skipped an important security step.

To securely expose these services, I decided to use **[Cloudflare tunnels](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)** in combination with **Tailscale**. In the Cloudflare dashboard, I navigated to **Zero Trust > Network > Tunnels** and created a new Cloudflared tunnel.

Next, I installed the **Cloudflared Docker image** on my server to establish the tunnel.

```
services:
  tunnel:
    image: cloudflare/cloudflared
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL\_TOKEN=\[CLOUDFLARE-TOKEN\]
networks: {}
```![](https://cdn-images-1.medium.com/fit/c/800/306/1*T2ISwRLUarteTlp6OUbtJA.png)✍︎Cloudflare picking up the tunnel I set up

Finally, I added a **public hostname** pointing to my Tailscale IP address, allowing the service to be accessible from the internet without directly exposing my server.

![](https://cdn-images-1.medium.com/fit/c/800/261/1*_sSZ-Yp45_RFC8Xzy-T72w.png)✍︎Public hostname record

Final Thoughts
--------------

Self-hosting started as a curiosity, but it quickly became one of the most satisfying projects I’ve ever done. It’s part tinkering, part control, part obsession and there’s something deeply comforting about knowing that all my services live on a box I can physically touch.

**Disclaimer:** This is simply the setup that works well for me. There are many valid ways to build a homeserver, and your needs or preferences may lead you to make different choices.
