
# 📘 Telecom ETL Project — Garage Education (SSIS Implementation)

**Source:**
Adapted from the *Garage Education* YouTube series by **Eng. Mostafa Alaa**.
Original implementation used **Scala** — this version reproduces the same logic using **SQL Server Integration Services (SSIS)**.

---

## 🎯 Project Objective

A **telecom company** has requested your expertise as an **ETL Developer** to design and build a pipeline that:

* Automatically extracts customer activity data from periodic CSV files.
* Cleans, validates, and transforms the data according to business rules.
* Loads the processed data into a **SQL Server database**.
* Tracks **data quality metrics (KPIs)** to ensure reliability and auditability.

---

## 📂 Business Scenario

* The system generates a new **CSV file every 5 minutes**.
* Each file contains **telecommunication activity data** (e.g., customer events, device information, network cells).
* You must **extract → validate → transform → load → audit → archive** each file.

---

## 📊 ETL Pipeline Overview

The following diagram shows the complete SSIS ETL workflow for telecom data:

<h3 align="center">📈 SSIS ETL Pipeline for Telecom Data</h3>
<p align="center">
  <img src="assets/etl_pipeline.png" alt="ETL SSIS Pipeline Diagram" width="600"/>
</p>



---

## 🧾 CSV File Structure

| Column Name    | Data Type | Length | Nullable | Example Value       | Meaning                                                                                         |
| -------------- | --------- | ------ | -------- | ------------------- | ----------------------------------------------------------------------------------------------- |
| **ID**         | Int       | —      | No       | 156                 | Unique record ID or transaction identifier                                                      |
| **IMSI**       | String    | 9      | No       | 310120265           | *International Mobile Subscriber Identity* — uniquely identifies a mobile subscriber (SIM card) |
| **IMEI**       | String    | 14     | Yes      | 490154203237518     | *International Mobile Equipment Identity* — unique identifier for the device                    |
| **CELL**       | Int       | —      | No       | 123                 | Cell Tower ID the device connected to                                                           |
| **LAC**        | Int       | —      | No       | 12                  | *Location Area Code* — identifies the network region                                            |
| **EVENT_TYPE** | String    | 1      | Yes      | 1                   | Type of network event (e.g., call, SMS, data session)                                           |
| **EVENT_TS**   | Timestamp | —      | No       | 2020-01-16 07:45:43 | Timestamp of the event occurrence                                                               |

---

## ⚙️ Transformation and Validation Rules

| Source Column  | Transformation Logic                            | Target Column    | Handling Invalid Cases             |
| -------------- | ----------------------------------------------- | ---------------- | ---------------------------------- |
| **ID**         | As-is                                           | `Transaction_ID` | —                                  |
| **IMSI**       | As-is                                           | `IMSI`           | Reject if null                     |
| **IMSI**       | Lookup in reference table → get `subscriber_id` | `Subscriber_ID`  | Use `-99999` if not found          |
| **IMEI**       | First 8 characters                              | `TAC`            | Use `-99999` if null or < 15 chars |
| **IMEI**       | Last 6 characters                               | `SNR`            | Use `-99999` if null or < 15 chars |
| **IMEI**       | As-is                                           | `IMEI`           | —                                  |
| **CELL**       | As-is                                           | `CELL`           | Reject if null                     |
| **LAC**        | As-is                                           | `LAC`            | Reject if null                     |
| **EVENT_TYPE** | As-is                                           | `EVENT_TYPE`     | —                                  |
| **EVENT_TS**   | Validate format → `YYYY-MM-DD HH:MM:SS`         | `EVENT_TS`       | Reject if null or invalid          |

---

## 🚫 Handling Rejected Records

Rejected records are:

* Stored in a **separate SQL table**: `Rejected_Transactions`
* Include:

  * All source columns
  * A `File_Name` column (original CSV name)
  * A `Rejection_Reason` column (optional for audit)

---

## 🧮 Key Performance Indicators (KPIs)

These KPIs help you **monitor ETL performance** and **data quality** for every file processed.

| KPI                           | Description                                   | Formula                                               | Example                   |
| ----------------------------- | --------------------------------------------- | ----------------------------------------------------- | ------------------------- |
| **Total Records (Input)**     | Total rows in the incoming CSV                | `COUNT(*) from CSV`                                   | 10,000                    |
| **Valid Records (Loaded)**    | Records successfully loaded into the database | `COUNT(*) from Target_Table`                          | 9,800                     |
| **Rejected Records**          | Records failing validation rules              | `COUNT(*) from Rejected_Table`                        | 200                       |
| **Rejection Rate (%)**        | Percentage of records rejected                | `(Rejected / Total) × 100`                            | (200 / 10,000) × 100 = 2% |
| **Data Traceability**         | Each record links back to its source file     | CSV file name stored in both Target & Rejected tables | `file_20250101_1200.csv`  |
| **ETL Run Duration**          | Time taken to process one file                | End time - Start time                                 | 00:00:25                  |
| **Timestamp Validation Rate** | Ratio of valid timestamp values               | `(Valid_Timestamps / Total) × 100`                    | 99.5%                     |

---

## 🧩 Database Tables Overview

### 1. `Telecom_Transactions`

✅ Stores **clean, validated** data.

### 2. `Rejected_Transactions`

🚫 Stores **invalid** records with rejection reasons.

### 3. `ETL_Audit_Log`

📊 Records KPI values and metadata for each CSV file processed.

| Column                    | Description               |
| ------------------------- | ------------------------- |
| `File_Name`               | Name of the processed CSV |
| `Total_Records`           | Total rows read           |
| `Valid_Records`           | Successfully loaded rows  |
| `Rejected_Records`        | Failed rows               |
| `Start_Time` / `End_Time` | ETL duration              |
| `Processed_By`            | SSIS package or job name  |

---

## 🧠 ETL Flow — SSIS Pipeline Overview

Below is a visual representation of the process.

```mermaid
flowchart LR
A[📁 CSV File (Every 5 mins)] --> B[📤 Extract Data (Flat File Source)]
B --> C[🔍 Transform Data (Derived Column / Lookup)]
C --> D{✅ Validation Rules Pass?}
D -->|Yes| E[💾 Load to SQL: Telecom_Transactions]
D -->|No| F[🚫 Load to SQL: Rejected_Transactions]
E --> G[🧮 Log KPIs (ETL_Audit_Log)]
F --> G
G --> H[📦 Move CSV to Archive Folder]
```

---

## 🧰 SSIS Components Used

| SSIS Component         | Role in Pipeline                                 |
| ---------------------- | ------------------------------------------------ |
| **Flat File Source**   | Reads input CSV                                  |
| **Derived Column**     | Data transformations (substring, default values) |
| **Lookup**             | Gets `Subscriber_ID` from reference table        |
| **Conditional Split**  | Separates valid and invalid records              |
| **OLE DB Destination** | Loads records into SQL tables                    |
| **File System Task**   | Moves processed files to archive                 |
| **Execute SQL Task**   | Logs KPI metrics in `ETL_Audit_Log`              |

---

## 🧩 Post-Processing Steps

1. After all transformations and loads:

   * Move the processed CSV to an **archive folder** (e.g., `D:\Telecom\Processed\`).
   * Log the KPI metrics in `ETL_Audit_Log`.

2. Optionally send **notifications or email alerts** if:

   * Rejection rate exceeds threshold (e.g., >5%).
   * Timestamp validation fails.
   * CSV file missing expected columns.

---

## 🏁 Final Deliverables

By completing this project, you will have:

✅ A working **ETL solution using SSIS**
✅ Automatic **data validation and cleansing**
✅ Complete **audit trail and KPI monitoring**
✅ Understanding of **Telecom Data Structures**
✅ Real-world ETL workflow knowledge

---

