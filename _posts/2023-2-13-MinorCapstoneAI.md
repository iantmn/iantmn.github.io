---
title: "🤖 Capstone project of the Minor Engineering with Artificial Intelligence: Using AI to classify how a product is used"
description: "For my minor Engineering with Artificial Intelligence I did a capstone project. The goal of this product was to create a tool that enables Industrial Designers to analyse how a product is used in the real world using sensors."
date: 2023-02-13
permalink: /posts/2023/02/MinorCapstoneAI/
categories: [Projects, 🖥️ Software Development]
tags: [Education 🎓, AI 🤖]
pin: true
---

For the full report [click here!](/assets/capstoneAI-report.pdf)

For my minor Engineering with Artificial Intelligence I did a capstone project. The goal of this project was to create a tool that enables Industrial Designers to analyse how a product is used in the real world using sensors. Normally this is done by using a video camera and manually analysing the video. This is a time consuming process and it is difficult to get accurate results. The tool we created uses a machine learning model to analyse the sensor data and automatically detect the usage of the product. This way the Industrial Designer can get accurate results in a short amount of time.

We started off with a literature study to find out what already exists in this field and what models are often used for analysing sensor data. We found out that there are a couple of models that are often used for analysing sensor data. After that we started the make a preprocessing pipeline to prepare the data for the machine learning model. We used the data from the sensors of a gopro camera. We used the accelerometer, gyroscope and magnetometer data. We also used the video data from the gopro camera. We used the video data to label the data.

The preprocessing pipeline consists out of multiple steps. First we synchronised the data from the sensors and the video. Then we used a sliding window to split the data into smaller parts. After that we extracted some features like the central power, peak value frequency, and more. We also used the [Fast Fourier Transform](https://en.wikipedia.org/wiki/Fast_Fourier_transform) to extract the frequency domain features.

After that we started to create a machine learning model. We used a random forrest classifier because of its good performance, its ability to handle a lot of data and the fact that it is relativly fast to train. We want our user to be able to use the tool at real time and this was therefore very important.

After that we choose a Jupyter notebook as our interface because it was easy to setup and it was easy to use. Due to limited time we couldn't make a web interface. However this is something that can be done in the future. All functions that are used are made into modules so that they can be used in a web interface.

During the project we noticed that we could use a form of [active learning](https://en.wikipedia.org/wiki/Active_learning_(machine_learning)) to make sure a user gets the best results with the least amount of data. Every iteration the user is prompted to label the data that the model is most uncertain about.

At the end we also made a basic type of novelty detection. This is a form of [anomaly detection](https://en.wikipedia.org/wiki/Anomaly_detection). This is useful for detecting when a product is used in a way it is not designed for. For example, when running shoes are used for hiking. This is useful for Industrial Designers because they can use this information to improve their product.

We also made a user manual to explain how to use the tool. This was in the form of a guided tour in the Jupyter notebook and a [website](https://datacentricdesign.github.io/iot-ml-design-kit/).

For more information about the project, see the [GitHub repository](https://github.com/iantmn/Capstone-AI-IoT) or [the website we made](https://datacentricdesign.github.io/iot-ml-design-kit/) for the project.