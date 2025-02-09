---
title: "📞 SIP + Vtiger CRM Connector"
description: "I created a connector between the SIP server and the Vtiger CRM system to automate the process of creating tickets."
date: 2023-12-01
permalink: /posts/2023/12/SIPVtigerCRM/
categories: [Projects, 🖥️ Software Development]
tags: [SIP 📞, Vtiger CRM 📊]
pin: false
---

My client struggled with their technical support calls because it took a lot of time to get the information from the customer and to create a ticket in their CRM system. I created a connector between their SIP server and their CRM system to automate this process. The connector is written in Python and uses the SIP protocol to communicate with the SIP server and the Vtiger API to communicate with the CRM system. When a call comes in, the connector automatically generates a pop-up in the CRM system with the customer's information. From this page a ticket can be created with one click.

The challenge was to get the SIP server to send the information to the connector. The SIP protocol is a complex protocol and it took a lot of time to understand how it works. I use regex to extract the information from the SIP message. To receive the SIP messages I use a library called pyVoIP. This library creates a softphone that can receive calls. I extract the phone number from the SIP message and use the databse of Vtiger to get the customer information. Then I use windows toasts to create a pop-up with the customer information. This pop-up can be used to go to the customer page in Vtiger where a ticket can be created.