---
title: "🛜 Proxmox let's encrypt using ACME transip DNS"
description: "Proxmox uses a self-signed certificate by default. This guide will show you how to use Let's Encrypt with ACME DNS-01 challenge to get a valid certificate, using Transip, including the two gotchas that make it fail silently."
date: 2025-3-10
permalink: /posts/2025/3/ProxmoxTransip/
categories: [Guides, 🛜 Networking]
tags: [Proxmox 🖥️, DNS 🌐, ACME 🛜]
pin: false
published: true
---

# Introduction

Proxmox ships with a self-signed certificate, so every visit to the web interface greets you with a browser warning. You learn to click through it, which is exactly the habit you do not want: once you ignore certificate warnings by reflex, a real one stops meaning anything.

A Let's Encrypt certificate fixes that, and Proxmox has ACME support built in. The catch is the challenge type. The usual HTTP-01 challenge requires Let's Encrypt to reach your node on port 80 from the internet, which is the last thing you want for a hypervisor. The DNS-01 challenge avoids it entirely: instead of proving you control a web server, you prove you control the domain, by letting Proxmox create a temporary `_acme-challenge` TXT record. Your node never needs to be publicly reachable at all, and you can issue certificates for hosts that only exist on your LAN.

Proxmox does this through DNS plugins borrowed from the [acme.sh](https://github.com/acmesh-official/acme.sh) project, and TransIP is among them. There are plenty of walkthroughs for the big providers and almost nothing for TransIP, so this guide covers it, including two things that will otherwise cost you an evening: how the API key is allowed to be used, and the permissions on the key file.

## What you need

- One or more Proxmox VE nodes, with root shell access
- A domain in a TransIP account
- A DNS name per node, for example `pve1.example.com`. It does not need to resolve publicly
- About ten minutes

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

If you use a global key in step 2, add one more line to the API data:
```
TRANSIP_Token_Global_Key=true
```

## 2. Transip API key
1. Go to [`https://www.transip.nl/cp/account/api/`](https://www.transip.nl/cp/account/api/) and create an API key (Key Pairs)
2. Decide how that key may be used, because this is set when you create it and cannot be changed afterwards:
	- **Static external IP address**: leave *Only allow whitelisted IP's* checked, and whitelist the external IP address of the Proxmox host(s)
	- **Anything else**: uncheck *Only allow whitelisted IP's*, and add `TRANSIP_Token_Global_Key=true` to the API data from step 1.2
	- Not sure which? See [Choosing a key type](#choosing-a-key-type) below
3. Go to the proxmox shell of a node and add the content of the private key inside `/etc/transip-private-key.pem`
4. Set the permissions on that file, or the plugin cannot read it:
```bash
sudo chown root:nogroup /etc/transip-private-key.pem
sudo chmod 640 /etc/transip-private-key.pem
```
5. Repeat step 3 and 4 for every node/host

Do not put the key in `/etc/pve`. See [Why the key file needs nogroup](#why-the-key-file-needs-nogroup) for the reason.

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

Proxmox renews automatically from here, roughly thirty days before expiry.

## Choosing a key type

**Use a whitelisted key if you can.** It is the default, it is the stronger of the two, and it costs you nothing extra. Only reach for a global key when your node's outgoing IP address is something you genuinely cannot pin down.

Whichever you pick, the setting in the TransIP control panel and the line in the API data have to match:

| "Only allow whitelisted IP's" | `TRANSIP_Token_Global_Key` | Result |
|---|---|---|
| Checked | leave it out | Works, but only from the whitelisted IP addresses |
| Unchecked | `true` | Works from any IP address |
| Checked | `true` | Does not work |

That third row is the easy mistake to make, because it looks like the fix. Adding the line to a key that was created as restricted does not promote it: TransIP will not hand out a global token for a key that is locked to a whitelist, which is rather the point of the whitelist.

### What a global key actually is

The name misleads, because it is not a different kind of key at all. It describes the token.

TransIP authentication runs in three stages. First there is the **key pair** you create in the control panel, whose private half ends up in `/etc/transip-private-key.pem`. That key is never used directly against the API. Instead the plugin uses it to **sign a request for a temporary access token**, which TransIP hands over if the signature checks out. acme.sh asks for a token that lives thirty minutes. Only then does the plugin make the **actual API calls** that create and later remove the `_acme-challenge` TXT record.

`global_key` is a field in that middle stage, the token request:

```json
{"login":"...","nonce":"...","read_only":"false","expiration_time":"30 minutes","label":"","global_key":"false"}
```

So it is a property of the token you ask for, not of the key you hold. The checkbox in the control panel decides whether the key is allowed to request such a token, and the line in the API data decides whether it actually does. That is the whole reason both halves are needed, and why one idea ends up with three names: TransIP calls it *Only allow whitelisted IP's*, the API field is `global_key`, and acme.sh calls the variable `TRANSIP_Token_Global_Key`.

It is worth being precise about what a global key costs you, because it is less than people assume and more than nothing. A global token is **not** less authenticated: the request still has to be signed with your private key, or TransIP issues nothing at all. What you give up is one layer:

- With a whitelist, an attacker needs your private key **and** has to reach TransIP from an address you approved
- With a global key, your private key is the only thing in the way

Which is also why the key file is the thing to protect, rather than the token. A token expires in thirty minutes. A private key does not expire at all.

### When to go global anyway

Pick the global key if any of these apply:

- Your WAN IP address is dynamic, which covers most home and small office connections
- Your node prefers IPv6 for outgoing connections, so TransIP sees a v6 address rather than the v4 one you whitelisted
- You run a cluster whose nodes leave the network through different addresses
- You are behind CGNAT, a VPN, or policy routing, so the address TransIP sees is not the one you expect

The reason this matters more than it first appears is the way whitelisting fails when it was the wrong choice. Ordering the certificate works, because you whitelisted the correct address that afternoon. Then your WAN address changes and the **renewal** fails, roughly sixty days later. Proxmox renews on a timer, so nothing tells you at the time. You find out when a certificate expires, or when you happen to open a task log.

So the advice is not to whitelist at any cost. It is to whitelist when your address holds still, and to switch deliberately when it does not, rather than finding out two months later.

If you do go global, create a **separate key pair** just for ACME rather than reusing one you already use elsewhere, so you can revoke it on its own. That key can add and delete DNS records for every domain in the account, and with a global key it is the only thing protecting them.

## Why the key file needs nogroup

This is the one most likely to cost you an evening, because the error message points at the wrong thing.

Proxmox does not run the DNS plugin as root. It drops privileges first, which you only ever see when something fails:

```
TASK ERROR: command 'setpriv --reuid nobody --regid nogroup --clear-groups --reset-env -- /bin/bash /usr/share/proxmox-acme/proxmox-acme setup transip example.com' failed: exit code 1
```

`setpriv --reuid nobody --regid nogroup` is the giveaway. The plugin runs as the unprivileged `nobody` user, whose group on Debian is `nogroup`. A key file that only root can read simply does not exist as far as the plugin is concerned.

What makes this hard to diagnose is what happens next. The plugin checks your key by looking for the line `BEGIN PRIVATE KEY` inside it. When the file cannot be opened at all, that check finds nothing, and the error blames the contents:

```
Key file doesn't seem to be a valid key: /etc/transip-private-key.pem
Error add txt for domain:_acme-challenge.example.com
```

Nothing is wrong with the key. It was never read. The fix is the `chown` and `chmod` in step 2.4.

`640` with group `nogroup` is the smallest permission that works: root can write it, the plugin can read it, and ordinary user accounts cannot. There is no reason to use `644`, which gains nothing here and makes the key readable by everyone.

The same logic explains why the key does not belong in `/etc/pve`. That is the Proxmox cluster filesystem, with its own ownership rules, and `nobody` cannot read it either. You get the identical misleading error.

One trade-off worth naming: `nobody` is a deliberately shared identity, so any other service on the node running as `nobody` can read this file too. This is the smallest permission that works rather than a genuinely tight one, which is one more argument for giving ACME its own TransIP key pair.

## Troubleshooting

If the task fails with `Key file doesn't seem to be a valid key`, it is almost certainly permissions rather than the key itself. Check `ls -l /etc/transip-private-key.pem` and confirm it is `root:nogroup` with mode `640`.

If issuing worked months ago and a renewal has now started failing, suspect the IP whitelist on the API key. Your external address has probably changed. This is the failure mode a global key is for.

If the task fails immediately with an authentication error, check that the username in the API data is your TransIP account username, not an email address, and that the key pair you pasted is the one the whitelist settings belong to.

If the challenge is created but validation fails, give DNS longer. Proxmox waits for TXT propagation before asking Let's Encrypt to check, but TransIP records carry a 300 second TTL, so a resolver that cached a negative answer can still be behind.

If it works on one node and not another, you have almost certainly not repeated steps 3 and 4 there. Each node needs its own copy of the key file with its own permissions, and each node's external address needs whitelisting if you went that route.

## Credits

Thanks to the reader who mailed me about the group `nogroup`, which is why this guide now covers the command to set the correct permissions.
