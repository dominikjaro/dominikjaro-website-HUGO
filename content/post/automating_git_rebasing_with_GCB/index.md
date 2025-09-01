---
title: "🚀 Automating Git Rebasing with Google Cloud Build"
date: 2025-09-01
description: "The goal was simple: for every open pull request, automatically rebase its feature branch onto `master` and force-push the result. "
categories: ["Cloud", "CI/CD"]
tags: ["automation"]
image: "cloudbuild.png"
---

As part of our CI/CD process, we need to keep our feature branches up-to-date with the master branch. This helps prevent merge conflicts and ensures our code is always based on the latest stable version. I recently built a Google Cloud Build pipeline to automate this tedious task.

This post will walk you through the key components of the Cloud Build YAML file that makes this happen.

## ⚙️ The Cloud Build Pipeline Explained

This pipeline consists of two main steps: configuring SSH access to GitHub and running the rebasing logic using `gh` CLI.

1. **🔐 SSH Key Configuration**
   To interact with GitHub in a non-interactive way, the pipeline needs an SSH key. We store this securely in Google Secret Manager. The first step retrieves this key and configures SSH for the subsequent Git commands.

```yaml
  # SSH key configuration
  - name: "gcr.io/cloud-builders/git"
    id:  "configure-ssh-key"
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
```

- `secretEnv`: This field is used to inject the secret `GITHUB_SSH_KEY` from Secret Manager into the build step's environment. The `$` is escaped with `$$` so that the value is not substituted during the build config parsing, but rather at runtime.
- `volumed`: We use a named volume to ensure the configured SSH key persists between build steps. This allows the rebasing step to use the same SSH configuration. 

2. **🔄 Rebasing All Open Pull Request Branches**
   This is the core of the automation. This step fetches all open pull requests using the `gh` CLI and then iterates through each one to perform the rebase.

```yaml
# Rebase all open PR branches onto master
  - name: "europe-west2-docker.pkg.dev/artifact_rep/docker/gh:latest"
    id: "rebase-all-branches"
    entrypoint: "bash"
    volumes:
      - name: "ssh"
        path: "/root/.ssh"
    args:
    - "-c"
    - |
      _OPEN_PR_BRANCHES=$(gh pr list --repo "${REPO_FULL_NAME}" --state open --json headRefName -q '.[].headRefName')
      
      git clone git@github.com:organization_repo/${REPO_NAME}.git
      cd ${REPO_NAME}
      
      git config --global user.email "[YOUR_GITHUB_EMAIL_ADDRESS]"
      git config --global user.name "[YOUR_GITHUB_USERNAME]"

      git remote update
      
      for feature_branch_name in $${_OPEN_PR_BRANCHES}; do
          echo "Processing $${feature_branch_name}"
          
          if [ -d ".git/rebase-merge" ]; then
            echo "Previous rebase detected, aborting..."
            git rebase --abort || rm -rf .git/rebase-merge
          fi
          
          git checkout $${feature_branch_name} 
          echo "Rebasing $${feature_branch_name} onto origin/master"
          git rebase origin/master
          echo "Pushing $${feature_branch_name} to origin"
          git push origin $${feature_branch_name} --force-with-lease
      done
    secretEnv:
      - "GITHUB_TOKEN"
```

- `name`: We use a custom-build Docker image that includes the `gh` CLI. This allows us to interact with the GitHub API to list open PRs.
- `_OPEN_PR_BRANCHES`: The `gh pr list` command is used to get a list of all open pull requests and extracts their head branch names using a `jq` query.
- **Rebasing Loop**: The `for` loop iterates through each of the open branches. For each one:
  1. It checks for an incomplete rebase from a previous run and aborts it.
  2. It checks out the feature branch.
  3. It performs a `git rebase origin/master`.
  4. It performs a `git push --force-with-lease`, which is a safer version of a force push.

This pipeline automates a key maintenance task, saving developers time and keeping our repository clean and organized.