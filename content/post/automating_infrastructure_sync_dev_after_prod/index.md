---
title: "🚀 Automating Infrastructure with Cloud Build & Atlantis -- sync_dev_after_prod"
date: 2025-09-04
description: "To automate a key workflow—syncing our master branch configuration to our dev-* environments—I created a robust Google Cloud Build pipeline."
categories: ["Cloud", "CI/CD"]
tags: ["automation", "terraform"]
image: "sync_after_prod.png"
---

## 🚀 Automating Infrastructure with Cloud Build & Atlantis

Managing infrastructure-as-code is a critical part of modern DevOps. Our team uses Terraform to define our infrastructure and Atlantis to orchestrate plans and applies through pull requests. To automate a key workflow—syncing our `master` branch configuration to our `dev-*` environments—I created a robust Google Cloud Build pipeline.

This pipeline is triggered by every push or merge to our `master` branch. It performs a series of checks, triggers a Terraform plan and apply, and sends a notification to our team's channel. Here's a breakdown of how it works.

## Phase 1: ⚙️ Setup and Synchronization

This first phase is all about preparing the environment and creating the specific branch that will trigger our workflow.

### Step 1: 🔐 SSH Key Configuration

Our pipeline needs to talk to GitHub to clone repositories and manage pull requests. We store an SSH key securely in Google's Secret Manager. This step retrieves that key and configures a secure SSH connection for all subsequent Git commands.

```yaml
# 1. Configure SSH key for authenticating with GitHub.
  - name: "gcr.io/cloud-builders/git"
    id:  "configure-ssh-key"
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

### Step 2: Concurrency Control

This is a critical, yet often overlooked, step. To prevent multiple builds from trying to apply infrastructure changes at the same time, this step checks for any other builds from the same trigger that are either running or queued. If it finds any, it simply waits for them to finish, ensuring only one sync workflow is running at any given time.

```yaml
# 2. It checks for other running or queued builds from the same trigger and waits for up to 1 hour.
  - name: "gcr.io/google.com/cloudsdktool/google-cloud-cli:slim"
    id: "wait-for-other-builds"
    waitFor: ['configure-ssh-key']
    entrypoint: "bash"
    args:
      - "-c"
      - |
        set -e
        TIMEOUT=3600
        INTERVAL=30
        ELAPSED=0

        echo "Checking for other running builds with tag: $$TRIGGER_TAG"

        while [ $$ELAPSED -lt $$TIMEOUT ]; do
          # List other builds for the same trigger that are currently running or queued.
          OTHER_BUILDS=$(gcloud builds list --region="${LOCATION}" --project="${PROJECT_ID}" --filter="tags='${_TRIGGER_TAG}' AND (status='WORKING' OR status='QUEUED') AND id!='${BUILD_ID}'" --format="value(id)")

          if [ -z "$$OTHER_BUILDS" ]; then
            echo "No other concurrent builds found. Proceeding."
            exit 0
          fi

          echo "Found other running or queued builds: $$OTHER_BUILDS"
          echo "This build will wait. Checking again in $${INTERVAL}s..."
          sleep $$INTERVAL
          ELAPSED=$$((ELAPSED + INTERVAL))
        done

        echo "Error: Timed out after 1 hour. Other builds are still running: $$OTHER_BUILDS"
        exit 1
```

### Step 3: Create the Sync Branch

The `master` branch is the source of truth for our infrastructure. This step creates a new branch named `sync-dev-after-prod`. The pipeline will then use this branch as the head for a new pull request. A crucial piece of logic here is to fail the pipeline if this branch already exists.

```yaml
# 3. Create a sync branch from the target branch
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "create-or-update-sync-branch"
    waitFor: ['wait-for-other-builds']
    entrypoint: "bash"
    volumes:
      - name: "ssh"
        path: "/root/.ssh"
    args:
      - '-c'
      - |
        set -e
        git clone git@github.com:${REPO_FULL_NAME}.git
        cd ${REPO_NAME}

        # Check if the sync branch already exists on the remote. If it does, cancel the pipeline.
        if git ls-remote --exit-code --heads origin "${_SYNC_BRANCH}"; then
          echo "Error: Sync branch '${_SYNC_BRANCH}' already exists on the remote."
          exit 1
        fi

        git config user.email "${_GIT_USER_EMAIL}"
        git config user.name "${_GIT_USER_NAME}"

        git fetch origin
        git checkout -B "${_SYNC_BRANCH}" "origin/${BRANCH_NAME}"

        # Create a commit to ensure the branch is updated and can be used for a new PR.
        date > sync-timestamp.txt
        git add sync-timestamp.txt
        git commit -m "chore: Trigger Atlantis sync and add bot as temporary codeowner [skip ci]"

        git push origin "${_SYNC_BRANCH}" --force
    secretEnv:
     - "GITHUB_TOKEN"
```

---

## Phase 2:🗺️ The Atlantis Plan Workflow

Now that our environment is ready and our branch is in place, we can trigger Atlantis to perform a `terraform plan`.

### Step 4: Create a Pull Request

This step uses the `gh` CLI to create a new pull request from our `sync-dev-after-prod` branch to master. This pull request is the core trigger for our Atlantis workflow. When Atlantis sees a new PR that matches its configuration, it automatically runs a `terraform plan` for all the affected projects. This step also ensures we don't create a new PR if one for this branch already exists.

```yaml
# 4. Create a pull request to trigger the Atlantis plan.
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "create-sync-pr"
    waitFor: ['create-or-update-sync-branch']
    entrypoint: "bash"
    volumes:
      - name: "ssh"
        path: "/root/.ssh"
    args:
      - '-c'
      - |
        set -e
        EXISTING_PR=$(gh pr list --repo "${REPO_FULL_NAME}" --head "${_SYNC_BRANCH}" --json number -q '.[0].number')

        if [ -n "$$EXISTING_PR" ]; then
          echo "Pull request #$${EXISTING_PR} for branch '${_SYNC_BRANCH}' already exists."
        else

          gh pr create \
            --repo "${REPO_FULL_NAME}" \
            --base "${BRANCH_NAME}" \
            --head "${_SYNC_BRANCH}" \
            --title "chore: Sync TF infrastructure from ${BRANCH_NAME} to dev environments" \
            --body "This is an automated PR to trigger an \`atlantis plan\` for all \`dev-*\` projects. It compares the current \`${BRANCH_NAME}\` branch configuration against the last applied state for the dev environments."
        fi
    secretEnv:
     - "GITHUB_TOKEN"
```

### Step 5: Trigger & Wait for the Atlantis Plan

This is one of the most complex steps. It does two things: **triggers the plan** by posting a comment on the new PR, and then **waits for completion** by periodically checking the PR for a final summary comment from the Atlantis bot.

```yaml
# 5. Trigger 'atlantis plan' and wait for all projects to complete.
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "trigger-atlantis-plan-workflow"
    waitFor: ['create-sync-pr']
    entrypoint: "bash"
    args:
      - '-c'
      - |
        set -e
        
        echo "Looking for PR with head branch: ${_SYNC_BRANCH}"
        _PR_NUMBER=$(gh pr list --repo "${REPO_FULL_NAME}" --head "${_SYNC_BRANCH}" --json number -q '.[0].number')

        if [ -z "$$_PR_NUMBER" ]; then
          echo "No open PR found for branch ${_SYNC_BRANCH}. Exiting gracefully."
          exit 0
        fi

        echo "Found PR number: $${_PR_NUMBER}"
        echo "$${_PR_NUMBER}" > /workspace/pr_number.txt

        echo "Triggering 'atlantis plan -p dev-.*' on PR #$${_PR_NUMBER}"
        gh pr comment "$${_PR_NUMBER}" --repo "${REPO_FULL_NAME}" --body "atlantis plan -p dev-.*"

        echo "Waiting for Atlantis plan to complete..."
        TIMEOUT=1800; INTERVAL=30; ELAPSED=0

        while [ $$ELAPSED -lt $$TIMEOUT ]; do
          # Wait for the final summary comment from Atlantis, which indicates all plans are complete.
          ATLANTIS_SUMMARY_COMMENT=$(gh pr view "$${_PR_NUMBER}" --repo "${REPO_FULL_NAME}" --json comments -q '.comments[] | select(.author.login == "[YOUR_USERNAME]" and (.body | test("Ran Plan for .* projects"))).body')
          if [ -n "$$ATLANTIS_SUMMARY_COMMENT" ]; then
            echo "Atlantis plan summary comment found. Proceeding."
            break
          fi
          sleep $$INTERVAL
          ELAPSED=$$((ELAPSED + INTERVAL))
          echo "Waited $${ELAPSED}s..."
        done

        if [ $$ELAPSED -ge $$TIMEOUT ]; then
          echo "Timed out waiting for Atlantis plan to complete."
          exit 1
        fi

        # Allow a moment for all comments to be posted before gathering statuses.
        sleep 60

        echo "Gathering Atlantis plan statuses..."
        ALL_COMMENTS=$(gh pr view "$${_PR_NUMBER}" --repo "${REPO_FULL_NAME}" --json comments --jq '.comments[].body')
        DEV_PROJECTS="dev-0 dev-1 dev-2 dev-3 dev-4 dev-5 dev-6 dev-7 dev-8 dev-90 dev-rc"

        PLAN_STATUS_JSON="["
        FIRST_ITEM=true

        for PROJECT in $$DEV_PROJECTS; do
          # Extract the comment block for each project to check its status accurately.
          PROJECT_OUTPUT=$(echo "$$ALL_COMMENTS" | awk -v proj="$$PROJECT" '/^### [0-9]+\. project: / && $0 ~ proj {p=1} /^### [0-9]+\. project: / && !($0 ~ proj) {p=0} p {print}')

          STATUS="success"
          if echo "$$PROJECT_OUTPUT" | grep -q "This project is currently locked"; then
            STATUS="locked"
          elif echo "$$PROJECT_OUTPUT" | grep -q "Plan Failed" || [ -z "$$PROJECT_OUTPUT" ]; then
            STATUS="failed"
          else
            STATUS="success"
          fi

          if [ "$$FIRST_ITEM" = false ]; then
            PLAN_STATUS_JSON="$${PLAN_STATUS_JSON},"
          fi
          PLAN_STATUS_JSON="$${PLAN_STATUS_JSON}{\"project\":\"$$PROJECT\",\"status\":\"$$STATUS\"}"
          FIRST_ITEM=false
        done
        PLAN_STATUS_JSON="$${PLAN_STATUS_JSON}]"

        echo "Final status JSON: $$PLAN_STATUS_JSON"
        echo "$$PLAN_STATUS_JSON" > /workspace/atlantis_plan_status.json
    secretEnv:
      - "GITHUB_TOKEN"
```

---

## Phase 3: ✍️ The Atlantis Apply Workflow

With the plans successfully completed, we can now proceed to apply them to our environments.

### Step 6: Update the PR Branch

To ensure our PR is still mergeable before applying, this step performs a quick `git rebase` of the sync branch onto the latest `master`. This prevents `atlantis apply` from failing due to a stale branch.

```yaml
# 6. Update the PR branch to ensure it's mergeable before applying.
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "update-pr-branch"
    waitFor: ['trigger-atlantis-plan-workflow']
    entrypoint: "bash"
    volumes:
      - name: "ssh"
        path: "/root/.ssh"
    args:
      - '-c'
      - |
        # This step prevents the PR from becoming unmergeable if the target branch advances
        # while Atlantis is running, which would block 'atlantis apply'.
        set -e
        cd ${REPO_NAME}

        git config user.email "${_GIT_USER_EMAIL}"
        git config user.name "${_GIT_USER_NAME}"

        echo "Rebasing '${_SYNC_BRANCH}' onto 'origin/${BRANCH_NAME}'..."
        git fetch origin
        git checkout "${_SYNC_BRANCH}"
        git rebase "origin/${BRANCH_NAME}"

        # Create a new commit to ensure the branch HEAD is updated.
        date > sync-timestamp.txt
        git add sync-timestamp.txt
        git commit -m "chore: Refresh branch to ensure mergeability [skip ci]"

        git push origin "${_SYNC_BRANCH}" --force
        sleep 40
```

### Step 7: Auto-Approve the Pull Request

Our repository has branch protection rules that require at least one approval before a PR can be merged. This step uses a special token to auto-approve the PR, which allows Atlantis to later merge the changes without any manual intervention.

```yaml
# 7. Auto-approve the PR to satisfy branch protection rules.
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "auto-approve-pr"
    waitFor: ['update-pr-branch']
    entrypoint: "bash"
    args:
      - "-c"
      - |
        set -e
        if [ ! -f /workspace/pr_number.txt ]; then
          echo "PR number not found, skipping approval."
          exit 0
        fi
        _PR_NUMBER=$(cat /workspace/pr_number.txt)

        echo "Approving PR #$${_PR_NUMBER}"
        GH_TOKEN="$$AUTO_APPROVE_TOKEN" gh pr review "$${_PR_NUMBER}" --repo "${REPO_FULL_NAME}" --approve --body "Automated approval for sync workflow."
        echo "PR #$${_PR_NUMBER} approved."
        sleep 60
    secretEnv:
      - "AUTO_APPROVE_TOKEN"
```

### Step 8: Trigger & Wait for the Atlantis Apply

This step uses the JSON file we generated earlier to trigger a selective apply. It iterates through the projects that had a successful plan and posts a comment `atlantis apply -p [project]` for each one. Similar to the plan step, it then waits for each apply to complete by polling for a specific `atlantis/apply` check run on the PR. The final statuses are logged to another JSON file.

```yaml
# 8. Trigger 'atlantis apply' for all projects with successful plans.
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "trigger-atlantis-apply-workflow"
    waitFor: ['auto-approve-pr']
    entrypoint: "bash"
    args:
      - "-c"
      - |
        set -e
        if [ ! -f /workspace/atlantis_plan_status.json ]; then
           echo "Atlantis plan status not found, skipping apply."
           exit 0
        fi
        if [ ! -f /workspace/pr_number.txt ]; then
          echo "PR number not found, skipping apply."
          exit 0
        fi

        _PR_NUMBER=$(cat /workspace/pr_number.txt)
        PLAN_STATUS_JSON=$(cat /workspace/atlantis_plan_status.json)

        # Extract only projects with a successful plan.
        SUCCESS_PROJECTS=$(echo "$$PLAN_STATUS_JSON" | jq -r '.[] | select(.status=="success") | .project')

        if [ -z "$$SUCCESS_PROJECTS" ]; then
          echo "No successful plans to apply. Skipping."
          echo "[]" > /workspace/atlantis_apply_status.json
          exit 0 
        fi

        APPLY_STATUS_JSON="["
        FIRST_ITEM=true

        for PROJECT in $$SUCCESS_PROJECTS; do
          APPLY_STATUS=""
          echo "---"
          echo "Triggering 'atlantis apply' for project: $${PROJECT} on PR #$${_PR_NUMBER}"
          gh pr comment "$${_PR_NUMBER}" --repo "${REPO_FULL_NAME}" --body "atlantis apply -p $${PROJECT}"
 
          # Add a delay to allow GitHub and Atlantis time to create the check run.
          echo "Waiting 40s for the apply check to be initiated for $${PROJECT}..."
          sleep 40
 
          echo "Polling for apply check completion for $${PROJECT}..."
          TIMEOUT=1800; INTERVAL=30; ELAPSED=0
          APPLY_FINISHED=false

          while [ $$ELAPSED -lt $$TIMEOUT ]; do
            CHECK_NAME="atlantis/apply: $${PROJECT}"
            
            CHECK_STATE=$(gh pr checks "$${_PR_NUMBER}" --repo "${REPO_FULL_NAME}" --json name,state -q ".[] | select(.name == \"$${CHECK_NAME}\") | .state")

            if [ "$$CHECK_STATE" == "SUCCESS" ]; then
              echo "GitHub check '$${CHECK_NAME}' reported SUCCESS."
              APPLY_STATUS="success"
              APPLY_FINISHED=true
              break
            elif [[ "$$CHECK_STATE" == "FAILURE" || "$$CHECK_STATE" == "ERROR" ]]; then
              echo "ERROR: GitHub check '$${CHECK_NAME}' reported state: $${CHECK_STATE}."
              APPLY_STATUS="failed"
              APPLY_FINISHED=true
              break
            fi

            sleep $$INTERVAL
            ELAPSED=$$((ELAPSED + INTERVAL))
            echo "Waited $${ELAPSED}s for check '$${CHECK_NAME}' to complete..."
          done

          if [ "$$APPLY_FINISHED" = false ]; then
            echo "Timed out waiting for GitHub check '$${CHECK_NAME}' to complete."
            APPLY_STATUS="failed"
          fi

          # Append to JSON
          [ "$$FIRST_ITEM" = false ] && APPLY_STATUS_JSON="$${APPLY_STATUS_JSON},"
          APPLY_STATUS_JSON="$${APPLY_STATUS_JSON}{\"project\":\"$${PROJECT}\",\"status\":\"$${APPLY_STATUS}\"}"
          FIRST_ITEM=false
        done

        # Finalize the JSON string and write to file
        APPLY_STATUS_JSON="$${APPLY_STATUS_JSON}]"
        echo "---"
        echo "Final apply status JSON: $$APPLY_STATUS_JSON"
        echo "$$APPLY_STATUS_JSON" > /workspace/atlantis_apply_status.json

        if echo "$$APPLY_STATUS_JSON" | grep -q '"status":"failed"'; then
          echo "One or more applies failed."
        else
          echo "All applies completed successfully."
        fi
    secretEnv:
      - "GITHUB_TOKEN"
```

---

## Phase 4: 💬 Notification & Cleanup

With the entire workflow complete, we'll notify the team and clean up the environment.

### Step 9: Prepare Notification Input

This step uses the `jq` utility to combine the plan and apply status JSON files into a single, comprehensive JSON payload. This payload will be used as input for our Python script in the next step.

```yaml
# 9. Prepare the input for the Teams notification template.
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "prepare-notification-input"
    waitFor: ['trigger-atlantis-apply-workflow']
    entrypoint: "bash"
    args:
      - "-c"
      - |
        set -e
        if [ ! -f /workspace/pr_number.txt ]; then
          echo "PR number not found, skipping notification preparation."
          echo '{"skip": true}' > /workspace/notification_input.json
          exit 0
        fi

        APPLY_STATUS_FILE="/workspace/atlantis_apply_status.json"
        PLAN_STATUS_FILE="/workspace/atlantis_plan_status.json"
        _PR_NUMBER=$(cat /workspace/pr_number.txt)
        _PR_URL="https://github.com/${REPO_FULL_NAME}/pull/$${_PR_NUMBER}"

        APPLY_STATUS_CONTENT=$$( [ -f "$$APPLY_STATUS_FILE" ] && cat "$$APPLY_STATUS_FILE" || echo "[]" )
        PLAN_STATUS_CONTENT=$$( [ -f "$$PLAN_STATUS_FILE" ] && cat "$$PLAN_STATUS_FILE" || echo "[]" )

        jq -n \
          --arg pr_number "$$_PR_NUMBER" \
          --arg pr_url "$$_PR_URL" \
          --argjson apply_status "$$APPLY_STATUS_CONTENT" \
          --argjson plan_status "$$PLAN_STATUS_CONTENT" \
          '{
            "pr_number": $pr_number,
            "pr_url": $pr_url,
            "apply_status": $apply_status,
            "plan_status": ($plan_status | map(select(.status == "locked")))
          }' > /workspace/notification_input.json

        APPLY_COUNT=$(jq '.apply_status | length' /workspace/notification_input.json)
        PLAN_COUNT=$(jq '.plan_status | length' /workspace/notification_input.json)
        if [ "$$APPLY_COUNT" -eq 0 ] && [ "$$PLAN_COUNT" -eq 0 ]; then
            echo "No content for notification. Skipping."
            echo '{"skip": true}' > /workspace/notification_input.json
        fi

        echo "Notification input JSON generated:"
        cat /workspace/notification_input.json
```

### Step 10: Generate the Teams Adaptive Card

This step uses a custom Python script, `main.py`, which is a key part of our pipeline. The script reads the JSON input and creates an Adaptive Card with a formatted summary of the plan and apply results. It saves this card as a JSON file in the build's output directory.

```python
#!/usr/bin/env python3

import json
import argparse
import os

def create_adaptive_card(data):
    """Builds the Python dictionary for the Adaptive Card."""
    pr_number = data.get("pr_number", "N/A")
    pr_url = data.get("pr_url", "#")
    apply_status = data.get("apply_status", [])
    plan_status = data.get("plan_status", [])  # These are the locked projects

    card_body = []

    # Main Title
    card_body.append({
        "type": "TextBlock",
        "text": f"Atlantis Sync Results for [PR #{pr_number}]({pr_url})",
        "wrap": True,
        "weight": "Bolder",
        "size": "Medium"
    })

    # Apply Status Section
    if apply_status:
        card_body.append({
            "type": "TextBlock",
            "text": "Atlantis Apply Status",
            "weight": "Bolder",
            "spacing": "Medium"
        })
        facts = []
        for item in apply_status:
            project = item.get("project", "N/A")
            status = item.get("status", "unknown")
            value = "✅ Success" if status == "success" else "❌ Failed"
            facts.append({"title": project, "value": value})
        
        card_body.append({"type": "FactSet", "facts": facts})

    # Locked Projects Section
    if plan_status:
        card_body.append({
            "type": "TextBlock",
            "text": "Locked Projects (Apply Skipped)",
            "weight": "Bolder",
            "spacing": "Medium"
        })
        facts = []
        for item in plan_status:
            project = item.get("project", "N/A")
            facts.append({"title": project, "value": "🔒 Locked"})

        card_body.append({"type": "FactSet", "facts": facts})

    # Final Card Structure
    adaptive_card = {
        "type": "AdaptiveCard",
        "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
        "version": "1.4",
        "body": card_body
    }
    return adaptive_card

def main():
    parser = argparse.ArgumentParser(description='Generate Teams Adaptive Card for Atlantis sync results.')
    parser.add_argument('--input', required=True, help='Path to the input JSON file (notification_input.json).')
    parser.add_argument('--output_dir', required=True, help='Directory to save the output card JSON file.')
    args = parser.parse_args()

    with open(args.input, 'r') as f:
        input_data = json.load(f)

    # Check for skip condition from the previous step
    if input_data.get("skip"):
        print("Skip condition found in input file. No card will be generated.")
        return

    adaptive_card = create_adaptive_card(input_data)

    os.makedirs(args.output_dir, exist_ok=True)

    output_path = os.path.join(args.output_dir, 'card.json')
    with open(output_path, 'w') as f:
        json.dump(adaptive_card, f, indent=2)

    print(f"Adaptive Card successfully generated at {output_path}")

if __name__ == "__main__":
    main()
```

```yaml
# 10. Generate Teams adaptive card using the Python script from the cloned repo.
  - name: "PATH_TO_YOUR_DOCKER_IMAGE]/sync-dev-after-prod:latest"
    id: "generate-teams-card"
    waitFor: ['prepare-notification-input']
    entrypoint: "bash"
    args:
      - "-c"
      - |
        set -e
        sync-dev-after-prod --input /workspace/notification_input.json --output_dir output
```

### Step 11: Send the Notification

A custom Docker image containing a notification tool reads the generated card JSON and sends it to our Teams channel via a webhook.

```yaml
# 11. Send the formatted notification to the Teams channel.
  - name: "PATH_TO_YOUR_DOCKER_IMAGE]/teams-notifications:latest"
    id: "send-teams-notification"
    waitFor: ['generate-teams-card']
    entrypoint: "bash"
    args:
      - "-c"
      - |
        set -e
        # Check if the output directory was created and is not empty.
        if [ ! -d "output" ] || [ -z "$(ls -A output)" ]; then
          echo "No notification cards found in 'output' directory. Skipping."
          exit 0
        fi

        echo "Sending notification to Teams..."
        teams-notifications --webhook $$TEAMS_WEBHOOK_URL --input_dir output --chunked true
    secretEnv:
      - "TEAMS_WEBHOOK_URL"
```

### Step 12: Close the Pull Request

Finally, we clean up after ourselves. This last step closes the sync pull request and automatically deletes the temporary `sync-dev-after-prod` branch, leaving no trace of the workflow in our Git history.

```yaml
# 12. Clean up by closing the PR.
  - name: "[PATH_TO_YOUR_DOCKER_IMAGE]/gh:latest"
    id: "close-sync-pr"
    waitFor: ['send-teams-notification']
    entrypoint: "bash"
    args:
      - '-c'
      - |
        set -e
        _PR_NUMBER=$(cat /workspace/pr_number.txt)

        echo "Closing PR #$${_PR_NUMBER} and deleting branch '${_SYNC_BRANCH}'"
        gh pr close "$${_PR_NUMBER}" --repo "${REPO_FULL_NAME}" --comment "Automated sync complete. Closing PR" --delete-branch
    secretEnv:
      - "GITHUB_TOKEN"
```      