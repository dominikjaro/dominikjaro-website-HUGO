```markdown
# 🌥️ **GCP Data Transfer Between Projects** 🌐

Welcome to the **GCP Data Transfer** guide! This repository provides a comprehensive step-by-step process to efficiently transfer data between Google Cloud Platform (GCP) projects.

## 📅 Date
**2024-09-30**

## 🗂️ Categories
- Cloud
- DevOps
- Shell

## 🏷️ Tags
- GCP
- Automation
- Shell

## 🎯 **Objective**

Transfer data from a **source GCP project** to a **destination project**, including:
- **Cloud Storage Buckets**
- **Firestore (Datastore)**
- **BigQuery Datasets**

### 🛠️ **Create in Terraform:**
- Buckets
- BQ Datasets
- Datastore DB

## 🔧 **Pre-Requisites**
- Access to both **source** and **destination** GCP projects.
- **Gsuite alias account** for secure data transfer management.
- Necessary scripts - [PATH_TO_THE_SCRIPTS]

## 📜 **Instructions**

1. ### 🆕 **Create a GSuite Alias Account**
    - Set up a **GSuite alias account** exclusively for data transfer.
    - Ensure **minimum required privileges** (e.g., access only to relevant buckets, Firestore, and BigQuery).

2. ### 🔐 **Grant Access in Destination Project**
    - Assign required permissions in the **Google Cloud Console** under the specific **Dev folder** or relevant folder in the **Destination Project**.
    - Ensure sufficient privileges for the alias account in the **Destination Project**.

3. ### 🔑 **Log into GCP with the Test Account**
    - Use the alias account to log into **Google Cloud Platform (GCP)**.
    - Validate access by navigating to the **destination project**.

4. ### 🚫 **Revoke Excess Access**
    - Prevent unintentional access by revoking any excess permissions in your terminal:
    ```bash
    gcloud auth revoke
    gcloud auth login
    gcloud projects list
    gcloud config get project
    ```

5. ### 🗑️ **Delete Resources in the Destination Project**
    - **Storage Buckets**: Navigate to **Cloud Storage** in the **Destination Project** and delete existing buckets that need recreation or refresh.
    - **Firestore Database**: Go to **Firestore** in the **Destination Project** and delete the database if required.

6. ### 📤 **Export Firestore Database from Source Project**
    - In the **Source Project**, export the Firestore database to a **Cloud Storage bucket**.

7. ### 🛠️ **Create Resources in Destination Project - Terraform**
    - **Cloud Storage Buckets**: Use **Terraform** to create necessary buckets in the **Destination Project**, ensuring configurations match the source.
    - **BigQuery Datasets**: Set up **BigQuery datasets** using **Terraform**, matching source schemas and settings.
    - **Firestore (Datastore)**: Create the necessary **Firestore in Datastore mode** database in the **Destination Project** using **Terraform**.

8. ### 📂 **Locate Bootstrap Scripts**
    - Find the [PATH_TO_THE_SCRIPTS] scripts in the platform repo to automate data seeding into the destination project.

9. ### 🔄 **Synchronize Data**
    - **Cloud Storage**:
    ```bash
    ./seed-gcs.sh [SOURCE] [DESTINATION]
    ```
    - **Firestore (Datastore)**:
    ```bash
    ./seed-datastore.sh [DESTINATION]
    ```
    - **BigQuery**:
    ```bash
    ./seed-bigquery.sh [SOURCE] [DESTINATION]
    ```

10. ### 📈 **Create Composite Indexes in Datastore (Destination)**
    - Define indexes in an `index.yaml` file based on query requirements.
    - Create indexes using:
    ```bash
    gcloud firestore indexes create index.yaml
    ```

11. ### 🛠️ **Update Service Config in Datastore**
    - In **Firestore (Datastore)**, update the **Service Config** for the application:
        - **Namespace**: `feefo`
        - **Kind**: `ServiceConfig`
        - Update the hostname to point to the correct VM within the destination project.

12. ### 🔄 **Bounce the Pods and VMs**
    - After transferring data and setting up services, **restart the pods and VMs** to ensure all services start fresh and utilize the new data.

---

🚀 **You're all set!** Your data should now be successfully transferred between GCP projects. If you encounter any issues, please refer to the troubleshooting section or contact support.

```
