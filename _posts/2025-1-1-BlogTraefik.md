---
title: "🛜 Using Traefik V3 as a Reverse Proxy for a docker compose server"
description: "I implemented Traefik as a reverse proxy to manage multiple services in a docker compose server. This post explains how to set up Traefik and configure it."
date: 2025-2-5
permalink: /posts/2025/2/traefik/
categories: [Blog posts, 🛜 Networking]
tags: [Traefik 🛜]
pin: false
published: true
---
# What is traefik?
Traefik is a modern HTTP reverse proxy and load balancer that makes deploying microservices easy. Traefik integrates with your existing infrastructure components (Docker, Swarm mode, Kubernetes, etc.) and configures itself automatically and dynamically.

I use it for my docker compose server, a server on which I host a variety of services. Traefik allows me to manage these services through a single entry point and provides me with a lot of flexibility in terms of routing and load balancing.

It has some benefits over other reverse proxies.For example, it can be managed by docker compose labels, which makes it very easy to use in a docker compose server. It also has a web interface that allows you to see the status of your services and the routing rules that are in place.

# Prerequisites
Before you start, you need to have docker and docker-compose installed on your server. You also need to have a domain name that you can point to your server.

A basic knowledge about networking and reverse proxies is also helpful, however I will try to explain everything as clearly as possible.

# Setting up Traefik
The first step is to create a docker-compose file for Traefik. Here is an example of a basic Traefik docker-compose file:

```yaml
services:
  traefik:
    image: traefik:v3.3.3
    restart: always
    command:
      - --configFile=/etc/traefik/traefik.yml # Path to the config file
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # Docker socket
      - ./acme.json:/acme.json # File to store SSL certificates
      - ./traefik.yml/:/etc/traefik/traefik.yml # Config file
    container_name: traefik
    networks:
      - traefik # Network for traefik
    labels:
      - "traefik.enable=true" # Enable traefik for this container
      - "traefik.http.routers.api.rule=Host(`traefik.example.com`)" # Set up a rule for the web interface, make sure to change the domain name and set up the DNS records!
      - "traefik.http.routers.api.service=api@internal" # Set up a service for the web interface
      - "traefik.http.routers.api.entrypoints=websecure" # Set up an entry point for the web interface
      - "traefik.http.routers.api.tls=true" # Enable TLS for the web interface
      - "traefik.http.routers.api.tls.certresolver=myresolver" # Use the certificate resolver
networks:
  traefik:
    external: true # Use an external network
```
Make sure to edit the URL in line 19 and set the DNS record to this server!

After that we need a basic traefik.yml config file. Located on your server at `./conf/traefik.yml`
```yaml
entryPoints:
  web:
    address: ":80" # HTTP port
  websecure:
    address: ":443" # HTTPS port

certificatesResolvers: # Certificate resolver
  myresolver:
    acme:
      email: <EMAIL>
      storage: acme.json
      httpChallenge:
        entryPoint: web

api: # Web interface enable
  dashboard: true
  insecure: true

providers: # Providers
  docker: # Docker provider for dynamic configuration inside docker compose files
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file: # File provider for static
    filename: /etc/traefik/traefik.yml
    watch: true
```
Make sure to change the email inside the yaml file!

When you have set up the docker-compose file and the config file, you need to create an acme.json file. This file is used to store the SSL certificates that Traefik will generate. You can create an empty file with the following command:

```bash
touch acme.json
chmod 600 acme.json
```

Now we just need to create the network for traefik. You can do this with the following command:

```bash
docker network create traefik
```

Now you can start Traefik with the following command:

```bash
docker-compose up -d
```

If you changed your DNS records to point to your server, you should now be able to access the Traefik web interface by going to `https://traefik.example.com`. You should see a dashboard with information about the services that are running on your server.

# Creating a new service
Now that Traefik is set up, you can start adding services to your docker compose server. Here is an example of a docker-compose file for a simple service:

```yaml
services:
  whoami:
    image: containous/whoami
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.example.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
      - "traefik.http.routers.whoami.service=whoami"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"
networks:
  traefik:
    external: true
```
Make sure to edit the URL in line 8 and set the DNS record to this server!

If you start this docker-compose file, Traefik will automatically detect the service and set up a routing rule for it. You should now be able to access the service by going to `https://whoami.example.com`. It handles the SSL certificate generation and renewal for you.

# Conclusion
Traefik is a powerful tool that makes managing multiple services on a docker compose server easy. It has a lot of features that make it very flexible and easy to use. I hope this post has helped you to set up Traefik and configure it for your own server. If you have any questions or feedback, feel free get in contact!

For more information, see the [Traefik documentation](https://doc.traefik.io/traefik/).