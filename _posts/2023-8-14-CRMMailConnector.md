---
title: "📩 Email + Vtiger CRM ticker connector"
date: 2023-08-14
categories: [Projects, 🖥️ Software Development]
tags: [Email 📩, Vtiger CRM 📊]
pin: false
---

When using a CRM like Vtiger there is a possibility to make a ticket for a client. This ticket has be to updated manually with comments and information. This is a time consuming task. I created a connector between the email server and the CRM system to automate this process. The connector is written in Python and uses the IMAP protocol to communicate with the email server and the Vtiger API to communicate with the CRM system. When an email comes in, the connector searches for a ticket ID in the subject. If it finds one, it automatically adds the email as a comment to the ticket. If it doesn't find one, it can be ignored, because it's not a ticket.