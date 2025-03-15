# Detecting Irregular Medical Conditions with Azure AI Anomaly Detector

This guide demonstrates how to build a real-time patient monitoring solution using Azure Synapse Analytics and Azure AI Anomaly Detector. It walks you through creating a sample dataset, integrating with Azure Synapse, and running anomaly detection to flag abnormal patient vitals.

<img width="810" alt="image" src="https://github.com/user-attachments/assets/6f66db30-b958-475e-9daa-ba5019fc1444" />


## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Creating the Sample Dataset](#creating-the-sample-dataset)
- [Integrating with Azure Synapse Analytics](#integrating-with-azure-synapse-analytics)
- [Running the Anomaly Detection Notebook](#running-the-anomaly-detection-notebook)
- [Benefits](#benefits)
- [Conclusion](#conclusion)
- [References](#references)

## Overview

Early detection of abnormal patient vitals (heart rate, blood pressure, SpO₂) is critical in healthcare. This solution leverages Azure AI Anomaly Detector to automatically flag irregular conditions from IoT sensor data, enabling timely intervention and improved patient outcomes.

## Prerequisites

- **Azure Subscription** (create a free account if needed)
- **Azure Synapse Analytics Workspace** with an Azure Data Lake Storage Gen2 account
- **Spark Pool** in your Synapse workspace
- Basic knowledge of **PySpark**

## Creating the Sample Dataset

Use the following PySpark snippet to create a synthetic dataset simulating patient monitoring data. The dataset includes intentional anomalies.

```python
from pyspark.sql.functions import lit

df = spark.createDataFrame([
    # Patient 001 - Normal readings with some anomalies
    ("2025-01-01T08:00:00Z", "patient_001", 72.0, 120, 80, 98.0),   # Normal
    ("2025-01-01T08:10:00Z", "patient_001", 75.0, 122, 82, 97.0),
    ("2025-01-01T08:20:00Z", "patient_001", 78.0, 121, 81, 97.0),
    ("2025-01-01T08:30:00Z", "patient_001", 74.0, 119, 79, 98.0),
    ("2025-01-01T08:40:00Z", "patient_001", 120.0, 160, 100, 90.0),  # Anomaly: High HR & BP, Low SpO₂
    ("2025-01-01T08:50:00Z", "patient_001", 73.0, 118, 78, 98.0),
    ("2025-01-01T09:00:00Z", "patient_001", 71.0, 117, 77, 99.0),
    ("2025-01-01T09:20:00Z", "patient_001", 45.0, 85, 55, 85.0),    # Anomaly: Bradycardia & Hypotension, Low SpO₂
    ("2025-01-01T09:50:00Z", "patient_001", 200.0, 180, 110, 92.0),  # Anomaly: Severe tachycardia & hypertension

    # Patient 002 - Consistently normal readings
    ("2025-01-01T08:00:00Z", "patient_002", 68.0, 118, 76, 98.0),
    ("2025-01-01T08:10:00Z", "patient_002", 70.0, 119, 77, 99.0),
    ("2025-01-01T08:20:00Z", "patient_002", 69.0, 118, 76, 98.0),
    ("2025-01-01T08:30:00Z", "patient_002", 72.0, 120, 78, 98.0),

    # Patient 003 - Oxygen saturation drop
    ("2025-01-01T08:00:00Z", "patient_003", 76.0, 125, 85, 99.0),
    ("2025-01-01T08:10:00Z", "patient_003", 78.0, 126, 86, 98.0),
    ("2025-01-01T08:20:00Z", "patient_003", 75.0, 123, 83, 97.0),
    ("2025-01-01T08:40:00Z", "patient_003", 80.0, 130, 88, 88.0),   # Anomaly: Sudden drop in SpO₂
    ("2025-01-01T08:50:00Z", "patient_003", 77.0, 128, 86, 96.0),
], ["timestamp", "patient_id", "heart_rate", "systolic_bp", "diastolic_bp", "spo2"])

df.write.mode("overwrite").saveAsTable("patient_vital_signs")
```

*Note: Normal ranges are 60–100 bpm for heart rate, ~120/80 mmHg for blood pressure, and 95–100% for SpO₂.*

## Integrating with Azure Synapse Analytics

1. **Create a Spark Table:**  
   Upload the CSV file (or use the PySpark code above) into your Azure Data Lake Storage Gen2 account.  
   Right-click the file and select **New Notebook > Create Spark table**.

2. **Open the AI Services Wizard:**  
   In the Synapse Studio, right-click your Spark table, select **Machine Learning > Predict with a model**, and choose **Azure AI Anomaly Detector**.

3. **Configure the Model:**  
   - **Granularity:** Choose an appropriate value (e.g., minute or hourly).  
   - **Timestamp Column:** Set to the `timestamp` field.  
   - **Value Column:** Set to the field representing the vital sign (e.g., `heart_rate` for univariate or multiple fields for multivariate detection).  
   - **Grouping Column:** Use `patient_id` to analyze each patient's data separately.

4. **Run the Notebook:**  
   A PySpark notebook will be generated that uses the SynapseML library to call the Anomaly Detector API. Run all cells to process your dataset and view the anomaly results.

## Benefits

- **Early Intervention:** Immediate alerts enable faster clinical decisions.
- **Resource Efficiency:** Prioritize critical cases to optimize hospital resource management.
- **Enhanced Safety:** Continuous monitoring reduces the risk of missed diagnoses.
- **Scalable Analytics:** Cloud-based processing handles large volumes of data seamlessly.

## Conclusion

By integrating Azure AI Anomaly Detector with Azure Synapse Analytics, healthcare providers can build a robust, scalable system for real-time patient monitoring. This solution automates anomaly detection in patient vitals, facilitating early interventions and better clinical outcomes.

## References

- Microsoft Learn: [Tutorial: Anomaly detection with Azure AI services](https://learn.microsoft.com/en-us/azure/synapse-analytics/machine-learning/tutorial-cognitive-services-anomaly)  
- Sample dataset adapted for healthcare monitoring.

---

Feel free to clone this repository and experiment with the sample code. Contributions and improvements are welcome!
