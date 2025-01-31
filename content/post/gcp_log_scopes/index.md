---
title: "📊 Streamlining Log Management with Log Scopes"
date: 2025-01-31
description: "Managing logs across multiple projects was challenging, requiring frequent context switching in GCP Logs Explorer. By implementing log scopes, we centralized logging into a single view, improving troubleshooting and monitoring. Using Terraform, we automated log scope creation, efficiently handling project limits and ensuring scalability. 🚀📊"
categories: ["Cloud", "IaC"]
tags: ["terraform", "gcp", "logging"]
image: "logs-explorer.png"
---

## Log Scope

At our company, managing logs across multiple projects was becoming a challenge. Every team had their own logging setup, making it difficult to get a unified view of all log data. When an issue arose, we had to jump between different projects in **GCP** **Logs Explorer**, slowing down troubleshooting and analysis.

To solve this, we aimed to **consolidate logging and monitoring into a single project.** The goal was simple: when using **Logs Explorer** in the **`xyz`** project, we should be able to search and find logs from multiple other projects.

This is where **log scopes** came in. By defining a custom log scope, we listed all relevant projects and log views under a single umbrella. This meant:

✅ A centralized view of logs across multiple projects.

✅ Faster and more efficient troubleshooting.

✅ Improved access control and monitoring.

Now, with a single query in **Logs Explorer**, we can analyze logs across all related projects without switching contexts—making our logging workflow seamless and efficient. 📊

## **What I Learned from It**

Through this process, I learned that **log scopes allow us to consolidate projects and views into a single log scope**. When you create a GCP project, folder, or organization, **Logging automatically creates a log scope named _Default**. This default log scope includes the project in which it was created, and the logs are stored in a **log bucket**. However, by leveraging custom log scopes, we can **extend visibility across multiple projects**—making log analysis much more efficient.

## **How I Used Terraform to Automate It**

This setup was implemented for our **service_dev** and **service_prod** projects and environments. However, a challenge arose:

•	**A single log scope can include only up to 5 projects.**

•	**We have 11 dev environments, each with 4 projects, plus the service project.**

To handle this, I used **Terraform** to automate the process:

1.	**Fetched the dev projects dynamically** using a Terraform data source from our Dev folder in GCP.

2.	**Created a map of environment instances to their projects** using **locals**.

3.	**Used a** for_each **loop** to generate a **log scope per dev environment**, ensuring each contained no more than 5 projects.

This approach ensured that:

✅ The Terraform code is **DRY (Don’t Repeat Yourself)**.

✅ There are **no manual dependencies**—it dynamically fetches projects.

✅ It efficiently scales with our environment structure.

```bash
####################################
# Fetch projects from the Dev folder
####################################
data "google_projects" "dev_projects" {
    filter = "parent.id:${var.dev_folder_id} lifecycleState=ACTIVE"
}

########################################################################
# Create a local variable to map environment instances to their projects
########################################################################
locals {
    # Define environment instances
    environment_instances = ["dev-0", "dev-1", "dev-2", "dev-3", "dev-4", "dev-5", "dev-6", "dev-7", "dev-8", "dev-90", "dev-rc"]

    # Create map of environment instances to their projects
    environment_project_map = {
        for env in local.environment_instances : env => toset([
            for project in var.dev_project_ids : project
            if try(length(regexall(".*-${env}(-.*)?$", project)) > 0, false)
        ])
    }
}

########################################
# Create Log Scopes for the dev projects
########################################
resource "google_logging_log_scope" "dev_project_logs" {
    for_each = local.environment_project_map

    parent         = "projects/${var.service_project_id}"
    name           = "projects/${var.service_project_id}/locations/global/logScopes/${each.key}"
    resource_names = concat([for project in each.value : "projects/${project}"], ["projects/${var.service_project_id}"])
}
```

*Note: This Terraform code does not include everything. I’m not showcasing the variables or how I pass values from the root module to its submodule.*

This Terraform configuration automates the creation of log scopes for each dev environment while adhering to the **5-project limit per scope**. Now, all logs are easily accessible within a **centralized Logs Explorer**—without any manual configuration!

## **Production Environment**

Setting up **service_prod** was much easier since we have fewer projects there. We didn’t need to worry about splitting projects into multiple scopes, so a **single log scope** was enough to consolidate all production logs into one view. This made the setup straightforward while still providing the same benefits of centralized logging and monitoring.

## **Documentation for Other Teams**

To ensure everyone can easily find and use these log scopes, I also **created a brief guide** for other teams. This document explains **where to locate the log scopes, how to use them in Logs Explorer, and best practices for troubleshooting**. Now, teams can efficiently access the logs they need without any confusion.

