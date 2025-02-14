---
title: "🔐 Ditching JSON Keys for Workload Identity Federation (WID) in Kubernetes"
date: 2025-02-14
description: "When running workloads in Kubernetes that need to access Google Cloud resources, a common approach has been to use a service account JSON key stored in a secret. However, this method has security vulnerabilities. Recently, I transitioned to using **Workload Identity Federation (WID)**, which eliminates the need for JSON keys while ensuring secure access.Here’s why WID is a game-changer and what I learned during this migration. 🚀"
categories: ["Cloud","Security","Kubernetes"]
tags: ["GCP","workload-identity-federation","json"]
image: "wid.jpg"
---

## ❌ The Problem with JSON Keys

Using service account JSON keys in Kubernetes involves storing them as **secrets** and mounting them into workloads. While this works, it introduces serious security risks:

1. **🔓 Exposure Risk** – If a JSON key is compromised, an attacker gains access to cloud resources, potentially leading to data breaches.
2. **🚫 Hard to Rotate** – JSON keys don’t expire unless manually rotated, making it difficult to enforce security best practices.
3. **⚠️ Secret Management Overhead** – Storing and managing secrets in Kubernetes requires additional tooling and maintenance (e.g., External Secrets, Vault, etc.).
4. **🎭 Identity Confusion** – Kubernetes workloads are unaware of the identity behind the JSON key, making it harder to track which workloads are using which credentials.

## ✅ Why Workload Identity Federation (WID) is Better

With **Workload Identity Federation**, we can securely authenticate Kubernetes workloads without storing long-lived credentials. Instead of using a JSON key, we:

1. **👤 Use a Google Cloud IAM Service Account** with the `Workload Identity User` role.
2. **🤖 Link it to a Kubernetes Service Account (KSA)** that the workload runs under.
3. **🔄 Enable automatic token exchange** – The workload uses its KSA identity to impersonate the IAM Service Account dynamically.

### 🎯 Key Benefits of WID

- **🔥 Eliminates JSON keys**, reducing credential exposure risks.
- **🔑 Short-lived, auto-rotating credentials**, improving security.
- **🛠️ Seamless authentication**, no need for secret management.
- **🔍 Better observability**, as workloads assume their own identities.

## 📌 The Migration Process

I made several changes to adopt WID:

1. **Updated Helm charts** to remove references to JSON keys.
2. **Created a Kubernetes Service Account (KSA)** and linked it to the Google Cloud IAM Service Account.
3. **Removed External Secrets and environment variables** that previously referenced the JSON key.
4. **Tested the new setup** – Everything worked perfectly! ✅
5. **Deleted the old JSON key** and its associated secrets to ensure no lingering credentials.

## 🎓 Lessons Learned

🔹 **Security first!** WID significantly improves security posture by removing static credentials.

🔹 **Simplicity matters.** No more JSON key management means less overhead and fewer things that can go wrong.

🔹 **Observability improves.** Since each workload assumes a unique identity, tracking access patterns is much easier.

🔹 **Testing is key.** Before deleting JSON keys, thorough testing ensures a smooth transition.

## 🚀 Final Thoughts

Moving away from JSON keys to **Workload Identity Federation** was a great step towards improving security and simplifying authentication for Kubernetes workloads. If you’re still using JSON keys in your clusters, I highly recommend making the switch! 🔄🔐
