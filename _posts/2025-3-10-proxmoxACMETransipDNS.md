---
title: "🛜 Proxmox let's encrypt using ACME transip DNS"
description: "Proxmox uses a self-signed certificate by default. This guide will show you how to use Let's Encrypt with ACME DNS-01 challenge to get a valid certificate, using Transip."
date: 2025-3-10
permalink: /posts/2025/3/ProxmoxTransip/
categories: [Guides, 🛜 Networking]
tags: [Proxmox 🖥️, DNS 🌐, ACME 🛜]
pin: false
published: true
---

# Introduction
Proxmox uses a self-signed certificate by default. This short guide will show you how to use Let's Encrypt with ACME DNS-01 challenge to get a valid certificate. There are a lot of examples for more common DNS providers, but not for Transip.

## 1. In proxmox datacenter->ACME
1. accounts -> Add a new account and use ACME directory **without staging**
```
Account name: <Name it whatever you want>
Email: <The contact email for these domains>
ACME Directory: let's encrypt V2 (NOT STAGING)
Accept TOS: Checked
```
2. Challenge plugin -> Set the following settings
```
Plugin ID:	<Name it whatever you want>
API data:	TRANSIP_Username=<USERNAME>
				   TRANSIP_Key_File=/etc/transip-private-key.pem
```
## 2. Transip API key
1. Go to [`https://www.transip.nl/cp/account/api/`](https://www.transip.nl/cp/account/api/) and create an API key (Key Pairs)
2. Whitelist the external ip-adres of the Proxmox host(s)
3. Go to the proxmox shell of a node and add the content of the private key inside `/etc/transip-private-key.pem`.
4. Repeat this for every node/host

## 3. In Proxmox Node->system->Certificates->ACME
1. Click `Add`
2. Use the following settings and click `create`
```
Challange type: DNS
Plugin: <Name of challenge plugin (step 1.2)>
Domain: <domain you use for (local) DNS resolving (e.g. pve1.example.com)
```
3. Select the account made in step 1.1 by `Using Account:`
4. Click `Order Certificates Now`
5. Wait for the promt to say `TASK OK` and reload the page.
6. It should now have a valid certificate. Repeat this for all nodes.