---
title: "🚀 Automating Terraform Docs with Cloud Build"
date: 2025-06-16
description: "How I automated terraform-docs for our platform infrastructure repo using GCP Cloud Build and Docker"
categories: ["Cloud", "IaC","GCP"]
tags: ["terraform","automation","cloud-build","docker"]
image: "terraform-docs.png"
---

## ✨ Introduction 

I recently automated the generation of Terraform module documentation using [`terraform-docs`](https://terraform-docs.io) and GCP Cloud Build. The goal? Stop manually running `terraform-docs` and let CI handle the boring stuff — especially for our **platform-infra** repo where we use modules heavily.

Now, whenever someone pushes a commit (excluding `master`), our Cloud Build pipeline kicks in, regenerates any changed module docs, and commits them back to the same branch.

## 🤔 Why terraform-docs?

[`terraform-docs`](https://terraform-docs.io) is a CLI tool that generates clean, readable documentation for Terraform modules. It scans your module and creates a Markdown table listing inputs, outputs, and descriptions. Super handy for keeping module usage understandable.

We use it to maintain a `README.md` for each module, but running it manually was becoming tedious and easy to forget.

## ⚙️ Prerequisites

Before starting, here's what you'll need:

- GCP project with:
  - Cloud Build
  - Artifact Registry
  - Secret Manager
- SSH key for GitHub automation (stored in Secret Manager)
- `terraform-docs` v0.20.0
- Basic familiarity with Docker and CI/CD

---

## 🧠 Step 1: Plan the Architecture

Here’s what I mapped out before jumping in:

- Trigger the pipeline on **any branch push except `master`**
- Only target our **platform-infra** repo
- Use a **custom Docker image** with `terraform-docs` preinstalled
- Loop through each module under `terraform/modules/`
- If it contains a `variables.tf` file → run `terraform-docs` and generate `README.md`
- Auto-commit back to the same branch **only if changes are detected**

---

## 🐳 Step 2: Create a Docker Image

I built a multi-stage Dockerfile that installs `terraform-docs` in a lightweight Alpine image:

```Dockerfile
FROM golang:alpine AS build
WORKDIR /app
RUN apk add --no-cache git make gcc libc-dev
RUN go install github.com/terraform-docs/terraform-docs@v0.20.0

FROM alpine:latest
COPY --from=build /go/bin/terraform-docs /usr/local/bin/terraform-docs
RUN apk update && \
    apk add --no-cache git bash openssh
ENTRYPOINT ["/usr/local/bin/terraform-docs"]
```

Then I pushed it to Artifact Registry using a simple Makefile.

## 🧪 Step 3: Test Script Locally

Wrote and tested a local script that:

- Loops through subdirectories with a variables.tf
- Runs terraform-docs
- Stages and commits only if there’s a diff

## 🔐 Step 4: Handle GitHub SSH and Run the Full Pipeline in Cloud Build

The full automation is handled in a single cloudbuild.yaml file. It does the following:

1. Configures SSH using a GitHub SSH key from Secret Manager 🗝️
2. Pulls the target branch from GitHub
3. Loops through module folders to run terraform-docs
4. Commits & pushes only if there are changes 📤

```yaml
timeout: "28800s"
options:
  pool:
    name: "projects/${PROJECT_ID}/locations/LOCATION/workerPools/${_WORKER_POOL_NAME}"
  substitution_option: "ALLOW_LOOSE"

substitutions:
  _WORKER_POOL_NAME: "WORKER_POOL_NAME" # Replace with your worker pool name

steps:
  # 🛠️ Step 1: Configure SSH
  - name: "gcr.io/cloud-builders/git"
    id: "configure-ssh-key"
    waitFor: ['-']
    entrypoint: "/bin/bash"
    args:
      - "-c"
      - |
        echo "$$GITHUB_SSH_KEY" >> /root/.ssh/id_rsa
        chmod 400 /root/.ssh/id_rsa
        cat <<EOF >/root/.ssh/config
        Hostname github.com
        IdentityFile /root/.ssh/id_rsa
        StrictHostKeyChecking no
        UserKnownHostsFile /dev/null
        LogLevel ERROR
        EOF
    secretEnv:
      - "GITHUB_SSH_KEY"
    volumes:
      - name: "ssh"
        path: "/root/.ssh"

  # 📦 Step 2: Run terraform-docs inside a Docker container
  - name: "PATH_TO_YOUR_DOCKER_IMAGE" # Replace with your Docker image path
    id: "generate-terraform-docs"
    entrypoint: "bash"
    volumes:
      - name: "ssh"
        path: "/root/.ssh"
    args:
      - '-c'
      - |
        #!/bin/bash

        git clone -b ${BRANCH_NAME} --single-branch --depth 1 git@github.com:[REPLACE_WITH_YOUR_GITHUB_OWNER]/${REPO_NAME}.git

        git config --global user.email "REPLACE_WITH_YOUR_EMAIL"
        git config --global user.name "REPLACE_WITH_YOUR_NAME"

        cd ${REPO_NAME}

        module_dir="REPLACE_WITH_MODULE_DIR" # e.g., "terraform/modules"

        find "$module_dir" -type f -name "variables.tf" ! -path "./.git/*" | \
        xargs -n 1 dirname | \
        sort -u | \
        while read -r module_path; do
          if [ -d "$module_path" ]; then
            (cd "$module_path" && terraform-docs markdown table . --output-file README.md)
          fi
        done

        ls -ltra

        git add -A

        if git diff-index --quiet HEAD; then
          echo "No documentation changes detected, no commit needed."
        else
          git commit -m "docs: Auto-generate Terraform module READMEs [skip ci]"
          echo "Documentation changes committed. Pushing to branch: ${BRANCH_NAME}"
          git push -u origin ${BRANCH_NAME}
        fi

availableSecrets:
  secretManager:
    - versionName: "REPLACE_WITH_YOUR_SECRET_VERSION_NAME" # e.g., "projects/PROJECT_ID/secrets/GITHUB_SSH_KEY/versions/latest"
      env: "GITHUB_SSH_KEY"
```

## ✅ Step 5: Finalise & Terraform It

Once I confirmed everything works via manual triggers, I:

- Created the Cloud Build trigger in Terraform
- Applied it
- Merged the initial batch of generated README files

```trigger.tf
resource "google_cloudbuild_trigger" "terraform_docs_generator" {
  description = "Automatically generates Terraform module READMEs on non-main branch pushes."
  disabled    = false
  filename    = "REPLACE_WITH_YOUR_CLOUDBUILD_FILE" # e.g., ".cloudbuild/terraform-docs.yaml"
  location    = var.location
  name        = "terraform-docs-generator"
  project     = var.project


  substitutions = {
    "_WORKER_POOL_NAME" = "REPLACE_WITH_YOUR_WORKER_POOL_NAME" 
  }

  tags = [
    "managed_by_terraform",
    "terraform-docs-automation",
  ]

  approval_config {
    approval_required = false
  }

  github {
    name  = "REPLACE_WITH_YOUR_GITHUB_REPO_NAME" # e.g., "platform-infra"
    owner = "REPLACE_WITH_YOUR_GITHUB_OWNER"

    push {
      # This regex matches any branch name that is NOT equal to your main branch (e.g. 'master')
      branch       = "master"
      invert_regex = true
    }
  }

  timeouts {}
}
```

Now it works like a charm. 🚀 On every push (except master), the pipeline runs and updates only what’s needed.

---

## 🧾 Result

- 📄 Consistent and up-to-date Terraform module documentation
- ⚙️ Fully automated via Cloud Build and Docker
- 🔁 Runs only when needed — no noisy commits
- 🧘 I don’t have to think about terraform-docs anymore!
  
## 🗣️ Final Thought

It doesn’t have to stay at the same pace as when we started 😉 — but it does have to be maintainable. This was a fun task that scratched my automation itch while saving future-me a lot of time.
