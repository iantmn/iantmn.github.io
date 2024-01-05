---
title: "🛜 Netplan CLI tool"
date: 2023-05-20
categories: [Projects, 🖥️ Software Development]
tags: [Network 🖧, Linux 🐧, netplan 🛜]
pin: false
---

Netplan is a tool for configuring networking on Linux. It is used by Ubuntu and other Linux distributions. It is a YAML network configuration abstraction for various backends. Netplan reads network configuration from /etc/netplan/*.yaml. During early boot it then generates backend specific configuration files in /run to hand off control of devices to a particular networking daemon. How this works is explained in the [Netplan.io documentation](https://netplan.io/). 

However my client found it difficult to use the YAML syntax and wanted a CLI tool to configure the network. I created this tool in Python. It uses the Netplan.io CLI to generate the YAML file with a easier to use interface. The tool is also used to automate the installation of their products. It works on most network configurations, but it is not a complete replacement for the Netplan.io CLI. Because of the complexity of the Netplan.io CLI and the requirements of my client, I decided to create a new tool instead of extending the Netplan.io CLI.