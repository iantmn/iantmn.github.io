---
title: "📩 Email + Vtiger CRM ticker connector"
description: "I created a connector between the email server and the Vtiger CRM system to automate the process of updating tickets."
date: 2023-08-14
permalink: /posts/2023/08/EmailVtigerCRM/
categories: [Projects, 🖥️ Software Development]
tags: [Email 📩, Vtiger CRM 📊]
pin: false
---

When using a CRM like Vtiger there is a possibility to make a ticket for a client. These tickets had previously been updated manually with comments and information. This is a time-consuming task. I created a connector between the email server and the CRM system to automate this process. The connector is written in Python and uses the IMAP protocol to communicate with the email server and the Vtiger API to communicate with the CRM system. When an email comes in, the connector searches for a ticket ID in the subject. If it finds one, it automatically adds the email as a comment to the ticket. If it does not find one, it can be ignored, because it is not ticket related.

After receiving the emails and extracting the ticker ID with regex. The hardest part began. The challenge was actually to use the API from Vtiger to work. There was almost no documentation so I had to try and reverse engineer the PHP functions responsible for the API. After a lot of trial and error I managed to get the API to work.