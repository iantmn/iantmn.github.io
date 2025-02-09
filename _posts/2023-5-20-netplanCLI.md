---
title: "🛜 Netplan CLI tool"
description: "I created a CLI tool for Netplan.io to make it easier to configure the network on Linux."
date: 2023-05-20
permalink: /posts/2023/05/NetplanCLI/
categories: [Projects, 🖥️ Software Development]
tags: [Network 🛜, Linux 🐧, netplan 🛜]
pin: false
---

## Simplifying network configuration on Linux with a custom Netplan CLI tool

Netplan is a widely used tool for configuring networking on Linux, particularly on distributions like Ubuntu. It allows for YAML-based network configuration, which is then translated into backend-specific configurations during boot, controlling network interfaces and services. This is explained in more detail in the [Netplan.io documentation](https://netplan.readthedocs.io/en/stable/).

Despite Netplan’s functionality, my client found it challenging to work with the YAML syntax for configuring their network. They required a more user-friendly solution, so I developed a custom CLI tool in Python to simplify the process.

## Key Features

🔧 Easier Network Configuration – The tool provides an intuitive command-line interface to generate Netplan’s YAML files, making it easier to configure network settings without dealing directly with YAML syntax.

⚙️ Automation – It can also automate product installations, streamlining the setup of network configurations alongside other deployment tasks.

💡 Flexibility – The tool supports most network configurations but is not intended to fully replace the original Netplan.io CLI due to its complexity and the client’s specific needs.

## Challenges & Learnings

⚙️ Custom Tool vs. Extending Netplan CLI – Instead of modifying the existing Netplan CLI, I opted to create a new tool because of the specific requirements and the complexity of Netplan’s original interface.

🔄 Balancing Flexibility & Simplicity – Ensuring the tool was simple to use while maintaining flexibility for complex network setups was a key challenge. I had to make design decisions that would allow the tool to handle the most common scenarios while leaving room for customization where needed.

🌐 Learning Networking – Throughout this project, I had to deepen my understanding of networking concepts, such as IP addressing, routing, network interfaces, subnetworks and VLANs. This was a steep learning curve, as networking can be a complex field. Understanding how network configurations are structured and how changes affect the network was essential for developing a reliable tool. The process of learning these concepts also helped me optimize the tool and better anticipate issues that users might encounter.

💡 Want to learn more? Feel free to contact me!