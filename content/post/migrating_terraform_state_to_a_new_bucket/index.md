---
title: "🚀 Migrating Terraform State to a New GCP Bucket"
date: 2025-02-28
description: "Recently, I had to refactor our Terraform code and migrate the Terraform state file to a new GCP bucket. 
                Fortunately, this was a straightforward migration, and I didn’t have to recreate any resources—just a quick state transfer was required.
                In this blog, I'll walk through the  step-by-step process I followed to ensure a smooth and  safe migration  of our Terraform state."
categories: ["Cloud","Terraform"]
tags: ["GCP","Terraform","Migration"]
image: "terraform_gcpbucket.jpg"
---

## 📌 Steps to Migrate Terraform State

### ✅ 1. Backup the Current State

Before making any changes, it’s always good practice to **backup the current state**:

```bash
terraform state pull > terraform.tfstate.backup
```

This ensures we have a **snapshot of the existing state**, just in case anything goes wrong.

### 🛠️ 2. Create a New GCS Bucket

Next, I created a new **Google Cloud Storage (GCS) bucket** to store the Terraform state:

```bash
gsutil mb gs://NEW_BUCKET_NAME
```

### 📂 3. Copy the State File to the New Bucket

Once the bucket was ready, I copied the existing Terraform state file from the old bucket to the new one:

```bash
gsutil cp -r gs://SOURCE_BUCKET_NAME/* gs://DESTINATION_BUCKET_NAME
```

### 📝 4. Update the Backend Configuration

Now, I updated the Terraform **backend configuration** in `main.tf` to point to the new bucket:

```hcl
terraform {
  backend "gcs" {
    bucket = "NEW_BUCKET_NAME"
    prefix = "terraform/state"
  }
}
```

### 🔄 5. Migrate the State

To finalize the migration, I ran the following command:

```bash
terraform init -migrate-state
```

Terraform automatically detected the change in backend configuration and migrated the state to the new bucket.

## 🔍 Verifying the Migration

After migrating the state, I performed several **verification steps** to ensure everything was intact.

### 🧐 1. Check the New State File

Pulled the **new** state file to compare with the old one:

```bash
terraform state pull > new_state.tfstate
```

### 🔍 2. Compare Old and New State Files

Checked if there were any unexpected changes:

```bash
diff terraform.tfstate.backup new_state.tfstate
```

### 🔄 3. Run a Plan to Ensure No Drift

To confirm the migration didn't introduce unintended changes, I ran:

```bash
terraform plan
```

✅ Expected Outcome: The terraform plan should show no changes, confirming that everything migrated successfully.

## 🛡️ Safety Checks

To ensure the integrity of the migration, I followed these **best practices**:

- **Plan should show no changes** after migration ✅
- **State list should match before & after migration** ✅
- **Diff between state files should be minimal** (only backend config changes) ✅
  
## 🎉 Conclusion

Migrating Terraform state can seem risky, but by **following a structured approach** and **verifying each step**, you can ensure a smooth transition **without disrupting your infrastructure**.

If you ever need to migrate your Terraform state, remember:

💡 **Backup first**

💡 **Verify integrity after migration**

💡 **Run a `terraform plan` to detect any unintended changes**