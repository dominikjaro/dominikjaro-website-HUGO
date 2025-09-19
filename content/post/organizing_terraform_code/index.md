---
title: "🗂️ Organising Terraform Code – module structure"
date: 2025-06-01
description: "In this post, I’ll document how I approached organizing Terraform code for managing Auth0 tenants independently of the broader platform infrastructure."
categories: ["Infrastructure as Code"]
tags: ["Terraform"]
image: "terraform-code.png"
---

## 📁 Structure Overview

When working with infrastructure as code, choosing the right Terraform module structure can be crucial for maintainability, scalability, and operational safety. In this post, I’ll document how I approached organizing Terraform code for managing Auth0 tenants independently of the broader platform infrastructure.
Here’s the final structure I implemented under our Terraform monorepo:

```
📁 org/
├── auth0/
│   ├── auth0-dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfstate
│   └── auth0-prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfstate

📦 modules/
└── auth0/
    ├── auth0-dev/
    │   ├── *.tf (config files)
    └── auth0-prod/
        ├── *.tf (config files)
```

⸻

## 🎯 The Problem

Previously, our auth0 Terraform configuration lived within the same platform folder as our other microservices. This posed a few issues:

- 🔄 Shared root modules meant shared state, leading to potential clashes between dev and prod environments.
- 🚫 Any change or issue in auth0 could block platform deployments.
- 💥 Less isolation increased the blast radius for errors or upgrades.

⸻

## ✅ The Solution

To reduce risk and improve isolation, I extracted auth0 from the shared platform root module and moved it under its own folder within the org/ structure.

Instead of using a single root module with multiple tfvars for environments (as we do for platform), I opted to create two completely separate root modules: one for auth0-dev and one for auth0-prod.

Why?

- 🔄 Auth0 has a distinct lifecycle compared to platform services.
- 🛡️ Decoupling improves safety — I can upgrade or test the dev tenant without touching production.
- 📂 Separate state files make it easier to manage lifecycle events (e.g., state migrations or imports).

⸻

## 💡 Key Takeaways

- 📝 Terraform code should be easy to read, even if it’s not perfectly DRY. Stability > cleverness.
- 🛡️ Organizing infrastructure into isolated root modules with clear boundaries can prevent cross-env interference.
- ⚙️ This structure allows more granular CI/CD workflows — you can plan/apply per service or environment.

⸻

## 📚 References

- [Reddit: Organizing Terraform code](https://www.reddit.com/r/Terraform/comments/1i1u9yn/organizing_terraform_code/)
- [Terrateam Blog: Terraform Code Organization](https://terrateam.io/blog/terraform-code-organization/)

⸻

🚀 This setup has been working well for us so far. It gives clarity, safer deployment boundaries, and the flexibility to scale or refactor as needed.