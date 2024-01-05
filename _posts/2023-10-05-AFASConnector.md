---
title: "AFAS file connector for key management system"
date: 2023-10-05
categories: [Projects, 🖥️ Software Development]
tags: [AFAS 📊]
pin: false
---

I made a connector for AFAS Software, a Dutch ERP system, to connect it to a key management system. The connector is written in Python and uses the AFAS Get Connector API to communicate with their instance of AFAS. The connector is used to synchronize data between the key management system and AFAS. This allows the client to use the AFAS software packages (the platform where they manage their employees, among other things) to manage the access rights of their employees. The software is used by a large Dutch company with over 1000 employees.