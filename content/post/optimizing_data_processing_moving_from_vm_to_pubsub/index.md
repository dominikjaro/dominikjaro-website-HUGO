---
title: "🚀 Optimizing Data Processing: Moving from VM to Pub/Sub 🔄"
date: 2024-10-25
description: "A high CPU usage issue in a VM caused delays in data transfer from Datastore to BigQuery. Switching to a Pub/Sub-based approach improved scalability, efficiency, and reliability. 🚀🔄"
categories: ["Cloud", "Data Engineering"]
tags: ["terraform", "gcp", "pubsub"]
image: "vm.jpg"
---

### 🛠 Background

I encountered an issue where a VM was experiencing high CPU usage. This problem affected data processing between Datastore and BigQuery, causing inconsistencies in sales and campaign reports.

### ⚠️ Issue

We observed anomalies in sales and campaign data. For example, a download that should have retrieved 330 sales only returned 4. Upon investigation, it became clear that data from Datastore was not being written to BigQuery fast enough.

The issue stemmed from a VM (`services-4`) responsible for handling this data transfer. The VM operates by:

1. 🔍 Fetching entity IDs from Datastore.
2. 📤 Writing them to BigQuery.
3. 😴 Entering a sleep state before the next cycle.

However, when handling a large batch of data, the VM would retrieve an excessive amount, enter sleep mode, and fail to complete the job efficiently. This caused CPU usage to spike and slowed down the entire process.

### 🔍 Root Cause

On Monday after lunch, a colleague ran a script in BigQuery or Datastore, which further burdened the VM. While no writes were missed in the Datastore-to-BigQuery pipeline, processing was significantly delayed due to the added workload.

### 🔧 Temporary Fix

To mitigate the issue, I temporarily increased the CPU cores of the VM from 2 to 4, which allowed it to catch up with the backlog. However, when the cores were reduced back to 2 the next day, the high CPU usage returned, indicating that the underlying issue was still unresolved.

### 🚀 Long-Term Solution

Rather than relying on a single VM-based script, we plan to transition to a more scalable and modern approach using **Pub/Sub**. This will allow for:

- ⚡ **Event-driven processing** instead of batch jobs.
- 📈 **Better scalability** to handle variable workloads.
- 🔄 **Improved efficiency**, reducing CPU strain on a single VM.

Using Pub/Sub is considered a **best practice** for this kind of data processing today. This transition will help ensure reliable and timely data transfer between Datastore and BigQuery while avoiding the pitfalls of a monolithic VM-based solution.

### 🎯 Conclusion

This incident highlighted the limitations of a VM-centric approach for handling large-scale data transfers. By adopting Pub/Sub, we can achieve **better performance, scalability, and resilience** in our data processing pipeline. ✅