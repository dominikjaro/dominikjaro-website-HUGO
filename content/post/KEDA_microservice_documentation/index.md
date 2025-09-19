---
title: "🚀 KEDA Microservice Documentation"
date: 2025-02-07
description: "📌 This document outlines my experience implementing a Kubernetes microservice using KEDA (Kubernetes Event-Driven Autoscaling) to scale based on Pub/Sub messages. The focus is on how KEDA enables efficient scaling of workloads in response to event-driven triggers and the key lessons I learned along the way."
categories: ["Kubernetes"]
tags: ["Terraform", "GCP", "PubSub"]
image: "keda.png"
---

## 🔧 My Approach

### 🏗 Infrastructure as Code (IaC)

I provisioned all required cloud infrastructure resources using Terraform, including:

- 🔑 **Workload Identity and IAM permissions:** To grant the microservice secure access to necessary GCP services.

- 📬 **Pub/Sub Topics and Subscriptions:** Configured for event-driven message processing.

### 🚀 Implementing the Kubernetes Microservice with KEDA

To enable event-driven autoscaling of the microservice, I used KEDA. This allowed the service to dynamically scale based on the number of messages in a Pub/Sub subscription and CPU utilization.

### 🤔 Why I Chose KEDA

Through my research and hands-on work, I found that KEDA provides:

- ⚡ **Efficient Scaling:** It scales the service based on event-driven triggers, ensuring optimal resource utilization.

- 🎯 **Granular Control:** It allows fine-tuned scaling based on Pub/Sub message load and CPU usage.

- 🔗 **Seamless Integration:** Works natively with Kubernetes and integrates well with Helm.

## ⚙️ KEDA Configuration

The KEDA configuration I used for this microservice is as follows:

```yaml
keda:
  enabled: true
  pollingInterval: 60  # ⏳ Polls every 60 seconds
  maxReplicaCount: 5    # 📈 Maximum replicas allowed
  minReplicaCount: 0    # 📉 Allows scaling down to zero
  cooldownPeriod: 600   # 🕒 Time to wait before scaling down
  spec:
    podIdentity:
      provider: gcp  # Uses GCP workload identity 🔐
  fallback:
    failureThreshold: 3
    replicas: 1
  advanced:
    restoreToOriginalReplicaCount: "true"
  authenticationRef:
    name: keda-trigger-auth-gcp-credentials
  triggers:
    - type: gcp-pubsub
      authenticationRef:
        name: keda-trigger-auth-gcp-credentials
      metadata:
        subscriptions:
          - "projects/my-project/subscriptions/my-subscription"
        mode: SubscriptionSize
        value: "1000000"  # 📊 Scale up when message count exceeds 1M
        activationThreshold: "1" # 🔄 Scale up when at least one message is present
    - type: cpu
      metadata:
        type: Utilization
        value: "80"   # 🚀 Scale up if CPU utilization exceeds 80%
        activationThreshold: "40" # ⚖️ Scale up when CPU utilization exceeds 40%
```

## 🤝 Collaboration and Development

During the implementation, I worked closely with one of our developers who was responsible for the application logic. This collaboration ensured that the microservice was properly integrated with the Pub/Sub event-driven architecture. It was a valuable experience in aligning infrastructure and development efforts for a seamless deployment.

## 📚 What I Learned

1. 🎛 **Fine-Tuning Scaling Parameters:** Finding the right balance for minReplicaCount, maxReplicaCount, and cooldownPeriod was crucial to prevent over-scaling or under-scaling.

2. 📡 **Pub/Sub Message Load Handling:** KEDA efficiently handles large message loads, but testing different SubscriptionSize values helped optimize performance.

3. 🕵️ **Monitoring and Troubleshooting:** Observability tools and logs played a key role in debugging scaling issues and ensuring the microservice behaved as expected.

## 📊 Load Testing

Once the microservice and KEDA configuration were deployed, I conducted a load test to validate:

- ✅ The ability of KEDA to scale up/down effectively based on Pub/Sub message load.

- ⚖️ The impact of different pollingInterval, minReplicaCount, and maxReplicaCount settings.

- 🚦 The responsiveness of the system under varying workloads.

## 🏁 Conclusion

By leveraging KEDA, I was able to dynamically scale the microservice based on both event-driven Pub/Sub messages and CPU utilization. This ensured optimal resource utilization and cost efficiency while maintaining performance and responsiveness.

This project gave me a deeper understanding of event-driven autoscaling in a Kubernetes environment and provided a scalable foundation for future workload expansions. The hands-on experience with KEDA and Terraform reinforced my ability to design and implement efficient cloud-native solutions. 🚀
