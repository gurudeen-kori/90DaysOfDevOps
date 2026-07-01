# Day 64 — Terraform State Management and Remote Backends

## Overview

Terraform's state file is the single source of truth mapping `.tf` configuration to real-world infrastructure. Today's focus: moving from local state to a remote backend, locking, importing existing resources, state surgery, and handling drift.

---

## Task 1: Inspecting State

Commands used:

```bash
terraform show
terraform state list
terraform state show aws_instance.<name>
terraform state show aws_vpc.<name>
```

**Q: How many resources does Terraform track?**
> _Fill in: number of resources from `terraform state list` output._

**Q: What attributes does the state store for an EC2 instance?**
Far more than what's declared in the `.tf` file. State captures the *full* resource as returned by the AWS API, including:
- `id`, `arn`, `availability_zone`, `private_ip`, `public_ip`, `private_dns`, `public_dns`
- `instance_state`, `instance_type`, `ami`
- Network interface details, security group IDs, subnet ID, VPC ID
- `root_block_device` details (volume ID, size, type, delete_on_termination)
- Tags, key name, monitoring status, tenancy, and dozens of computed/default fields

Only a handful of these are ever set explicitly in the config — the rest are populated by Terraform from the provider's API response so it can detect drift and compute diffs accurately.

**Q: What is the `serial` number in `terraform.tfstate`?**
The `serial` is a monotonically increasing counter incremented every time the state is written. Terraform uses it to detect conflicting or out-of-order writes — if two processes try to push state with mismatched serials, Terraform knows something is wrong and refuses to blindly overwrite.

---

## Task 2: S3 Remote Backend + DynamoDB Locking

### Why remote state?

Local state (`terraform.tfstate` on disk) is dangerous:
- A single deleted or corrupted file loses the mapping between config and real infrastructure
- No collaboration — two people can't safely share a local file
- No locking — concurrent applies can corrupt state

### Infrastructure created

```bash
aws s3api create-bucket \
  --bucket terraweek-state-<yourname> \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

aws s3api put-bucket-versioning \
  --bucket terraweek-state-<yourname> \
  --versioning-configuration Status=Enabled

aws dynamodb create-table \
  --table-name terraweek-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

### Backend configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "terraweek-state-<yourname>"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}
```

Ran `terraform init`, confirmed **yes** to copying existing local state into the new backend.

### Local vs. Remote State — Diagram

```
LOCAL STATE (before)                    REMOTE STATE (after)
┌─────────────────┐                     ┌──────────────────────────┐
│   Your Machine    │                     │      Your Machine          │
│                    │                     │                             │
│  terraform.tfstate │                     │  (no local state file)     │
│  (single copy)     │                     │                             │
│                    │                     │  terraform init/plan/apply │
│  Risk:              │                     └────────────┬────────────┘
│  - Lost if deleted  │                                  │
│  - No locking       │                                  ▼
│  - Not shareable    │                     ┌──────────────────────────┐
└─────────────────┘                     │   S3 Bucket                 │
                                         │   dev/terraform.tfstate     │
                                         │   (versioned)                │
                                         └────────────┬────────────┘
                                                        │ lock/unlock
                                                        ▼
                                         ┌──────────────────────────┐
                                         │   DynamoDB Table            │
                                         │   terraweek-state-lock      │
                                         │   (LockID hash key)         │
                                         └──────────────────────────┘
```

### Verification

- S3 bucket contains `dev/terraform.tfstate` ✅
- Local `terraform.tfstate` is now empty/removed ✅
- `terraform plan` shows **no changes** — confirms state migrated correctly ✅

> _Insert screenshot: S3 bucket showing `dev/terraform.tfstate` object here._

---

## Task 3: State Locking

**Test:** Ran `terraform apply` in Terminal 1 (left waiting at the confirmation prompt), then ran `terraform plan` in Terminal 2 while the lock was held.

**Error message from Terminal 2:**
```
Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        <LOCK_ID>
  Path:      terraweek-state-<yourname>/dev/terraform.tfstate
  Operation: OperationTypeApply
  Who:       <user@host>
  Version:   <terraform version>
  Created:   <timestamp>
```

> _Insert screenshot: lock error from Task 3 here._

**Why locking is critical for team environments:**
Without locking, two engineers (or two CI pipeline runs) could write to the state file at the same time. The second write could overwrite the first, silently losing track of resources Terraform just created or modified — leading to duplicate resources, orphaned infrastructure, or Terraform trying to destroy things it no longer "sees." DynamoDB's conditional writes on the `LockID` key give Terraform a distributed mutex, ensuring only one `apply`/`plan`-with-refresh operation touches state at a time.

**Recovering from a stale lock:**
```bash
terraform force-unlock <LOCK_ID>
```
Only used when certain no other operation is genuinely running — otherwise it can cause the exact corruption locking is meant to prevent.

---

## Task 4: Importing an Existing Resource

Manually created bucket `terraweek-import-test-<yourname>` in the AWS console (outside Terraform).

Added a minimal resource block:

```hcl
resource "aws_s3_bucket" "imported" {
  bucket = "terraweek-import-test-<yourname>"
}
```

Imported it into state:

```bash
terraform import aws_s3_bucket.imported terraweek-import-test-<yourname>
```

Ran `terraform plan` — iterated on the config until it matched reality exactly and the plan showed **"No changes."**

Confirmed with `terraform state list` — the imported bucket now appears alongside the other tracked resources.

**Q: Difference between `terraform import` and creating a resource from scratch?**
- **Create from scratch:** Terraform is the origin — it calls the AWS API to create the resource, then records the result in state. Config and reality are in sync from the very first `apply`.
- **Import:** The resource already exists in AWS, created by some other means (console, CLI, another IaC tool). `terraform import` only adds an entry to the *state file* — it does **not** generate the `.tf` config. You must hand-write a resource block that matches the real resource's attributes; if it doesn't match, `plan` will show a diff (Terraform trying to "correct" the mismatch on the next apply). Import is a one-time bridge to bring unmanaged infrastructure under Terraform's control without recreating it.

---

## Task 5: State Surgery — `mv` and `rm`

**Rename in state:**
```bash
terraform state list
terraform state mv aws_s3_bucket.imported aws_s3_bucket.logs_bucket
```
Updated the `.tf` file's resource label to `logs_bucket` to match. `terraform plan` showed no changes — the rename was purely a state-level relabeling, no destroy/recreate.

**Remove from state (without destroying):**
```bash
terraform state rm aws_s3_bucket.logs_bucket
```
`terraform plan` afterward showed Terraform wanting to *create* a new bucket — because it no longer knows this bucket exists, even though it's still sitting in AWS untouched.

**Re-import to bring it back:**
```bash
terraform import aws_s3_bucket.logs_bucket terraweek-import-test-<yourname>
```

**Q: When to use `state mv` vs `state rm`?**
- **`state mv`** — when refactoring code: renaming a resource, moving it into a module, or splitting/merging Terraform configurations. It preserves the link between config and the real resource without triggering a destroy + recreate cycle, which would cause downtime for stateful resources (e.g., databases, EC2 instances with attached data).
- **`state rm`** — when you want Terraform to *stop managing* a resource but leave it running in AWS. Common cases: handing a resource off to another team/tool, decommissioning a workspace without deleting the underlying infra, or deliberately excluding a sensitive resource from Terraform's control. The resource is untouched in AWS — only the state's knowledge of it is deleted.

---

## Task 6: Simulating and Fixing State Drift

**Setup:** Ran a full `apply` so config, state, and AWS were all in sync.

**Manual drift introduced (outside Terraform, via AWS console):**
- Changed the EC2 instance's `Name` tag to `"ManuallyChanged"`
- (Optionally) modified the instance type / added a new tag while stopped

**Detecting the drift:**
```bash
terraform plan
```
Terraform refreshes real infrastructure state and compares it against the `.tf` config, surfacing a diff — e.g.:
```
~ resource "aws_instance" "web" {
    ~ tags = {
        ~ "Name" = "ManuallyChanged" -> "web-server"
      }
  }
```

**Reconciliation — chose Option A (enforce config as truth):**
```bash
terraform apply
```
This pushed the config's original tag value back onto the real EC2 instance, overwriting the manual change.

Ran `terraform plan` again → **"No changes."** Drift resolved.

> _Insert diagram/notes on the drift example: what was changed manually, and the before/after tag value._

**Q: How do teams prevent state drift in production?**
- **Restrict console/CLI write access** — give engineers read-only IAM permissions in production; only a CI/CD service role can run `apply`.
- **Route all changes through CI/CD** — infrastructure changes only happen via a merged PR that triggers a pipeline `plan`/`apply`; no one hand-edits resources.
- **Scheduled drift detection** — run `terraform plan` (or `terraform apply -refresh-only`) on a cron/CI schedule to catch drift early, even if the access rule is broken.
- **Alerting on non-empty plans** — pipe drift-detection output into Slack/monitoring so any diff is visible immediately.
- **Audit logging (CloudTrail)** — traces exactly who made a manual change outside Terraform, for accountability.

---

## Command Reference — When to Use What

| Command | Use case |
|---|---|
| `terraform state mv` | Renaming a resource or refactoring into modules without destroy/recreate |
| `terraform state rm` | Stop managing a resource in Terraform while leaving it running in AWS |
| `terraform import` | Bring a pre-existing, manually created resource under Terraform management |
| `terraform force-unlock` | Clear a stale lock when certain no other operation is running |
| `terraform refresh` / `apply -refresh-only` | Sync state with real infrastructure without changing anything |

---

## Summary

Migrated state from local to an S3 + DynamoDB remote backend with locking, imported a pre-existing S3 bucket into Terraform management, performed state surgery (`mv`/`rm`/re-import), and simulated + reconciled state drift on an EC2 instance. State management is the backbone of reliable, collaborative infrastructure-as-code — remote backends prevent data loss, locking prevents corruption, and disciplined drift detection keeps reality and code in sync.

#90DaysOfDevOps #TerraWeek #DevOpsKaJosh #TrainWithShubham
