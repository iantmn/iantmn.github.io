---
title: "📊 AFAS file connector for key management system"
description: "I made a connector for AFAS Software, a Dutch ERP system, to connect it to a key management system."
date: 2023-10-05
permalink: /posts/2023/10/AFASConnector/
categories: [Projects, 🖥️ Software Development]
tags: [AFAS 📊]
pin: false
---

I made a connector for AFAS Software, a Dutch ERP system, to connect it to a key management system. The connector is written in Python and uses the AFAS Get Connector API to communicate with their instance of AFAS. The connector is used to synchronize data between the key management system and AFAS. This allows the client to use the AFAS software packages (the platform where they manage their employees, among other things) to manage the access rights of their employees. The software is used by a large Dutch company with over 1000 employees.

The AFAS get connector API is a REST API that can be used to retrieve data from AFAS. The API is documented on the [AFAS website](https://help.afas.nl/help/EN/SE/App_Cnr_Rest_GET.htm). The API uses a token to connect. After generating the token, the right data needs to be collected. The data was given in form of a json and needed to be converted to a csv file. The connector uses the requests library to communicate with the API.