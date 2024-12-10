---
title: "🌥️ Moving data between GCP projects/buckets 🌐"
date: 2024-09-30
description: "I was tasked with setting up a new development environment for testing and development. It lacked the necessary data, so I had to carefully transfer it from one project to another. This document outlines my experience and the steps I took to transfer data between Google Cloud Projects, including Cloud Storage buckets, Firestore (Datastore), and BigQuery datasets. This guide can be helpful when creating a new environment and seeding it with data from an existing project."
categories: ["Cloud","shell"]
tags: ["GCP","Automation","shell"]
image: "GCP buckets.png"
---

### 🎯 Objective

Transfer data from a source GCP project to a destination project, including Cloud Storage buckets, Firestore (Datastore), and BigQuery datasets.

**Create in Terraform:**

- Buckets
- BQ Datasets
- Datastore db

### 🛠️ Pre-Requisites

- Access to both source and destination GCP projects.
- **Gsuite alias account** for managing the data transfer securely.
- The scripts you will need - [GCP-data-transfer-between-projects-GitHub-repo](https://github.com/dominikjaro/GCP-data-transfer-between-projects.git)

## 📋 Instructions

1. #### Create a GSuite Alias Account
    - Set up a **GSuite alias account** that will be used solely for this data transfer.
    - Ensure the account has **minimum required privileges** (e.g., access to only relevant buckets, Firestore, and BigQuery).

2. #### Grant Access in Destination Project
    - Grant the required permissions in the **Google Cloud Console** under the specific **Dev folder** or relevant folder in the **Destination Project**.
    - Ensure this account has sufficient privileges in the **Destination Project**.

3. #### Log into GCP with the Test Account
    - Use the alias account to log into Google Cloud Platform (GCP).
    - Validate access by navigating to the **destination project**.

4. #### Revoke Excess Access
    - To prevent any unintentional access, revoke any excess permissions in your terminal:
    - This ensures that only the intended project is accessed during the transfer.

    ```bash
    gcloud auth revoke
    gcloud auth login
    gcloud projects list
    gcloud config get project
    ```

5. #### Delete Storage Buckets in the Destination Project 
    - Navigate to **Cloud Storage** in the **Destination Project** and delete any existing buckets that need to be recreated or refreshed.

6. #### Delete the Firestore Database in the Destination Project
    - Go to **Firestore** in the **Destination Project** and delete the database if required. Ensure that you are aware of the data that needs to be replaced.

7. #### Export Firestore Database from Source Project
    - In the **Source Project**, export the Firestore database to a **Cloud Storage bucket**.

8. #### Create Storage Buckets in the Destination Project - Terraform 
    - Use **Terraform** to create the necessary **Cloud Storage buckets** in the **Destination Project**.
    - Ensure that bucket configurations match the source project for consistency.

9. #### Create BigQuery Datasets in the Destination Project - Terraform 
    - Use **Terraform** to set up the **BigQuery datasets** in the **Destination Project**.
    - Ensure that dataset schemas and settings match the source.

10. #### Create Firestore (Datastore) in the Destination Project - Terraform 
    - Again, using **Terraform**, create the necessary **Firestore in Datastore mode** (default) database in the **Destination Project**.

11. #### Locate Bootstrap Scripts 
    - Find the [`bootstrap scripts in my GitHub repository here`](https://github.com/dominikjaro/GCP-data-transfer-between-projects.git).
    - These scripts will automate the seeding of data into the destination project.

12. #### Synchronize Cloud Storage Data 
    - Run the script [`seed-gcs.sh`](https://github.com/dominikjaro/GCP-data-transfer-between-projects/blob/master/seed-data/seed-gcs.sh), which synchronizes data between Cloud Storage buckets from the source to the destination project.
    - This step ensures that all files from the source bucket are transferred to the destination bucket.

    ```bash
    ./seed-gcs.sh [SOURCE] [DESTINATION]
    ```

13. #### Synchronize Firestore (Datastore) Data 
    - Use the [`seed-datastore.sh`](https://github.com/dominikjaro/GCP-data-transfer-between-projects/blob/master/seed-data/seed-datastore.sh) script to synchronize the exported **Datastore** content from the **source bucket** to the **destination bucket**.
    - This script also imports data into the **Firestore (Datastore)** in the **Destination Project**.
    - Key environment variables to set:
        - SRC_LOCATION: Source project location.
        - SRC_BUCKET: Name of the source bucket.
        - SNAPSHOT: Snapshot of the data.
        - DEST_PROJECT: Destination project name.

    ```bash
    ./seed-datastore.sh [DESTINATION]
    ```

14. #### Create Composite Indexes in Datastore (Destination) 
    - Define indexes in an [`index.yaml`](https://github.com/dominikjaro/GCP-data-transfer-between-projects/blob/master/seed-data/index.yaml) file based on query requirements.
    - Use gcloud to create the indexes:

    ```bash
    gcloud firestore indexes create index.yaml
    ```

15. #### Synchronize BigQuery Data 
    - Run the [`seed-bigquery.sh`](https://github.com/dominikjaro/GCP-data-transfer-between-projects/blob/master/seed-data/seed-bigquery.sh) script to synchronize BigQuery data from the source to the destination project.

    ```bash
    ./seed-bigquery.sh [SOURCE] [DESTINATION]
    ```

16. #### Update Service Config in Datastore 
    - In **Firestore (Datastore)**, update the **Service Config** for the application:
        - Namespace: []
        - Kind: []
        - Update the hostname to point to the correct VM within the destination project.

17. #### Bounce the Pods and VMs 
    - After all data is transferred and services are set up, **restart the pods and VMs** to ensure all services start fresh and use the new data.