---
title: "🤖 AI Minor Project: Leveraging AI to Analyze and Classify Product Usage in the Engineering with Artificial Intelligence Minor"
description: "For my Minor in Engineering with Artificial Intelligence, I worked on a project aimed at creating a tool to help Industrial Designers analyze how a product is used in the real world using sensor data."
date: 2023-02-13
permalink: /posts/2023/02/MinorCapstoneAI/
categories: [Projects, 🖥️ Software Development]
tags: [Education 🎓, AI 🤖]
pin: true
---

## Project Overview:

In traditional product usage analysis, video cameras are commonly used to record and analyze how a product is used. However, this process is time-consuming and often yields less accurate results. The goal of our project was to develop a tool that utilizes machine learning to automatically analyze sensor data and detect the product’s usage. This solution allows Industrial Designers to gain accurate insights in a fraction of the time.

## Project Process:

### 1. 📖 Literature Study & Model Selection:

We began by conducting a literature study to understand the existing solutions in this field and the machine learning models frequently used to analyze sensor data. This helped us identify key models that could be effective for our task.
### 2. ⚙️ Preprocessing the Data:

To prepare the sensor data for machine learning, we created a comprehensive preprocessing pipeline. The data was sourced from a GoPro camera’s sensors: accelerometer, gyroscope, and magnetometer. We also used the video data from the camera to label the sensor data.

Preprocessing Steps:

- Data Synchronization: We synchronized the sensor and video data to ensure they aligned correctly.
- Sliding Window: We used a [**sliding window technique**](https://www.geeksforgeeks.org/window-sliding-technique/) to split the data into smaller, manageable chunks.
- Feature Extraction: Features such as central power, peak value frequency, and others were extracted. Additionally, we applied the [**Fast Fourier Transform (FFT)**](https://en.wikipedia.org/wiki/Fast_Fourier_transform) to extract frequency domain features.

### 3. 🤖 Machine Learning Model:

After an analysis for different ML models, we chose a [**Random Forest classifier**](https://www.ibm.com/think/topics/random-forest) for the machine learning model. Its strong performance, ability to handle large datasets, and relatively fast training time made it ideal for fast analysis.

### 4. 🖥️ Interface Development:

We opted for a Jupyter Notebook interface due to its ease of use and quick setup. While we lacked the time to build a web interface, all functions were modularized, making future web integration feasible.

### 5. 🏃‍♂️ Active Learning:

We discovered that [**active learning**](https://en.wikipedia.org/wiki/Active_learning_(machine_learning)) could enhance the model’s performance by selecting the most uncertain data points for the user to label. This ensured that the model received the most relevant data, minimizing the amount of labeling required. We created a custom algorithm to analyse which datapoint we should ask.

### 6. 🔍 Novelty Detection:

We incorporated novelty detection (a form of [**anomaly detection**](https://www.ibm.com/think/topics/machine-learning-for-anomaly-detection)) to identify when a product is being used in an unintended way. For example, detecting if running shoes are being used for hiking. This feature helps Industrial Designers identify potential product flaws, areas for improvement or opportunities.

## Results: 

### 1. ⏱️ Fast Analysis:

The machine learning model processes sensor data much faster than manual analysis, allowing Industrial Designers to receive insights about how a product is being used. This eliminates the need for time-consuming manual analysis, enabling designers to quickly iterate on product designs or make adjustments based on how the product is performing in real-world conditions. For every hour of footage, manual annotation takes two hours. Our model can give results about dozens of hours of footage in an hour work.

### 2. 📊 Accurate Data-Driven Insights:

By automatically classifying product usage, the tool provides accurate, quantitative insights about how a product is being used. This reduces the risk of human error that can come with video-based analysis or subjective judgment, ensuring that designers have access to reliable data to make informed decisions.

### 3. 📈 Scalability & Flexibility:

The system is designed to scale across different product types. While the initial prototype focused on products with GoPro sensors, the tool can be adapted for other products with sensor data, making it versatile for a wide range of applications in industrial design. The modularity of the system also makes it easy to add more sensors or adjust the analysis pipeline to suit different needs.

### 4. 👟 Improved Product Design:

By identifying patterns in how a product is used, the tool highlights potential design flaws that may not have been initially apparent. For example, it can detect that a product is used incorrectly, leading to straining. This allows Industrial Designers to rethink certain design elements and improve the product’s overall user experience.

### 5. 🔍 Novelty Detection (Anomaly Detection):

The tool incorporates novelty detection, which is particularly useful in identifying unusual or unexpected usage patterns. This feature is valuable for discovering when a product is being used in ways that were not anticipated, allowing designers to make proactive changes to the design or develop new features to meet unaddressed user needs.

## Challenges:
1. **📹Sensor Data Noise:** The sensor data from GoPro devices, especially accelerometers and gyroscopes, often contained noise. This posed a challenge in distinguishing between useful and irrelevant information, requiring sophisticated filtering techniques and data cleaning to improve the accuracy of the model.
2. **🏷️ Data Labeling:** Creating a framework where data and video are synchronised and can be labeled by a user took time to develop. The active learning algorithm helped reduce the amount of manual labeling required, but first creating a system that could handle this was a challenge.
3. **🤖 Model Overfitting:** Training the model on a limited dataset led to overfitting, where the classifier performed well on the training data but struggled with unseen data. To address this, we used techniques like cross-validation and increased the dataset size to improve the generalization of the model.
4. **⚙️ Data Processing:** Processing sensor data in was a challenge, particularly in ensuring the model could classify product usage quickly enough to provide actionable insights. Using advanced techniques like the Fast Fourier Transform helped extract relevant features efficiently.
5. **🧩 Diverse Product Types:** The initial prototype was focused on a specific product with GoPro sensors, and adapting the tool to work with other product types, such as wearables or home appliances, required additional adjustments. Creating a flexible architecture was necessary for future scalability.
6. **⏩ Performance**: Ensuring the tool could handle large datasets efficiently and provide real-time feedback during labeling and analysis was crucial. Optimizing the model’s performance and the tool’s responsiveness was a continuous challenge.

## Additional Resources:

- For a more detailed report, [click here](/assets/capstoneAI-report.pdf).
- For the GitHub repository: [Click here](https://github.com/iantmn/Capstone-AI-IoT).
- For the website we created for this project: [Click here](https://datacentricdesign.github.io/iot-ml-design-kit/).