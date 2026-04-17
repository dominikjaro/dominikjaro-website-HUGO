---
title: "Beyond the kubectl Command: How I Passed the CKA 🎓"
date: 2026-04-17
description: "Using Kubernetes and passing the CKA exam are two completely different things. Here's what I learned after weeks of study and finally passing one of the hardest exams I've ever taken."
categories: ["Kubernetes"]
tags: ["Kubernetes", "CKA", "Certification", "DevOps", "Cloud"]
---

## ✨ Introduction

I've been working as a Cloud Engineer at Feefo for a while now, and Kubernetes is part of my daily life. But honestly? Using Kubernetes and passing the **Certified Kubernetes Administrator (CKA)** exam are two completely different things.

After weeks/months of study, I finally passed. Here is what I learned and why it was one of the hardest exams I've ever taken.

---

## 🔍 The "Holistic" Change

Before I started studying, I knew how to deploy apps and fix basic issues. But the CKA forced me to look at the cluster **holistically**.

I had to stop looking at Kubernetes as just a tool and start looking at it as a collection of moving parts. I did deep dives into:

- **The Internals** — How `etcd` actually stores data and how to restore it when it dies.
- **Security** — Writing RBAC rules and managing TLS certificates manually (using `openssl` in the terminal).
- **Networking** — Understanding exactly how a Pod talks to a Service and why DNS might fail.

It changed my approach from *"I think this works"* to *"I know exactly why this is happening."*

---

## ⏱️ The Hardest Part: The Pressure

The CKA isn't a multiple-choice exam. There is no "A, B, or C." It is just you, a terminal, and a list of broken clusters.

The biggest challenge wasn't just the technical stuff — it was the **time**. You have **2 hours** to finish **17–20 complex tasks**. You barely have time to think or look at the documentation. You need to have **"muscle memory."** You have to know the commands so well that your fingers type them before your brain even finishes the thought.

> If you get stuck on a question for more than 10 minutes, you are in trouble. You have to move fast, execute perfectly, and keep your cool under pressure.

---

## 📚 The Resources I Used

If you are planning to take the exam, these are the tools that actually helped me:

| Resource | Why It Helped |
|---|---|
| **[Killer.sh](https://killer.sh)** | The gold standard. Much harder than the real exam, but prepares you for the "panic" of the real thing. |
| **[Killercoda](https://killercoda.com)** | Great for practising specific scenarios for free in a browser terminal. |
| **Sailor.sh Mock Exams** | Really good for testing your speed and getting used to exam-style questions. |
| **Pluralsight** | Good for the theoretical foundations and deep dives. |
| **AWS (Personal Lab)** | I used my own AWS environment to build clusters from scratch using `kubeadm`. There is no better way to learn than breaking your own cluster and fixing it. |

---

## 💡 Final Thoughts

Passing the CKA isn't just about the badge. It's about the **confidence**. Now, when I'm working at Feefo and a node goes `NotReady` or a certificate expires, I don't feel lost. I know exactly where to look.

If you are thinking about doing it — **do it**. Just be ready to work hard!
