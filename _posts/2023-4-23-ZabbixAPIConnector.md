---
title: "📊 Zabbix API connector"
description: "I created a connector between the Zabbix API and the API of my client's product. This way the client can monitor their product in Zabbix."
date: 2023-04-23
permalink: /posts/2023/04/ZabbixAPIConnector/
categories: [Projects, 🖥️ Software Development]
tags: [Zabbix 📊, Linux 🐧, API ⚙️]
pin: false
---

Zabbix is a monitoring tool that can monitor things like servers, networks, network attached storages, and much more. It is powerful and has futures like alerting, dashboards, auto discovery of hosts and their items and reporting. It is used by many companies to monitor their infrastructure. For more information about Zabbix, see the [Zabbix website](https://www.zabbix.com/).

My client wanted to use Zabbix to monitor the API they added to their product. This API gave statistics about the product and the client wanted to monitor these statistics in a central place. I setup a Zabbix server and made some templates to monitor the API. I also wrote a Python script that Zabbix uses to automatically discover the API statistics because the API is still in development and the statistics can change. Previously the client had to manually change the template in Zabbix, but now this is done automatically.

The Zabbix server also provides alerting when one of their machines goes down. This is done by using the Zabbix agent on the machines. The Zabbix agent sends the status of the machine to the Zabbix server. When the Zabbix server does not receive a status update for a certain amount of time, it sends an alert to the client. This way the client can respond quickly when one of their machines goes down.

The Zabbix server also provides a dashboard with the most important statistics. This way the client can see the status of their machines at a glance, or they can troubleshoot problems.

Setting up all of this was a challenge because it involved various aspects. For example, networking, proxies, encryption, Linux, Zabbix, Python and the API. But it was a fun challenge, and I learned a lot from it.