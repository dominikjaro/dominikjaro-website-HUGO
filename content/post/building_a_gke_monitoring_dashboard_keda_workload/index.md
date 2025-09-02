---
title: "📊 Building a GKE Monitoring Dashboard for a KEDA Workload"
date: 2025-09-02
description: "Running event-driven microservices on GKE using KEDA is fantastic for cost efficiency and scalability."
categories: ["Cloud", "Kubernetes", "IaC"]
tags: ["Automation", "Monitoring", "Terraform"]
image: "monitoring3.png"
---

Running event-driven microservices on GKE using KEDA is fantastic for cost efficiency and scalability. But when something goes wrong, you need a way to quickly see what's happening. That's why I created a dedicated monitoring dashboard in Google Cloud, and I made sure to define it all in Terraform.

## ⚙️ Our Monitoring Strategy

The strategy was two-fold:

1. A Dashboard: To provide a single pane of glass for visual, real-time health checks.

2. Alerting Policies: To proactively notify our team in Microsoft Teams when specific issues arise.

All of this was defined as code using Terraform, which is crucial for version control and repeatability.

## 📈 The Key Dashboard Widgets

Our dashboard focuses on the most critical metrics for a KEDA-driven, Pub/Sub-based service. The goal isn't just to see if the service is up, but to understand if it's processing events efficiently.

1. Pub/Sub Message Backlogs & Errors

These widgets tell us if messages are piling up or failing to process. They're the first place to look when something goes wrong.

   - High Unacknowledged Messages: This is the most direct indicator of a backlog. If the count of unacknowledged messages starts to grow, it means the service is failing to keep up with the incoming message volume. It's the primary scaling signal for KEDA.

   - Dead-Letter Messages: This widget tracks messages that fail to be processed after a certain number of retries and are moved to a dead-letter queue. Any activity here is a clear sign of persistent message processing failures that require immediate attention.

   - Expired ACK Deadlines Count: This is a crucial metric for catching slow processing. It shows how many messages exceeded their acknowledgment deadline. A high number here means our pods are taking too long to process messages, even if they're not failing entirely.

1. Delivery Latency
This tells us how long messages are waiting in the system from end to end.

   - Publish to Ack Delta: This measures the total time from when a message is published to the topic to when it is successfully acknowledged by our service. It's an excellent measure of our overall system latency.

   - Pull to Ack Delta: This measures the time from when a message is delivered to the service to when it is acknowledged. It tells us how long our application is taking to process a message.

1. Throughput and Service Health
These widgets give us a high-level view of our service's performance.

   - Published Message Count: A simple but vital metric to track the volume of messages our service is receiving. It helps us understand the load and correlate it with any performance changes.

   - Delivery Latency Health Score: This is a custom score that combines various latency metrics into a single, easy-to-read number. It gives us an immediate, at-a-glance view of service health.

   - Log Panels: I included a dedicated panel to display log entries. This is essential for immediate troubleshooting. By including filters for severity=ERROR, we can see stack traces and error messages directly on the dashboard. This saves time and avoids having to navigate to a separate log explorer.

---
By combining these metrics, the dashboard provides a comprehensive view of our service's health, throughput, and error rates. It allows us to not only react to problems but also to proactively identify bottlenecks before they become critical issues.
