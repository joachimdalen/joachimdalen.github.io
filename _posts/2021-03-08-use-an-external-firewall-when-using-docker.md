---
layout: single
title: "Why you should use an external firewall when using Docker"
date: "2021-03-08"
categories:
  - Security
tags:
  - Infrastructure
  - Tools
  - Docker
  - Networking
excerpt: "Do you use Docker on cloud hosted machines and expose container ports to the host? Then this might be worth a read"
header:
  overlay_image: assets/images/posts/05-docker-firewall/header.jpg
  overlay_filter: rgba(0, 0, 0, 0.5)
  teaser: assets/images/posts/05-docker-firewall/header.jpg
  image: assets/images/posts/05-docker-firewall/header.jpg
  caption: "Photo credit: [**Thomas Jensen**](https://unsplash.com/@thomasjsn?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [**Unsplash**](https://unsplash.com/s/photos/computer-network?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)"
toc: true
---

I guess most of us have experimented with Docker at some point, maybe just on our local machine or on a cloud-hosted virtual machine. There are some things that you should keep in the back of your head when using it in the cloud.

# Did you know?

One thing I think a lot of people don't realize by using the `-p` argument (when using CLI) or the `ports` (when using Docker Compose) is that this will bypass your host firewall.

Let me illustrate with an example.

On my host I have Nginx installed and the host firewall activated. `sudo ufw status verbose` returns the following:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                           Action      From
--                           ------      ----
80,443/tcp (Nginx Full)      ALLOW IN    Anywhere
80,443/tcp (Nginx Full (v6)) ALLOW IN    Anywhere (v6)
```

As you can see, the host firewall is active and blocking all traffic except on ports related to Nginx.

Now I deploy and start the following container

```bash
docker run -it --rm -d -p 8080:80 --name web nginx
```

Let us take a look at what `docker ps` returns:

```
CONTAINER ID   IMAGE  ... PORTS                  NAMES
315fb926d486   nginx  ... 0.0.0.0:8080->80/tcp   web

```

We see our Nginx container listed here, as well as the port `0.0.0.0:8080`. But what does this mean? Using the bind address `0.0.0.0` means that the container will listen on port `8080` on all network interfaces.

Taking a look at the host firewall again, we see that there have been no changes.

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                           Action      From
--                           ------      ----
80,443/tcp (Nginx Full)      ALLOW IN    Anywhere
80,443/tcp (Nginx Full (v6)) ALLOW IN    Anywhere (v6)
```

`ufw` provides an abstraction layer but uses `iptables` in the background. Docker modifies `iptables` directly when adding port and network configuration. If we take a look in the [Docker documentation](https://docs.docker.com/network/iptables/) we find the following statements _(Please read the documentation for full context)_:

> This means that if you expose a port through Docker, this port gets exposed no matter what rules your firewall has configured.

> By default, all external source IPs are allowed to connect to the Docker host. To allow only a specific IP or network to access the containers, insert a negated rule at the top of the `DOCKER-USER` filter chain.

> It is not possible to completely prevent Docker from creating `iptables` rules....

So even if we haven't specified any rules to allow our container to access the internet the underlying rules created by Docker will automatically publish access to your container. After deploying the container above, try going to `<your-public-virtal-machine-ip>:8080` and you should see the following:

![Nginx Welcome](/assets/images/posts/05-docker-firewall/nginx-welcome.png)

# Fixing the issue

It is easy to not think about this when deploying your containers, but it can pose a huge security risk to your data, host, and application.

There are several ways we can fix this. The simple fix is to set `iptables` to `false` in `/etc/docker/daemon.json`. This should make Docker honor the `ufw` rules and not expose container ports.

I want to look back to the third excerpt from the documentation I posted earlier:

> It is not possible to completely prevent Docker from creating `iptables` rules....

As I do fully trust the fix over, I advise everyone that uses Docker to host their application to use an external firewall to maintain better control on what traffic is allowed from and to a host.

# Firewalls

I will not go into details about how to deploy or configure your firewall, because that depends from provider to provider. Deploying this external firewall will have both pros and cons. Just to mention some:

- Traffic on disallowed ports is rejected before it even reaches your host (depending on the provider)
- You maintain better control over what traffic is allowed in and out

There are also some cons to it:

- One more step to perform when needing to deploy new services or change ports for existing applications
- Can increase cost and latency

## Some other tips for securing your application

- Never use the default port for anything
- Perform regular security audits of your infrastructure
- Always secure your applications and side services with authentication, even if they are just running on the local network.

Thank you for the read!
