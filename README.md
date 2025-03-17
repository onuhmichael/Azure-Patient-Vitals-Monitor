<img width="800" alt="image" src="https://github.com/user-attachments/assets/6e4b8969-0e7a-4462-ac43-2ee642312fa6" />

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Simulating IoT Hub Data with a Python Script](#simulating-iot-hub-data-with-a-python-script)
- [Configuring Azure Stream Analytics](#configuring-azure-stream-analytics)
  - [Setting Up the IoT Hub Input](#setting-up-the-iot-hub-input)
  - [Output 1: All Data to Azure Data Lake Gen2 (CSV)](#output-1-all-data-to-azure-data-lake-gen2-csv)
  - [Output 2: Anomaly Detection to Power BI](#output-2-anomaly-detection-to-power-bi)
- [Setting Up Azure Data Lake Gen2 for CSV Output](#setting-up-azure-data-lake-gen2-for-csv-output)
- [Visualizing Anomalies in Power BI](#visualizing-anomalies-in-power-bi)
- [Benefits](#benefits)
- [Conclusion](#conclusion)
- [Additional Resources](#additional-resources)

---

## Overview

Real-time monitoring of patient vital signs is essential for early intervention in healthcare. In this project, sensor data (e.g., heart rate, blood pressure, SpO₂) is simulated locally and sent to Azure IoT Hub. Azure Stream Analytics picks up the real-time data and uses two queries:

1. **Data Archive Query:** Selects *all* incoming data and writes it to Azure Data Lake Gen2 as CSV files for storage and later analysis.
2. **Anomaly Detection Query:** Evaluates each record against set thresholds and only outputs messages flagged as “Anomaly” to Power BI—enabling a real-time dashboard that highlights critical conditions.

This approach minimizes cost and complexity while delivering immediate, actionable insights.

---

## Prerequisites

- **Azure Subscription:** If you don’t have one, you can create a free account.  
- **Azure IoT Hub:** Set up an IoT Hub with at least one device (for simulation).  
- **Azure Data Lake Storage Gen2:** Create an account (and a file system) to store output files.  
- **Azure Stream Analytics (ASA):** Create a job to process IoT Hub events.  
- **Power BI Service/Workspace:** For building real-time and interactive dashboards.  
- **Python Environment:** Installed on your development machine with the required packages.

---

## Simulating IoT Hub Data with a Python Script

Save the script below as `IOT_HUB_SIMULATOR.py` on your local machine. This script uses the Azure IoT Device SDK to simulate patient vital sign data and sends it to your IoT Hub.

```python
from azure.iot.device import IoTHubDeviceClient
import json
import time
import random

# IoT Hub Device Connection String (replace with your actual connection string)
CONNECTION_STRING = "HostName=mikeiothub.azure-devices.net;DeviceId=my_iot_device;SharedAccessKey=YOUR_SHARED_ACCESS_KEY"

# Function to generate simulated patient vitals
def generate_patient_data():
    return {
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "patient_id": "patient_001",
        "heart_rate": random.randint(50, 120),       # Heart rate in beats per minute
        "systolic_bp": random.randint(90, 160),        # Systolic blood pressure
        "diastolic_bp": random.randint(60, 100),       # Diastolic blood pressure
        "spo2": random.uniform(85.0, 100.0)            # Oxygen saturation level
    }

# Function to send data continuously to IoT Hub
def send_data_to_iot_hub():
    client = IoTHubDeviceClient.create_from_connection_string(CONNECTION_STRING)
    print("Connecting to IoT Hub...")
    client.connect()
    try:
        while True:
            data = generate_patient_data()
            message = json.dumps(data)
            client.send_message(message)
            print(f"Sent: {message}")
            time.sleep(2)  # Sends data every 2 seconds
    except KeyboardInterrupt:
        print("Simulation stopped.")
    finally:
        client.disconnect()

if __name__ == "__main__":
    send_data_to_iot_hub()
```

> **Note:** Replace `YOUR_SHARED_ACCESS_KEY` with your actual shared access key from IoT Hub.

Run this script to begin sending simulated patient data into your IoT Hub.

---

## Configuring Azure Stream Analytics

With your IoT Hub receiving events, create an Azure Stream Analytics (ASA) job with the following settings:

### Setting Up the IoT Hub Input

1. In your ASA job’s **Inputs** section, add a new input.
2. Choose **IoT Hub** as the source.
3. Enter the connection details (IoT Hub name, endpoint, etc.). Name this input (for example, `IoTHubInput`).

### Output 1: All Data to Azure Data Lake Gen2 (CSV)

This output saves every incoming record to Azure Data Lake Gen2 for storage and historical analysis.

1. Create an output in ASA and select **Azure Data Lake Storage Gen2** as the sink.
2. Configure output details:
   - **File Format:** CSV
   - **File Path:** Use a partitioned folder structure such as `patient-data/{date}/{hour}` to keep your data organized.
   - **Output Alias:** Name it, for example, `ADLS_Output`.

3. **Query 1:**  
   This simple query sends every event received to Data Lake Gen2.

   ```sql
   SELECT *
   INTO [ADLS_Output]
   FROM [IoTHubInput];
   ```

### Output 2: Anomaly Detection to Power BI

This output processes data in real time to detect anomalies and sends only flagged events to Power BI.

1. Create another output in ASA and select **Power BI** as the sink.
2. Authenticate with your Power BI workspace, and create or select a dataset and table (e.g., `PatientAnomalies`).
3. **Query 2:**  
   This query evaluates each event and passes along only those records that breach thresholds—indicating an anomaly.

   ```sql
   SELECT
       timestamp,
       patient_id,
       heart_rate,
       systolic_bp,
       diastolic_bp,
       spo2,
       CASE 
           WHEN heart_rate > 100 OR heart_rate < 60 THEN 'Anomaly'
           WHEN systolic_bp > 140 OR diastolic_bp > 90 THEN 'Anomaly'
           WHEN spo2 < 90 THEN 'Anomaly'
           ELSE 'Normal'
       END AS status
   INTO [PowerBI_Output]
   FROM [IoTHubInput]
   WHERE 
       (heart_rate > 100 OR heart_rate < 60)
       OR (systolic_bp > 140 OR diastolic_bp > 90)
       OR (spo2 < 90);
   ```

This query does the following:
- **Preserves** key vital sign data.
- **Applies a `CASE` expression** to decide if the record is "Anomaly" or "Normal."  
- **Filters events** so that only records flagged as anomalies are sent to Power BI—keeping your dashboard focused on actionable alerts.

---

## Setting Up Azure Data Lake Gen2 for CSV Output

1. In your Azure Storage account, ensure you have Data Lake Gen2 enabled.
2. Create a file system (for example: `patient-anomalies`).
3. Within your ASA output configuration, set the folder path and file naming convention (e.g., `patient-data/{date}/{hour}/output.csv`).
4. The ASA job will write CSV files automatically at defined intervals based on event watermarks.

---

## Visualizing Anomalies in Power BI

1. **Connect to Your Power BI Dataset:**
   - Log in to Power BI Service and open your workspace.
   - Locate the dataset (e.g., `PatientAnomalies`) that ASA writes to.
2. **Build a Dashboard:**
   - Create visualizations such as time-series graphs for heart rate, blood pressure, and SpO₂.
   - Utilize card visuals or color-coded indicators to highlight anomalies.
   - Set up data‑driven alerts in Power BI to notify your team when a critical anomaly is detected.
3. **Iterate and Optimize:**
   - Experiment with filtering, aggregation, and drill-down capabilities to provide clinical teams with actionable insights.

---

## Benefits

- **Cost Efficiency:** By sending all data for storage in ADLS and filtering only anomalies to Power BI, you save on processing costs.
- **Real-Time Insights:** Critical conditions surface immediately thanks to the Power BI integration.
- **Scalability:** ADLS Gen2 handles high volumes of IoT data while ASA and Power BI provide a responsive and adaptive analytics platform.
- **Actionable Data:** Automatic anomaly detection allows healthcare teams to intervene swiftly, potentially saving lives.

---

## Conclusion

By following this guide, you can implement a robust real-time patient monitoring solution with minimal overhead. Simulated IoT sensor data from your Python script feeds into IoT Hub, ASA routes the data both to long-term storage (CSV files in ADLS Gen2) and a real-time dashboard (Power BI) for anomaly detection. This architecture not only reduces complexity but also provides targeted clinical insights to drive prompt intervention.

---

## Additional Resources

- **Stream Analytics Query Patterns:** [Common Patterns](https://go.microsoft.com/fwLink/?LinkID=619153)  
- **Stream Analytics Query Language Documentation:** [Documentation](https://docs.microsoft.com/stream-analytics-query/query-language-elements-azure-stream-analytics)  
- **Azure IoT Hub Documentation:** [Azure IoT Hub](https://docs.microsoft.com/azure/iot-hub/)  
- **Azure Data Lake Gen2 Documentation:** [Azure Data Lake Storage Gen2](https://docs.microsoft.com/azure/storage/blobs/data-lake-storage-introduction)  
- **Power BI Real-Time Streaming Data:** [Streaming Datasets in Power BI](https://docs.microsoft.com/power-bi/connect-data/service-real-time-streaming)

---

By carefully following this guide, you should be able to implement the project end-to-end. 
